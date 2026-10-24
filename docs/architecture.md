# Architecture

## The monorepo in one picture

```
                         coding-agent (the `pi` CLI)
                         modes: interactive, print, json, rpc
                          │        │        │        │
              ┌───────────┘        │        │        └───────────┐
              ▼                    ▼        ▼                    ▼
          agent core             tui       ai             codemode / mcp
        (agent + loop)             │     (providers)        (sandbox, client)
              │                    │        │
              └────────────────────┴────────┴──────────────► telemetry

   Standalone:  chord (objects/contracts) ──► durable (experimental harness) ──► env
                                                   │
                                                   └──► ai, codemode, telemetry
```

There is no protocol/client/server stack and no second CLI runtime in this checkout.
`packages/coding-agent/src/experimental/` contains only a source resolver used by `pi-test.sh`
and codegen scripts.

## Packages and dependency direction

Build order is the dependency graph (`package.json:16`):

```
telemetry → chord → tui → codemode → mcp → ai → durable → env → agent → coding-agent
```

Workspace dependencies are exact pins, not ranges:

| Package | Name | Depends on |
|---|---|---|
| `telemetry` | `@earendil-works/pi-telemetry` | *(nothing)* |
| `chord` | `@earendil-works/chord` | `telemetry`, `typebox` |
| `tui` | `@earendil-works/pi-tui` | `get-east-asian-width`, `marked`, `marked-cjk-friendly` |
| `codemode` | `@earendil-works/pi-codemode` | `quickjs-wasi` |
| `mcp` | `@earendil-works/pi-mcp` | `cross-spawn` |
| `ai` | `@earendil-works/pi-ai` | `telemetry`, provider SDKs, `typebox` |
| `durable` | `@earendil-works/pi-durable` | `chord`, `ai`, `codemode`, `telemetry` (+ Photon, `diff`, `typebox`) |
| `env` | `@earendil-works/pi-env` | `chord`, `durable` |
| `agent` | `@earendil-works/pi-agent-core` | `ai`, `typebox` |
| `coding-agent` | `@earendil-works/pi-coding-agent` | `agent`, `ai`, `tui`, `codemode`, `mcp` (+ app deps) |
| `evals` | `@earendil-works/pi-evals` | *(private; no runtime deps)* |

`chord` deliberately depends on no Pi application package. It is a general object/contract
runtime and is used here by `durable` and `env`, not by the `pi` CLI.

## Where a feature belongs

`CONTRIBUTING.md:7-11` is the rule: pi's core is minimal; features that do not belong in the
core should be an extension.

| You want to… | Put it in | Not in |
|---|---|---|
| Add a provider, API, model, image model, or classifier | `packages/ai/src/providers/`, `src/api/`, `scripts/generate-models.ts` | `coding-agent` |
| Add a tool, command, event handler, CLI flag, or keybinding | an extension under `packages/coding-agent/examples/extensions/` | `core/tools/`, `core/extensions/types.ts` |
| Add a terminal widget or key handling | `packages/tui/src/components/` or `src/keys.ts` | `coding-agent` |
| Change the core agent loop | `packages/agent/src/agent-loop.ts` | `durable` |
| Add a durable task, document, or storage backend | `packages/durable/src/` (+ `docs/spec.md`) | `packages/agent` |
| Add sandbox/tool-calling script semantics | `packages/codemode/src/` | `coding-agent` |
| Add MCP protocol or transport behavior | `packages/mcp/src/` | `coding-agent` |
| Add user-facing documentation | `packages/coding-agent/docs/` | these internal docs |

The CLI ships **four** built-in extensions (`src/extensions/index.ts`):
`llama.cpp` (not replaceable) and `codemode`, `tool-search`, `mcp` (replaceable by an extension
that registers the same tool/command). It also carries 70 single-file example extensions and 9
directory examples. That ratio is the intended shape: built-ins are infrastructure, examples
show the extension API.

## Request lifecycle (shipping path)

Follow one prompt through the CLI:

1. `src/modes/interactive/interactive-mode.ts`, `modes/print-mode.ts`, or `modes/rpc/rpc-mode.ts`
   calls into `AgentSession` (`core/agent-session.ts`).
2. `core/sdk.ts:191` `createAgentSession()` builds a `pi-agent-core` `Agent` at
   `sdk.ts:403-453`. Its `streamFn` (`:412-423`) calls `ModelRuntime.streamSimple`.
3. `Agent.prompt()` / `Agent.continue()` (`packages/agent/src/agent.ts:382-405`) reject
   concurrent runs, then `runWithLifecycle` (`agent.ts:517`) creates the `AbortController` and
   calls `runAgentLoop` / `runAgentLoopContinue`.
4. `runAgentLoop` (`packages/agent/src/agent-loop.ts:111-160`) announces tool changes, emits
   `agent_start`/`turn_start`, then enters `runLoop` (`agent-loop.ts:171-330`).
5. Each turn: `prepareNextTurn` (optional) → pending messages → `prepareRequest` (optional) →
   `streamAssistantResponse` (`:390`) → stop/error check → `executeToolCalls` (`:518`) →
   `finishTurn` (optional) → `turn_end`; then steering and follow-up queues are polled.
6. Tool execution picks sequential (`:540`) or parallel (`:596`). It is sequential when
   `toolExecution === "sequential"` or any addressed tool has `executionMode: "sequential"`.
   `beforeToolCall` and `afterToolCall` wrap each call around `prepareToolCall`/finalization.
7. The stream goes through pi-ai: `ModelRuntime` → `Models.stream()` →
   `Model.compat` flags → `src/api/<api>.lazy.ts` → a concrete `src/api/<api>.ts` →
   provider SDK or hand-rolled HTTP, emitting `AssistantMessageEvent`s.
8. `Agent.processEvents` (`agent.ts:575`) reduces events into `AgentState` and awaits listeners
   in registration order. `AgentSession` persists entries, drives the TUI, and emits extension
   events from there.

## Agent runtimes

| Runtime | Entry | State | Consumers |
|---|---|---|---|
| **Core agent** | `packages/agent/src/agent.ts`, `agent-loop.ts` | in-memory `AgentState`; transcript lives in messages | `coding-agent`, SDK users |
| **Durable harness** | `packages/durable/src/harness/` | committed conversations, entries, tasks, documents; crash recovery | `env`, standalone apps; **not the CLI** |

The core agent is provider-agnostic and only depends on `pi-ai` types and `StreamFn`. The
coding agent installs `ModelRuntime.streamSimple` as its stream function and wires extensions
around the loop.

The durable harness is a separate implementation with its own task scheduler, storage
backends, tool pipeline, compaction, and Chord-backed documents. See
[infrastructure.md](infrastructure.md#durable--earendil-workspi-durable).

## Reading order for a first change

- **CLI:** `src/cli.ts` → `src/main.ts` → `core/sdk.ts` → `core/agent-session.ts` →
  `modes/interactive/interactive-mode.ts`.
- **Loop:** `packages/agent/src/types.ts` (539 lines, all contracts) →
  `agent-loop.ts` (`runLoop`, then `executeToolCalls`) → `agent.ts` (`processEvents`).
- **Providers:** `packages/ai/src/types.ts` → `models.ts` (`Models` / `createProvider`) →
  one `src/api/<x>.ts` and one `src/providers/<x>.ts`.
- **Extensions:** `packages/coding-agent/src/core/extensions/types.ts` →
  `examples/extensions/` → `docs/extensions.md`.

## Related

- [development.md](development.md) — build, check, test, run from source
- [conventions.md](conventions.md) — the rules `npm run check` enforces
