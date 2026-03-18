# OpenCode Onboarding & Learning Plan

## Context

OpenCode is an open-source AI coding agent with a client/server architecture. The core engine (`packages/opencode`) exposes a Hono HTTP API that multiple frontends (TUI, web app, desktop) consume. The codebase is built on Bun, SolidJS, Effect library, Vercel AI SDK, and Drizzle ORM + SQLite.

This plan is structured as a progressive deep-dive: start with the big picture, then trace a single request end-to-end, then branch into subsystems.

---

## Phase 1: Big Picture & Entry Points

**Goal:** Understand what the repo is and how to run it.

1. Read `CLAUDE.md` and `packages/opencode/src/index.ts` — CLI entry point, available commands
2. Run `bun dev` to launch the TUI and get a feel for the product
3. Read `packages/opencode/src/cli/` — understand the yargs command tree
4. Read `packages/opencode/src/server/server.ts` — see all HTTP routes at a glance

**Key files:**
- `packages/opencode/src/index.ts`
- `packages/opencode/src/cli/cmd/tui/thread.ts` (how TUI boots)
- `packages/opencode/src/server/server.ts`

---

## Phase 2: End-to-End Request Flow

**Goal:** Trace a single user prompt from keypress to LLM response.

Follow this flow in order:

```
User types prompt in TUI
  → thread.ts (Worker RPC)
  → POST /session/:id/message (server.ts)
  → Session.appendMessage() (session/index.ts)
  → LLM.stream() (session/llm.ts)
  → Vercel AI SDK streamText()
  → SessionProcessor loop (session/processor.ts)
    ↳ text-delta → update TextPart
    ↳ tool-call  → create ToolPart → Tool.execute()
    ↳ tool-result → store output, continue stream
  → SSE /event endpoint → TUI re-renders
```

**Key files (read in this order):**

### `packages/opencode/src/cli/cmd/tui/thread.ts`
Entry point for the TUI command. Spawns a single long-lived Bun Worker at startup (not per message).
Creates an RPC client — a thin wrapper over `postMessage`/`onmessage` that makes worker calls look like async function calls (`client.call("fetch", ...)`). All HTTP requests from the TUI go through this RPC client instead of a real TCP socket.

### `packages/opencode/src/cli/cmd/tui/worker.ts`
Runs inside the Bun Worker thread. Boots the Hono HTTP server, SQLite, session logic, and all agent infrastructure. Subscribes to the internal event bus and forwards events back to the main thread via `Rpc.emit("global.event", event)` so the TUI can re-render.

### `packages/opencode/src/session/prompt.ts`
The orchestration layer. Contains two key functions:
- **`prompt()`** — creates the user message in DB, then calls `loop()`
- **`loop()`** — the outer `while(true)` loop (line 297). Each iteration: finds the last user message, checks if done, handles pending subtasks/compaction, builds the system prompt, resolves tools, creates a `SessionProcessor`, and calls `processor.process()`. Breaks when processor returns `"stop"` or finish reason is not `"tool-calls"`.
- **`resolveTools()`** — assembles the full tool list from three sources: `ToolRegistry.tools()` (built-ins), `MCP.tools()` (MCP servers), and LSP tools. Builds a `Tool.Context` closure per tool that wires up permission checks and part updates.
- **`insertReminders()`** — injects synthetic text parts into the last user message on each loop iteration: `plan.txt` when in plan mode, `build-switch.txt` when switching plan → build.

### `packages/opencode/src/session/index.ts`
Pure data layer for sessions. Key functions:
- **`updateMessage()`** — upserts a message row and publishes `MessageV2.Event.Updated` to the bus
- **`updatePart()`** — upserts a part row and publishes `MessageV2.Event.PartUpdated` (full DB write)
- **`updatePartDelta()`** — publishes `MessageV2.Event.PartDelta` only (no DB write — used for streaming text deltas to avoid write amplification)
- **`getUsage()`** — normalizes token counts and calculates cost across providers (Anthropic counts cached tokens differently from OpenAI/OpenRouter)
- **`messages()`** — loads all messages+parts for a session in order

### `packages/opencode/src/session/llm.ts`
Thin wrapper around Vercel AI SDK's `streamText()`. Key responsibilities:
- **`stream()`** — resolves the language model adapter, assembles the system prompt array (agent prompt or provider-specific prompt + environment + custom), merges provider/agent/variant options, calls `streamText()` with the tool list and message history
- **`resolveTools()`** — final permission filter: removes any tool disabled by the agent's ruleset before the LLM call. The LLM never sees disabled tools.
- System prompt is assembled as a 2-part array (header + rest) to maximize Anthropic prompt caching

**System prompt assembly (final order sent to LLM):**
```
1. agent.prompt  OR  SystemPrompt.provider(model)   ← who you are + how to behave
2. SystemPrompt.environment(model)                  ← working dir, platform, date
3. SystemPrompt.skills(agent)                       ← available slash commands
4. InstructionPrompt.system()                       ← CLAUDE.md / custom instructions
```

Parts 2-4 are assembled in `prompt.ts:656` and passed as `system[]` into `LLM.stream()`.
Part 1 is prepended inside `llm.ts:73`.

**Provider-specific prompts (`session/system.ts:provider()`):**
Each model family gets a different base prompt because they respond to instructions differently:

| Model pattern | Prompt file | Style |
|---|---|---|
| `claude` | `session/prompt/anthropic.txt` | Conversational, nuanced, TodoWrite guidance |
| `gpt-` / `o1` / `o3` | `session/prompt/beast.txt` | Aggressive repetition ("KEEP GOING", "DO NOT STOP") — fights GPT's tendency to stop early |
| `gpt-5` (Codex OAuth) | `session/prompt/codex_header.txt` | Sent via `options.instructions` field, not system prompt |
| `gemini-` | `session/prompt/gemini.txt` | Gemini-tuned |
| `trinity` | `session/prompt/trinity.txt` | Trinity-tuned |
| everything else | `session/prompt/qwen.txt` | Anthropic-style but without TodoWrite |

`build` and `plan` agents have **no `agent.prompt`** — they fall through to `SystemPrompt.provider(model)`.
Subagents (`explore`, `compaction`, `title`, `summary`) have their own `agent.prompt` and skip `provider()` entirely.

**Tool definitions are NOT in the system prompt.** Each tool's `description` and `inputSchema` are passed directly to `streamText({ tools })` via the Vercel AI SDK. The LLM learns what tools exist from the tool definitions per-call, not from the system prompt. The only exception is skills — those are listed in the system prompt via `SystemPrompt.skills()`.

### `packages/opencode/src/session/processor.ts`
The inner loop — handles one LLM round-trip. Key responsibilities:
- **`create()`** — factory that holds per-response state: `toolcalls` map (callID → ToolPart), `blocked` flag, `needsCompaction` flag
- **`process()`** — inner `while(true)` loop (line 50) that iterates `stream.fullStream`. Dispatches on event type:
  - `text-start/delta/end` — creates/streams/finalizes a `TextPart`
  - `reasoning-start/delta/end` — same pattern for reasoning/thinking tokens
  - `tool-input-start` → creates `ToolPart` with `status: "pending"`
  - `tool-call` → updates to `status: "running"`, runs doom loop check (same tool + same args 3× in last 3 parts → asks permission)
  - `tool-result` → updates to `status: "completed"` with output
  - `tool-error` → updates to `status: "error"`, sets `blocked=true` if user denied
  - `start-step` → takes filesystem snapshot via `Snapshot.track()`
  - `finish-step` → calculates cost/tokens, checks context overflow → sets `needsCompaction`
- Returns `"continue"` | `"stop"` | `"compact"` to the outer loop

### `packages/opencode/src/session/message-v2.ts`
All message and part type definitions. A session message has a `role` (`user` | `assistant`) and an array of typed parts:
- `TextPart` — LLM text output
- `ReasoningPart` — chain-of-thought / thinking tokens
- `ToolPart` — a tool call with state machine: `pending → running → completed | error`
- `StepStartPart` / `StepFinishPart` — bookend each LLM round-trip, carry token/cost/snapshot data
- `PatchPart` — file diff produced after tool execution
- `FilePart` — user-attached file
- `SubtaskPart` / `CompactionPart` — pending work items processed by the outer loop

### `packages/opencode/src/tool/tool.ts`
Defines the `Tool.Info` interface every tool must implement:
- `id` — tool name (matches permission key)
- `description` — shown to LLM
- `parameters` — Zod schema for input validation
- `execute(args, ctx)` — the tool implementation
- `Tool.Context` — injected into every tool execution: `sessionID`, `messageID`, `abort` signal, `ask()` (permission check), `metadata()` (update the running ToolPart UI), `messages` (full conversation history)

### `packages/opencode/src/tool/registry.ts`
Registers all built-in tools and filters them per agent/model. `ToolRegistry.tools(model, agent)` returns only tools compatible with the current model and not disabled by the agent's permission ruleset.

**Key concept:** The processor loop is the heart of the system. It handles streaming events and dispatches tool calls, loops back to the LLM after tool execution, detects doom loops (same tool + args 3x), and compacts context when tokens overflow.

**Agent setup differences (build vs plan):**
- Both use the same base system prompt (`anthropic.txt` for Claude, `beast.txt` for GPT, etc.) from `session/system.ts`
- **build**: can use all tools; `plan_enter` allowed; `plan_exit` denied
- **plan**: `edit`/`write` denied except `.opencode/plans/*.md`; `plan_exit` allowed; `plan_enter` denied
- On each loop iteration, `insertReminders()` appends a synthetic reminder to the last user message — `plan.txt` ("READ-ONLY, ZERO exceptions") in plan mode, `build-switch.txt` ("you may now make file changes") when switching back
- Permission ruleset is **hard enforcement** (tools removed from LLM call); reminder text is **soft enforcement** (steers behavior in natural language)

**Tool attachment chain:**
```
prompt.ts:resolveTools()
  → ToolRegistry.tools()     built-in tools
  → MCP.tools()              MCP server tools
  → LSP tools                language server tools
  → passed to processor.process({ tools })
    → passed to LLM.stream({ tools })
      → llm.ts:resolveTools()   strips permission-denied tools
        → streamText({ tools })  LLM sees final filtered list
```

**How subagents are called (`tool/task.ts`):**

The main agent calls the `task` tool like any other tool. There is no special dispatch — it's a regular tool call that blocks until the child finishes.

```
LLM emits tool-call: task({ subagent_type: "explore", prompt: "...", description: "..." })
  → task.ts:execute()
    → Session.create({ parentID: ctx.sessionID })   creates a child session in DB
    → SessionPrompt.prompt({                         runs the full loop() for child
        sessionID: child.id,
        agent: "explore",
        model: agent.model ?? parent model,
        parts: resolvePromptParts(params.prompt),
      })
      → child session runs its own while(true) loop to completion
      → child LLM only sees tools allowed by explore agent's permission ruleset
    → takes last TextPart from child result
    → returns string:
        "task_id: <session_id>\n<task_result>\n{text}\n</task_result>"

  → processor receives tool-result with that string
  → parent LLM sees it on the next round-trip and continues
```

**Key constraints on child sessions:**
- `todowrite: deny`, `todoread: deny` — subagents don't manage the parent's todo list
- `task: deny` by default — subagents can't spawn further subagents (unless the agent config explicitly has task permission)
- `task_id` parameter — optional, resumes an existing child session instead of creating a new one
- Child uses parent's model unless the agent config specifies its own `model`

---

## Phase 3: Agent System

**Goal:** Understand what agents are and how they differ.

1. Read `packages/opencode/src/agent/agent.ts` — Agent.Info structure, all built-in agents
2. Note: agents differ by **permission ruleset**, **model**, **temperature**, and **system prompt**
3. Agents: `build` (default), `plan` (read-only), `explore` (search/read only), `general`, `compaction`, `title`, `summary`
4. The `build` agent has broad permissions; `plan` agent cannot write files

**Key concept:** Agents are just configuration — same processor, different restrictions and prompts.

---

## Phase 4: Tool System

**Goal:** Understand the 27 built-in tools and how to add custom ones.

1. Explore `packages/opencode/src/tool/` directory
2. Read `packages/opencode/src/tool/tool.ts` — Tool.Info interface, Tool.Context
3. Read a simple tool (e.g., `read.ts` or `glob.ts`) and a complex one (`bash.ts`, `edit.ts`)
4. Understand `Truncate.output()` — tools auto-truncate large output
5. Understand permission checks inside tools: `ctx.ask()`

**Built-in tools:**

| Tool | What it does |
|------|-------------|
| `bash` | Executes bash commands in a persistent shell session |
| `read` | Reads file content with optional line offset/limit |
| `write` | Writes/overwrites a file on disk |
| `edit` | Exact string replacement in a file (oldString → newString) |
| `apply_patch` | GPT-model alternative to `edit` — uses Add/Update/Delete patch format |
| `glob` | Finds files by pattern (e.g. `**/*.ts`), sorted by modification time |
| `grep` | Searches file contents by regex, optionally filtered by file type |
| `webfetch` | Fetches a URL and returns content as markdown/text/html |
| `websearch` | Real-time web search via Exa AI (requires opencode provider or flag) |
| `codesearch` | Searches code-specific context via Exa Code API — libraries, SDKs, API refs |
| `task` | Spawns a subagent session to handle a subtask autonomously |
| `todowrite` | Creates/updates a structured task list to track progress |
| `plan_exit` | Exits plan mode and signals switch to build agent (experimental flag only) |
| `question` | Asks the user a question mid-task to clarify requirements or get a decision |
| `skill` | Loads a skill (SKILL.md + bundled files) into conversation context |
| `lsp` | Language server integration — hover, definitions, references, diagnostics (experimental flag) |
| `batch` | Runs 1-25 independent tool calls concurrently to reduce latency (experimental config) |
| `invalid` | Placeholder that returns an error — catches malformed tool calls from the LLM |

---

## Phase 4b: Plugin & Hook System

**Key files:**
- `packages/opencode/src/plugin/index.ts` — plugin loader and `Plugin.trigger()`
- `packages/plugin/src/index.ts` — `Hooks` interface definition

**What a plugin is:**
A plugin is a function that returns a `Hooks` object. Registered in config via `plugin: ["file:///path/to/plugin.ts"]` or an npm package name. Loaded once at startup per instance.

```ts
export default async function(input: PluginInput): Promise<Hooks> {
  return {
    "tool.execute.before": async (input, output) => {
      output.args.myExtra = "injected"
    },
  }
}
```

**How `Plugin.trigger()` works:**
Called at every hook point in the codebase. Iterates all loaded plugin hooks for that name, passes `output` object to each — plugins mutate it in place. Returns the (possibly modified) output.

```ts
const params = await Plugin.trigger("chat.params", { sessionID, model }, { temperature: 0.7 })
// params.temperature may have been modified by a plugin
```

**Available hooks:**

| Hook | When it fires | What you can modify |
|---|---|---|
| `event` | every bus event | observe only |
| `config` | at startup | — |
| `tool` | registration | add custom tools |
| `auth` | auth flow | credentials |
| `chat.message` | new user message received | message parts |
| `chat.params` | before every LLM call | temperature, topP, options |
| `chat.headers` | before every LLM call | HTTP headers |
| `permission.ask` | permission check | allow/deny/ask decision |
| `tool.execute.before` | before any tool runs | tool args |
| `tool.execute.after` | after any tool runs | tool output string |
| `shell.env` | before bash executes | environment variables |
| `experimental.chat.system.transform` | system prompt assembly | system prompt array |
| `experimental.chat.messages.transform` | before LLM call | full message history |
| `experimental.session.compacting` | before compaction | compaction prompt |
| `experimental.text.complete` | after LLM text finishes | text content |

**Built-in internal plugins** (always loaded, not via config):
- `CodexAuthPlugin` — handles OpenAI Codex OAuth
- `CopilotAuthPlugin` — handles GitHub Copilot auth
- `GitlabAuthPlugin` — handles GitLab auth

**App lifecycle — when plugins load (`project/bootstrap.ts`):**

Plugins are loaded once per instance via `InstanceBootstrap()`, which is called on the first request to `/session` or `/project`. Not at server startup — lazy, per working directory.

```
bun dev
  → worker.ts spawns → Hono server ready

first session/project request
  → Instance.init() → InstanceBootstrap()
    → Plugin.init()
      → load internal plugins (direct imports — Codex, Copilot, Gitlab)
      → read config.plugin[] list
      → BunProc.install(pkg, version)   install from npm if needed
      → import(pluginPath)              dynamic import at runtime
      → plugin(input)                   call plugin function → get Hooks object
      → push Hooks into hooks[] array
      → Bus.subscribeAll → forward all events to hook["event"]
    → LSP.init(), Format.init(), FileWatcher.init(), VcsService.init()

every subsequent Plugin.trigger() call
  → iterate in-memory hooks[] array   fast, no re-loading
```

`Instance.state()` caches the result — plugin loading only happens once per working directory for the lifetime of the process.

---

## Phase 5: LLM Provider Abstraction

**Goal:** Understand how OpenCode supports 20+ LLM providers.

1. Read `packages/opencode/src/provider/provider.ts` — Provider.Model, auth management, model registry
2. Read `packages/opencode/src/provider/transform.ts` — how models are wrapped for Vercel AI SDK
3. Understand the Vercel AI SDK `streamText()` interface — OpenCode is provider-agnostic because Vercel AI SDK normalizes all providers
4. Read `packages/opencode/src/session/llm.ts` — how provider + model are selected per message

---

## Phase 6: Effect Library & Service Layer

**Goal:** Understand why Effect is used and how services are structured.

1. Read `packages/opencode/src/effect/runtime.ts` — ManagedRuntime, layer composition
2. Read `packages/opencode/src/effect/instances.ts` — all InstanceServices (FileService, VcsService, PermissionEffect, etc.)
3. Read any `*.effect.ts` file to see the pattern: `Effect<A, E, InstanceServices>`
4. Understand `runPromiseInstance()` — the main execution entry point

**Key concept:** Effect is used for typed dependency injection and typed errors. Instead of passing services as arguments, they're provided via layers. `InstanceServices` is the "context" available to all tool/session code for a given working directory.

---

## Phase 7: Data Layer

**Goal:** Understand the database schema and how data flows.

1. Read `packages/opencode/src/session/session.sql.ts` — SessionTable, MessageTable, PartTable, TodoTable, PermissionTable
2. Read `packages/opencode/src/storage/db.ts` — SQLite init (WAL mode, 64MB cache)
3. Read `packages/opencode/src/session/schema.ts` — branded ID types (SessionID, MessageID, PartID)
4. Understand the ID ordering: SessionID is descending (newest first), MessageID/PartID are ascending

---

## Phase 8: Frontend (TUI)

**Goal:** Understand the terminal UI architecture.

1. Read `packages/opencode/src/cli/cmd/tui/thread.ts` — how Worker is spawned
2. Read `packages/opencode/src/cli/cmd/tui/worker.ts` — RPC server in Worker thread
3. Read `packages/opencode/src/cli/cmd/tui/app.tsx` — SolidJS component tree, context providers
4. Read `packages/opencode/src/cli/cmd/tui/context/sdk.tsx` — how SSE events are consumed and coalesced
5. Understand event coalescing: high-frequency SSE events are batched (16ms) to avoid flooding the UI

---

## Phase 9: Subsystems (Pick Your Interest)

After the core flow, explore subsystems in any order:

### Config System
- `packages/opencode/src/config/config.ts` — 8-level precedence chain (global → project → managed)
- How enterprise "managed config" overrides everything

### Permission System
- `packages/opencode/src/permission/` — Ruleset, glob-style rules
- How tools call `ctx.ask()` to request user approval at runtime

### MCP (Model Context Protocol)
- `packages/opencode/src/mcp/index.ts` — OAuth, stdio/HTTP transports, tool conversion
- How external MCP servers add tools to the agent

### LSP (Language Server Protocol)
- `packages/opencode/src/lsp/index.ts` — lazy server spawning per file extension
- How the `lsp` tool provides hover/definition/references/diagnostics

### Session Sharing & Control Plane
- `packages/opencode/src/share/` — public session sharing
- `packages/console/` — multi-tenant SaaS infrastructure (SST/AWS)

### Web UI
- `packages/app/` — SolidJS web app, same API as TUI
- `packages/desktop/` — Tauri v2 wrapper around web app

---

## Verification / Hands-On Exercises

1. **Trace a tool call:** Add a `console.log` in `packages/opencode/src/session/processor.ts` around the `tool-call` event handler. Run `bun dev`, send a message that triggers a tool, observe the log.

2. **Read a session from DB:** After a session, inspect `~/.local/share/opencode/opencode.db` with a SQLite client to see the raw SessionTable, MessageTable, PartTable rows.

3. **Add a custom tool:** Create a new tool file in `packages/opencode/src/tool/`, register it in `registry.ts`, and test it in the TUI.

4. **Watch the SSE stream:** `curl -N http://localhost:4096/event` while sending a message to see raw SSE events.

5. **Run typechecker:** `cd packages/opencode && bun typecheck` to verify you haven't broken any types.

---

## Learning Sequence Summary

```
Phase 1: Entry points & product feel       (~1 hour)
Phase 2: Request flow end-to-end           (~2 hours)  ← most important
Phase 3: Agent system                      (~30 min)
Phase 4: Tool system                       (~1 hour)
Phase 5: LLM provider abstraction         (~1 hour)
Phase 6: Effect library & services        (~1 hour)
Phase 7: Data layer                        (~30 min)
Phase 8: TUI frontend                      (~1 hour)
Phase 9: Pick subsystems of interest       (open-ended)
```
