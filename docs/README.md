# Pi Developer Onboarding

Internal docs for engineers new to the `pi` monorepo. Everything here is derived from the
current checkout; file paths are real, and line numbers are from this tree.

Every workspace package shares version `1.4.0`; all require **Node >= 22.19.0**.

## What pi is

Pi is a minimal, extensible agent harness and a coding-agent CLI built on it.

- `packages/agent` — the core agent loop: tool calling, streaming, state, events.
- `packages/ai` — one API for many LLM providers, plus image and classifier models.
- `packages/tui` — the terminal UI library used by the CLI.
- `packages/coding-agent` — the `pi` CLI, SDK, extensions, MCP, and codemode.

Supporting libraries: `codemode` (sandboxed tool-calling JavaScript), `mcp` (MCP client),
`telemetry` (explicit spans and metrics), and `chord` (standalone object/contract runtime).

## Read in this order

| # | Doc | Read it when |
|---|-----|--------------|
| 1 | [architecture.md](architecture.md) | You need the package map, dependency direction, and where a feature belongs. |
| 2 | [development.md](development.md) | You are about to build, run, or test something. |
| 3 | [conventions.md](conventions.md) | You are writing code that must pass `npm run check`. |
| 4 | [agent-runtime.md](agent-runtime.md) | You are touching `packages/ai` or `packages/agent`. |
| 5 | [coding-agent.md](coding-agent.md) | You are touching `packages/coding-agent` (the `pi` CLI). |
| 6 | [tui.md](tui.md) | You are touching `packages/tui`. |
| 7 | [infrastructure.md](infrastructure.md) | You are touching chord, telemetry, codemode, mcp, durable, env, or evals. |

Prefer these existing docs when they overlap: `AGENTS.md` (repo rules), `CONTRIBUTING.md`
(contribution gate), `packages/coding-agent/docs/` (44 user- and extension-facing docs), the
package `README.md` files, and `.pi/skills/` (`add-llm-provider.md`, `github-work.md`,
`interactive-testing.md`, `release.md`).

## The one thing to know first

There are **two agent runtimes**, and they share no implementation.

| Runtime | Entry | State | Used by |
|---------|-------|-------|---------|
| Core agent | `packages/agent/src/agent.ts`, `agent-loop.ts` | in-memory, process-local | `packages/coding-agent` via `core/sdk.ts` |
| Durable harness | `packages/durable/src/harness/` | durable, crash-recoverable, commits before publishing | not wired into the CLI; used by `packages/env` and standalone apps |

The `pi` binary uses the **core agent**. The durable harness is an experimental, separate
runtime with its own conversations, tasks, documents, storage backends, and tool pipeline.
`packages/agent` has no harness, session backend, or pico3 code anymore. If someone says
"change the agent loop", confirm they mean `packages/agent/src/agent-loop.ts`.

## The second thing to know

There is no experimental client/server stack in this checkout.

- **CLI stack:** `coding-agent` → `agent` + `ai` + `tui` + `codemode` + `mcp`.
- **Durable stack:** `durable` → `chord` + `ai` + `codemode` + `telemetry`; `env` adds a
  remote `ExecutionEnv` over SSH.
- **`chord`** is a general-purpose runtime, not a Pi-specific layer. It depends only on
  `telemetry` and `typebox`; Pi's CLI does not import it.
- `protocol`, `client`, `server`, and `session-backends/sqlite-node` no longer exist.
- `evals` is private and dev-only. `env` has no in-repo consumer besides its own tests.

## Quick start

```bash
npm install --ignore-scripts   # do not run lifecycle scripts
npm run hydrate:model-data    # required once: packages/ai/src/providers/data/ is gitignored
npm run build:offline         # or npm run build, which hydrates first
npm run check                 # biome + invariants + tsc + browser smoke
./test.sh                     # full suite, isolated HOME, no API keys
./pi-test.sh                  # run pi from TypeScript sources
```

`npm run check` does not run tests. `./test.sh` is the supported test entry point.
