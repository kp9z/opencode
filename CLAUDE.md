# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## About

OpenCode is an open-source AI coding agent. It has a client/server architecture: `packages/opencode` is the core server/engine, and it has multiple frontends (TUI, web app, desktop app).

The default branch is `dev`. Local `main` may not exist; use `dev` or `origin/dev` for diffs.

## Commands

```bash
bun install           # Install dependencies
bun dev               # Start TUI (runs against packages/opencode dir by default)
bun dev <directory>   # Start TUI against a specific directory
bun dev serve         # Start headless API server (port 4096)
bun dev web           # Start server + open web interface
bun dev:web           # Web UI dev server only (http://localhost:5173, requires server running separately)
bun dev:console       # Console/dashboard app
bun dev:storybook     # Storybook component library
bun typecheck         # Run TypeScript checks across all packages (via Turbo)
```

Run tests from within specific packages (e.g., `packages/opencode`), never from the repo root — the root `test` script is intentionally blocked.

Run `bun typecheck` from package directories (e.g., `packages/opencode`), not `tsc` directly.

After modifying the API or SDK (`packages/opencode/src/server/server.ts`), regenerate the SDK:

```bash
./script/generate.ts
./packages/sdk/js/script/build.ts  # regenerate JS SDK specifically
```

Build a standalone executable:

```bash
./packages/opencode/script/build.ts --single
# Output: ./packages/opencode/dist/opencode-<platform>/bin/opencode
```

## Architecture

Bun monorepo managed with Turbo. Key packages:

- **`packages/opencode`** — Core engine, server, agent logic, tools, CLI. The heart of the codebase.
  - `src/cli/` — CLI entry points (yargs commands)
  - `src/cli/cmd/tui/` — Terminal UI (SolidJS + [opentui](https://github.com/sst/opentui))
  - `src/agent/` — Agent definitions (build/plan agents)
  - `src/server/` — Hono HTTP API server
  - `src/session/` — Session management
  - `src/tool/` — Agent tools (file edit, bash, search, etc.)
  - `src/lsp/` — Language Server Protocol client
  - `src/mcp/` — Model Context Protocol support
  - `src/provider/` — LLM provider abstraction (via Vercel AI SDK)
  - `src/config/` — Configuration loading
  - `src/effect/` — Effect library service infrastructure
- **`packages/app`** — Shared web UI components (SolidJS), used by web and desktop
- **`packages/desktop`** — Native desktop app (Tauri v2, wraps `packages/app`)
- **`packages/console/*`** — Multi-tenant SaaS console (SST/AWS infrastructure)
- **`packages/sdk/js`** — Published JavaScript SDK (`@opencode-ai/sdk`)
- **`packages/plugin`** — Plugin system (`@opencode-ai/plugin`)

**Key technologies:**
- Runtime: Bun
- UI: SolidJS (both TUI and web)
- HTTP server: Hono
- DB: Drizzle ORM + SQLite
- LLM integration: Vercel AI SDK (provider-agnostic)
- Service layer: Effect library
- Code parsing: Tree-sitter
- Infrastructure: SST (AWS)

## Style Guide

These rules are **mandatory** for all code written in this repo (from `AGENTS.md`):

**Naming — single word by default:**
```ts
// Good
const cfg = ...
function journal(dir: string) {}

// Bad
const openConfig = ...
function prepareJournal(dir: string) {}
```
Prefer: `pid`, `cfg`, `err`, `opts`, `dir`, `root`, `child`, `state`. Inline variables used only once.

**No destructuring — use dot notation:**
```ts
obj.a   // good
const { a } = obj  // bad
```

**No `else` — use early returns:**
```ts
if (condition) return 1
return 2
```

**No `try`/`catch` — use `.catch()`**

**`const` over `let`** — use ternaries or early returns instead of reassignment.

**Drizzle schema fields — snake_case** (avoids needing string column name overrides):
```ts
project_id: text().notNull()  // good
projectId: text("project_id")  // bad
```

**Avoid `any` types.** Use Bun APIs (`Bun.file()`, etc.) when available. Rely on type inference; avoid explicit annotations unless needed for exports.

## Debugging

To debug with breakpoints, run with `--inspect`:
```bash
bun run --inspect=ws://localhost:6499/ --cwd packages/opencode ./src/index.ts serve --port 4096
# Then attach TUI:
opencode attach http://localhost:4096
```

Use `bun dev spawn` if you need breakpoints to work in server code while running the TUI.

VSCode launch/settings examples: `.vscode/launch.example.json`, `.vscode/settings.example.json`.
