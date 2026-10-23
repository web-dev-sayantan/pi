# Architecture

## The repo is two stacks plus shared vocabulary

Most confusion in this codebase comes from this: there is a **shipping stack** and an
**experimental stack**, and they overlap in the `chord` package.

```
                       ┌──────────────────────────────────────────┐
   SHIPPING            │  coding-agent  (the `pi` CLI)             │
   (what users run)    │    └─ src/modes/{interactive,print,json,rpc}│
                       └────────────────────┬─────────────────────┘
                                            │
              ┌───────────────┬─────────────┼──────────────┐
              ▼               ▼             ▼              ▼
          agent-core          tui           ai          chord   (context only)
       (agent.ts + harness/)             (41 providers)
              │
              ├──────────────────────────────► chord       (pico3 services)
              └──────────────────────────────► telemetry

                       ┌──────────────────────────────────────────┐
   EXPERIMENTAL         │  coding-agent/src/experimental/{micro,mini,services} │
   (PI_EXPERIMENTAL=1)  └───────┬──────────────────────────┬────────┘
                               ▼                          ▼
                          client ◄── protocol ──► server
                               │                          │
                               └─────► durable (unwired) ◄┘
```

Dependency direction is strict and enforced. `npm run build` hardcodes the topological order
in `package.json:16`:

```
chord → tui → telemetry → ai → durable → agent → sqlite-node → protocol → client → server → coding-agent
```

Workspace dependencies (nothing else crosses package boundaries):

| Package | Depends on |
|---|---|
| `chord` | *(nothing — only `esbuild`)* |
| `telemetry` | *(nothing)* |
| `tui` | *(nothing — only `get-east-asian-width`, `marked`)* |
| `ai` | `telemetry` |
| `durable` | `chord`, `ai` |
| `agent` | `chord`, `ai`, `telemetry` |
| `session-backends/sqlite-node` | `ai`, `agent` — **unwired** |
| `protocol` | `chord` |
| `client` | `chord`, `protocol` |
| `server` | `chord`, `agent`, `protocol` |
| `coding-agent` | `chord`, `agent`, `ai`, `tui` (+ dev: `client`, `protocol`, `server`) |
| `evals` | *(private, dev-only)* |

Chord is the odd one: `README.md:1-8` says it is deliberately **not a Pi package** and depends on
no other Pi workspace package, so it can be used by unrelated applications. `PLANNING.md:26-37`
goes further — it bans Pi vocabulary (`Session`, `Agent`, `model`, `tool`, `hook`, `server`,
`client`) from becoming Chord concepts, and states the API is **not a stable contract yet**.

## Where a feature belongs

`CONTRIBUTING.md:7-11` is the rule: *pi's core is minimal; if your feature does not belong in the
core, it should be an extension.*

Concretely:

| You want to… | Put it in | Not in |
|---|---|---|
| Add a provider / model metadata | `packages/ai/src/providers/<id>.ts` + `scripts/generate-models.ts` | `coding-agent` |
| Add a tool, command, event handler, CLI flag, keybinding | an extension: `packages/coding-agent/examples/extensions/` | `core/tools/`, `core/extensions/types.ts` |
| Add a terminal widget or key handling | `packages/tui/src/components/` | `coding-agent` |
| Change the agent loop | `packages/agent/src/agent-loop.ts` — but see the three-runtimes warning | |
| Add a durable/rpc/storage concept | `packages/agent/src/harness/` (+ update `packages/agent/docs/harness.md`) | `agent.ts` |

`packages/coding-agent` has exactly **one** built-in extension
(`src/extensions/llama/`, hidden, registered at `src/extensions/index.ts:4`) and ~64 example
extensions. That ratio is the intended shape of the project.

## Request lifecycle (shipping path)

Follow one user prompt through the code:

1. `src/modes/interactive/interactive-mode.ts` (or `modes/print-mode.ts`, `modes/rpc/rpc-mode.ts`)
   calls into `AgentSession`.
2. `packages/coding-agent/src/core/sdk.ts:380-415` constructs a `pi-agent-core` `Agent`. The
   `streamFn` it installs (`sdk.ts:389-400`) calls `ModelRuntime.streamSimple`.
3. `Agent.prompt()` / `Agent.continue()` (`packages/agent/src/agent.ts:374-387`) guard against
   concurrent runs, then `runWithLifecycle` (`agent.ts:507-530`) creates the `AbortController`
   and calls `runAgentLoop`.
4. `runAgentLoop` (`packages/agent/src/agent-loop.ts:101-125`) runs the turn loop
   (`runLoop`, `agent-loop.ts:162-320`): emit pending messages → `prepareRequest` →
   `streamAssistantResponse` → `executeToolCalls` → `finishTurn` → poll steering/follow-up queues.
5. Tool execution: `executeToolCalls` (`agent-loop.ts:505-520`) picks sequential or parallel.
   Sequential if `toolExecution === "sequential"` **or** any addressed tool declares
   `executionMode: "sequential"` — one sequential tool serializes the whole batch.
   `beforeToolCall` / `afterToolCall` hooks wrap each call (`agent-loop.ts:703-861`).
6. The stream call goes through pi-ai: `Models.stream()` →
   `Model.compat` flags → `api/<provider>.lazy.ts` → a concrete `src/api/<provider>.ts`
   → provider SDK, emitting `AssistantMessageEvent`s.
7. `processEvents` (`agent.ts:565-612`) reduces events into `AgentState` and awaits listeners
   **in registration order**. `AgentSession` subscribes to those and persists entries, drives the
   TUI, and fires extension events.

## Three runtimes in `packages/agent`

Confirm which one you mean before changing anything.

| Runtime | Entry | State model | Who uses it |
|---|---|---|---|
| **Core agent** | `src/agent.ts`, `src/agent-loop.ts` | in-memory, process-local, `AgentState` | `coding-agent` via `core/sdk.ts` |
| **Harness** | `src/harness/` | durable, crash-recoverable; entry tree + typed bound values + lanes; JSONL (`session/jsonl/`, format 4) or SQLite | not wired into the CLI |
| **Pico3** | `src/harness/pico3/` (exported only as `./experimental/pico3`) | durable, compose-agent-behaviour-from-task-kinds | `coding-agent/src/experimental/micro/` |

The harness shares **only types** with the core agent. `src/harness/` contains zero imports of
`agent.ts` or `agent-loop.ts`; it drives provider calls through its own
`src/harness/execution/assistant.ts`.

The normative spec is `packages/agent/docs/harness.md`. Its §0.9 is an implementation-status
ledger with named debt items (J1, C1, R11, R12, T1, S3, H1) — read it before assuming a harness
feature is complete. Note also `packages/agent/docs/mobile-handoff/README.md:22-28`:
**"If a doc and the code disagree, the code wins."**

`src/harness/search/index.ts` is four interfaces and zero implementation, and it is re-exported
from the package root. Do not build on it (`docs/harness.md:149`).

## Reading order for a first change

- **In the CLI**: `src/main.ts` (bootstrap order, ~985 lines) → `core/sdk.ts` (agent wiring) →
  `core/agent-session.ts` (the shared session class) → `modes/interactive/interactive-mode.ts`
  (the TUI mode).
- **In the loop**: `packages/agent/src/types.ts` (500 lines, all contracts) →
  `src/agent-loop.ts` (`runLoop` then `executeToolCalls`) → `src/agent.ts` (`processEvents`).
- **In providers**: `packages/ai/src/types.ts` (message + stream contracts) →
  `src/models.ts` (`Models` / `createProvider`) → one `src/api/<x>.ts` and one
  `src/providers/<x>.ts`.

## Related

- [development.md](development.md) — build, check, test, run from source
- [conventions.md](conventions.md) — the rules `npm run check` enforces
