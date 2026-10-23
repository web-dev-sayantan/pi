# Pi Developer Onboarding

Internal docs for engineers new to the `pi` monorepo. Everything here is derived from the
code — file paths are real, line numbers are from the current checkout.

## What pi is

Pi is an agent harness: a runtime for LLM tool-calling loops (`packages/agent`), a unified
multi-provider LLM API (`packages/ai`), a terminal UI library (`packages/tui`), and a
self-extensible coding agent CLI built on all three (`packages/coding-agent`).

## Read in this order

| # | Doc | Read it when |
|---|-----|--------------|
| 1 | [architecture.md](architecture.md) | You need the package map, dependency direction, and where a feature belongs. |
| 2 | [development.md](development.md) | You are about to build, run, or test something. |
| 3 | [conventions.md](conventions.md) | You are writing code that must pass `npm run check`. |
| 4 | [agent-runtime.md](agent-runtime.md) | You are touching `packages/ai` or `packages/agent`. |
| 5 | [coding-agent.md](coding-agent.md) | You are touching `packages/coding-agent` (the `pi` CLI). |
| 6 | [tui.md](tui.md) | You are touching `packages/tui`. |
| 7 | [infrastructure.md](infrastructure.md) | You are touching chord, durable, telemetry, protocol, client, server, or evals. |

Existing docs you should prefer over these: `AGENTS.md` (repo rules for humans and agents),
`CONTRIBUTING.md` (contribution gate), `packages/agent/docs/harness.md` (the normative harness
spec), `packages/coding-agent/docs/` (user- and extension-facing documentation).

## The one thing to know first

`packages/agent` ships **three independent agent runtimes**. They share types and nothing else.

| Runtime | Entry | State | Used by |
|---------|-------|-------|---------|
| Core agent | `packages/agent/src/agent.ts` | in-memory, process-local | `packages/coding-agent` via `core/sdk.ts` |
| Harness | `packages/agent/src/harness/` | durable, crash-recoverable, JSONL/SQLite | not yet wired into the CLI |
| Pico3 | `packages/agent/src/harness/pico3/` | experimental kernel, `./experimental/pico3` | `coding-agent/src/experimental/` |

`src/harness/` never imports `agent.ts` or `agent-loop.ts`. If you were told to "change the
agent loop", confirm which of the three you mean. Only the core agent is on the hot path for
the shipped `pi` binary.

## The second thing to know

There are **two stacks** plus a shared vocabulary package.

- **Shipping** (what users run): `tui` + `ai` + `agent` + `coding-agent`, with JSONL session
  storage in `agent/src/harness/session/`.
- **Experimental** (gated on `PI_EXPERIMENTAL=1`, in `coding-agent/src/experimental/`): `chord`
  + `durable` + `protocol` + `client` + `server`.
- **`chord`** is shared by both, which is why `pi-agent-core` depends on it and re-exports its
  context, even though every Chord *service* consumer sits in `src/experimental/`.

Three packages have **no in-repo consumers** and should not be treated as load-bearing:
`durable`, `session-backends/sqlite-node`, and `evals` (private). See
[infrastructure.md](infrastructure.md).

## Quick start

```bash
npm install --ignore-scripts   # do not run lifecycle scripts
npm run build:offline         # needs packages/ai/src/providers/data/ (see development.md)
npm run check                 # biome + 7 repo invariants + tsc --noEmit
./test.sh                     # full suite, isolated HOME, no API keys
./pi-test.sh                  # run pi from TypeScript sources
```
