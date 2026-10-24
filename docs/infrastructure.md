# Infrastructure Packages

Everything that is not `ai`, `agent`, `tui`, or `coding-agent`.

| Package | Status | One-liner |
|---|---|---|
| `telemetry` | **stable** | Vendor-neutral spans, metrics, propagation, and stored-trace contracts. |
| `chord` | **used, unstable** | Standalone object/contract runtime with replicated state, links, hot reload, and SQLite helpers. |
| `codemode` | **stable, used by CLI** | Sandboxed JavaScript whose only capability is calling injected tools. |
| `mcp` | **stable, used by CLI** | Standalone Model Context Protocol client with stdio/HTTP/OAuth. |
| `durable` | **experimental** | Durable conversation/task/document harness. Consumed by `env`, not the CLI. |
| `env` | **new, standalone** | Remote `ExecutionEnv`: a Rust daemon over SSH plus the client. No in-repo consumer. |
| `evals` | private, dev-only | Behavioral evals for the coding agent. |

There is no `protocol`, `client`, `server`, or `session-backends/sqlite-node` package in this
checkout. The old remote stack and its SQLite backend were removed.

## `telemetry` — `@earendil-works/pi-telemetry`

Contracts, not an exporter: no backend dependency, no ambient current-span state, no global
tracer. Runtime dependencies: none.

The package now ships:

- `TelemetryContext.startSpan(options, callback)` and `TelemetrySpan`; a span **is** a context, so
  child spans nest naturally.
- `SpanAttributes`, `SpanLink`, `SpanOptions`, `StoredSpan`/`StoredTrace`, and `TelemetryStore`
  query contracts.
- Optional `TelemetryPropagation` carriers and `TelemetryMetrics` counters/up-down
  counters/histograms.
- `NOOP_TELEMETRY_CONTEXT`, `NOOP_TELEMETRY_METRICS`, `NOOP_TELEMETRY_PROPAGATION`.
- `InMemoryTelemetryContext` and `./render` (`renderTrace`).
- `./testing`, an adapter conformance suite.

Design choices that will surprise you:

1. **There is no `span.end()`.** The callback settling *is* the end. Status is `ok` unless you set
   it or the callback throws.
2. **Completion attributes are optional.** Calling `setAttributes()` zero times is valid; it exists
   for early failures and cancellation.
3. `defineTelemetrySchema` is a **pure identity function with no runtime validation**. Duplicate
   span names across composed schemas are a compile error (`UniqueTelemetrySchemas`), and
   `ExactTelemetryAttributes` uses `Record<..., never>` to reject unknown keys. The safety is in
   the types.

Consumers: `pi-ai` (`AI_TELEMETRY_SCHEMA` in `packages/ai/src/telemetry.ts`), `chord`
(`Context.telemetry` and the `/telemetry-sqlite` adapter), and `durable`
(`DURABLE_TELEMETRY_SCHEMA` in `packages/durable/src/harness/telemetry.ts`).
`pi-coding-agent` declares no dependency on it.

`packages/chord/src/telemetry-sqlite/` implements `TelemetryStore` over SQLite with buffered
spans, ten-second metric aggregates, retention, and bounded queues.

## `chord` — `@earendil-works/chord`

A standalone runtime for applications made of **objects** that live in several processes and can
be hot reloaded.

- An **object** is state plus code at an address. It is created on demand and lives while someone
  references it.
- Everyone gets a **reference**, never the object. References work locally and across processes
  and survive reloads.
- A **facet** is a plugin's code for one host; a **registry** maps contracts to facets.
- **State** is observable JSON. Remote viewers get a snapshot followed by deltas.
- **Hot reload** swaps code while work in flight finishes on the old code; state is kept and
  incompatible changes migrate at a safe point.

Contracts are runtime values described with TypeBox schemas. Hosts exchange JSON frames over
links: in memory, a Node IPC channel, or a WebSocket.

Status: published, but unstable in practice — recent minor releases in `CHANGELOG.md` carry
`### Breaking Changes`. `PROTOCOL_VERSION = 4`; peers with different versions refuse to talk.

Exports (9 subpaths): `.`, `./context`, `./output`, `./delta`, `./node`, `./telemetry-sqlite`,
`./sqlite`, `./sqlite/node`, `./sqlite/cloudflare`. Runtime deps: `pi-telemetry` and `typebox`
only.

Module map:

| Path | What |
|---|---|
| `src/contract.ts`, `declare.ts`, `object.ts` | `local`/`remote` contracts, cells/resources/listeners, factories. |
| `src/host.ts`, `registry.ts`, `reload.ts` | `createHost`, `defineFacet`, `defineRegistry`, lifetimes, hot reload. |
| `src/connection.ts`, `link.ts`, `wire.ts`, `websocket.ts`, `node.ts` | Links, framing, protocol version, IPC/WebSocket transports. |
| `src/state.ts`, `cell.ts`, `watch.ts` | Replicated state, public subscriptions, watches. |
| `src/delta/` | Transactional JSON tracker/diff/apply plus a **normative** `README.md` with mutation rights. |
| `src/context/index.ts` | Explicit `Context`: cancellation + non-optional telemetry span. |
| `src/output.ts` | Bounded output streams and paced progress. |
| `src/sqlite/`, `src/telemetry-sqlite/` | Portable SQLite facade, migrations, and telemetry storage. |

Three design details worth knowing:

- **The delta engine refuses unknown verbs loudly** (`src/delta/index.ts:187-189`, `:273-275`):
  *"Silently skipping an unknown verb is how a newer producer's op vanishes."* It rejects
  `__proto__`/`constructor`/`prototype` path segments on every decode and apply, and rejects
  out-of-range indices that would allocate huge sparse arrays.
- **State is not frozen.** The tracker uses trusted immutable ownership rather than defensive
  copying; mutating a published value can corrupt authority. Clone or serialize at any mutable
  trust boundary (`src/delta/README.md`).
- **`Context` carries telemetry explicitly.** `BACKGROUND_CONTEXT` uses the no-op span; adapters
  start root spans and never travel inside a context.

Consumers: `durable` (documents, watches, delta, context) and `env` (context). The `pi` CLI does
not import it.

Docs: `README.md`, `docs/tutorial.md`, `docs/guide.md`, `docs/spec.md` (normative), and
`src/delta/README.md`.

## `codemode` — `@earendil-works/pi-codemode`

Runs model-written JavaScript in a QuickJS VM compiled to WebAssembly. The only capability is
calling injected tools; nested tool calls never enter the LLM context — only the script's output
and return value do.

- Node/Bun: `CodemodeSandbox` runs each execution in a worker thread.
- Cloudflare Workers/DO: `RemoteCodemodeSandbox` on `/portable`, with the deployable object in
  `/cloudflare`.
- `InlineCodemodeSandbox` runs on the calling thread for other runtimes.

Scripts use `tools`, `ALL_TOOLS`, `text()`, `image()`, `exit()`, `store()`/`load()`, `console`,
and `return`; they may start with a `// @options:` line. There are no Node APIs, network, timers,
modules, or `WebAssembly`.

Key invariants:

- `execute()` resolves with an outcome; script failures are data (`error.kind` is
  `script | timeout | aborted | sandbox`), not rejected promises.
- Output and store limits are explicit: 16 Mi chars / 100,000 output items, 256 Ki per stored
  value / 1 Mi total.
- `sandbox.start()` gives deterministic replay: virtual clock, seeded randomness, UTC, and
  ordered results.
- `portable` and `cloudflare` entries are budgeted so they cannot reach Node built-ins, the
  worker-thread host, or the file-based wasm loader.

Consumers: `coding-agent` (the built-in `codemode` tool and its worker) and `durable` (its
`./codemode` extension). `README.md` is the reference.

## `mcp` — `@earendil-works/pi-mcp`

A small, standalone Model Context Protocol client. It does not depend on the official MCP SDK or
on other Pi packages. Runtime dependency: `cross-spawn`.

- Transport-neutral `McpClient`; `StdioTransport`, `StreamableHttpTransport`, and an in-memory
  testing transport.
- `McpClient` owns initialization, request correlation, timeouts, cancellation, server requests,
  and protocol helpers: tools, resources, resource templates, progress, logging, and
  tool-list-change notifications.
- `/oauth` provides the MCP OAuth client subset, discovery, callback server, and credential
  provider; it does not open a browser or choose credential storage.
- Protocol target: MCP `2025-11-25`, negotiating down to `2024-11-05`. Batch JSON-RPC, legacy
  HTTP+SSE, servers, sampling, and tasks are out of scope.

Consumer: `coding-agent` (`src/extensions/mcp/`), including `pi mcp` and the
`test:mcp-conformance` harness.

## `env` — `@earendil-works/pi-env`

Remote execution environments for Durable:

- `daemon/` is a small Rust program that runs on the remote machine and speaks a framed protocol
  on stdin/stdout.
- `RemoteExecutionEnv` implements Durable's `ExecutionEnv` through a `Connection`.
- `ssh.ts` handles host-key scanning/acceptance, daemon deployment (named by SHA-256 and verified
  before every start), and SSH connection setup.
- `LocalExecutionEnv` is `NodeExecutionEnv` plus `listFiles()`, matching the remote daemon's
  behavior. `RemoteTerminal` and `RemoteWatcher` cover terminals and file watching.

Wire protocol v1: `u32 length | u8 type | u32 id | u32 jsonLength | json | bytes`, 16 MiB cap, a
sync line that discards shell startup noise, and framing errors that end the connection. SSH runs
with batch mode, strict host-key checking, no forwarding, no shared connections, and no configured
commands.

No package in the repo consumes `env`; it is published for Durable applications. Docs:
`README.md`, `docs/protocol.md`, `docs/semantics.md`.

## `durable` — `@earendil-works/pi-durable`

**Experimental.** A durable agent harness: conversations, model turns, tool calls, and your own
state are committed to storage before anything is shown. If the process dies mid-turn, reopening
the storage resumes from the last commit.

Built on `pi-ai` for model access, `chord` for document state, `pi-codemode` for code mode, and
`pi-telemetry` for spans/metrics. `env` depends on it; the `pi` CLI does not.

Core model:

- **Harness**: one open storage plus the scheduling machinery. Every change goes through one
  atomic commit line; nothing is published before its commit is stored.
- **Conversation**: a transcript of immutable entries. `root()` creates the root conversation;
  conversations can fork.
- **Entry**: one immutable transcript record (`pi.user`, `pi.assistant`, `pi.tool-result`,
  `pi.system`, `pi.reset`, or an app kind).
- **Task**: a durable state machine that checkpoints every step. Built-in tasks are
  `pi.generation` and `pi.tool`; tasks can own child tasks.
- **Submission**: input or an entry handed to a conversation, with `wait()` for its answer.
- **Document**: typed JSON changed in the same commit as entries. Built-ins cover agent choices,
  provider identity, live state, inbox, usage, and nested calls.

Features: resumable/crash-safe tasks, steering and follow-ups, compaction, retries, tool hooks,
nested tool calls, tool search/deferred tools, codemode with journaled replay, image processing,
per-conversation usage/cost, task graphs, forking, read-only sessions, and shared storage/env
conformance suites.

Storage: memory, JSONL, IndexedDB, and SQLite (Node and Cloudflare). SQLite uses WAL +
`synchronous = NORMAL`; JSONL `fsync` is off by default. Acknowledged commits survive process
crashes, but the newest may be lost after a power or host failure. One process owns a storage at a
time; there is no cross-process locking.

Errors: a storage error is fatal to the open Session; an invalid read fails only that read; user
code fails only its own unit (`SessionFailed`, `ReadOnlySession`, `ConversationBusy`,
`ReadAfterWrite`, `StorageRequestError` in `src/errors.ts`).

Docs: `README.md` (usage) and `docs/spec.md` (normative). `./testing` exports
`createStorageConformance`, `createEnvConformance`, and runner helpers.

## `evals` — `@earendil-works/pi-evals`

`"private": true`; no runtime `dependencies`, seven devDependencies including `pi-ai`,
`pi-coding-agent`, and `vitest-evals`. It is an application, not a library: no `src/index.ts`.

- `evals/*.docs.eval.ts` are documentation-lift evals. `eval:docs` runs each case in
  `without_docs` and `with_docs` containers and reports lift.
- Other `evals/*.eval.ts` files are host evals, run by Vitest on this machine.
- Runner in `src/` (`cli.ts`, `docker.ts`, `plan.ts`, `report.ts`, `harness.ts`).
- Two Docker images are built once per comparison run; cases are discovered in both and must
  match. Every `(case, variant, model, runNumber)` arm is planned before execution and run in a
  fresh container.
- A pair counts toward lift only when **both** arms produce exactly one score; missing, duplicate,
  skipped, pending, unscored, or errored arms block the pair and withhold headline lift.
  Repetition order alternates by run number.
- Requires `PI_PROVIDER` and `PI_MODEL`, or `--provider`/`--model`.

Artifacts may contain prompts, responses, generated code, and tool output. Docker alone cannot
prove agent-written code never touches the network.

## Related

- [architecture.md](architecture.md) — package map and dependency order
- [development.md](development.md) — build order, which is this dependency graph
