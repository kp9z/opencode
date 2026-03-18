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
1. `packages/opencode/src/session/index.ts` — `appendMessage()`
2. `packages/opencode/src/session/llm.ts` — `LLM.stream()`
3. `packages/opencode/src/session/processor.ts` — the main processing loop
4. `packages/opencode/src/session/message-v2.ts` — all message/part types
5. `packages/opencode/src/tool/tool.ts` — Tool.Info interface
6. `packages/opencode/src/tool/registry.ts` — how tools are registered

**Key concept:** The processor loop is the heart of the system. It handles streaming events and dispatches tool calls, loops back to the LLM after tool execution, detects doom loops (same tool + args 3x), and compacts context when tokens overflow.

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

**Built-in tool categories:**
| Category | Tools |
|----------|-------|
| File ops | `read`, `write`, `edit`, `multiedit`, `glob`, `ls` |
| Search | `grep`, `codesearch`, `websearch`, `webfetch` |
| Execution | `bash`, `apply_patch` |
| Management | `task`, `todo`, `plan`, `question`, `skill` |
| Advanced | `lsp`, `batch` |

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
