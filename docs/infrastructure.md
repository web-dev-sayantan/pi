# Infrastructure Packages

Everything that is not `ai`, `agent`, `tui`, or `coding-agent`. Most of it is experimental or
unwired — read the status column before you invest.

| Package | Status | One-liner |
|---|---|---|
| `telemetry` | **production, stable** | Vendor-neutral telemetry contracts, type-level only. |
| `chord` | **real dependency, explicitly unstable** | Application-composition runtime: facets, services, replicated state, JSON deltas, context. |
| `protocol` | experimental, load-bearing for client/server | Routed envelopes + hand-written CBOR + framing. `PROTOCOL_VERSION = 8`. |
| `client` | experimental | Transport-neutral client for the protocol. No auto-recovery. |
| `server` | experimental | Multi-presentation attachment host. Forwards opaque envelopes. |
| `durable` | **unwired** | Durable conversation/task/document runtime ("Pico"). No in-repo consumers. |
| `session-backends/sqlite-node` | **dead** | `node:sqlite` session backend, superseded. No consumers. |
| `evals` | private, dev-only | Behavioral evals; builds two Docker images per case. |

## `telemetry` — `@earendil-works/pi-telemetry`

Contracts only: no exporter, no `AsyncLocalStorage`, no ambient current-span state, no backend
dependency (`README.md:11-13`). Zero dependencies at all.

The API is 22 lines: `TelemetryContext.startSpan(options, cb)` and `TelemetrySpan` — and a span
**is** a context, so child spans nest naturally.

Three design choices that will surprise you:

1. **There is no `span.end()`.** The callback settling *is* the end. Status is `ok` unless you set
   it or the callback throws.
2. **`endAttributes` are always optional** ("completion enrichment"). Calling `setAttributes()`
   zero times is valid — it exists for early failures and cancellation (`README.md:317`).
3. `defineTelemetrySchema` is a **pure identity function with no runtime validation**. Duplicate
   span names across composed schemas are a *compile* error (`UniqueTelemetrySchemas`), and
   `ExactTelemetryAttributes` uses `Record<..., never>` to reject unknown keys. The safety is all
   in the types.

Consumers: `pi-ai` (request options), `pi-agent-core` (which owns
`AI_TELEMETRY_SCHEMA` / `HARNESS_TELEMETRY_SCHEMA` / `AGENT_TELEMETRY_SCHEMAS` in
`src/harness/telemetry.ts`, and the context key in `src/harness/context.ts:12`).
`pi-coding-agent` does **not** import it.

`packages/agent/docs/telemetry-schema.md` is **generated** from those schema constants by
`packages/agent/scripts/generate-telemetry-docs.ts` (one function, `--check` mode fails when
stale). Do not edit it by hand.

## `chord` — `@earendil-works/chord`

The most confusing package, because it is a real dependency of six packages while insisting it
is not a Pi package.

`README.md:1-8`: *"It is developed as a standalone package in the Pi monorepo, but it is not a Pi
package: it does not depend on any other Pi workspace package and can be used by unrelated
applications."*

`PLANNING.md:3`: *"**This is not a stable public API contract yet.**"* And `PLANNING.md:26-37`
imposes hard constraints: no Pi deps, no imports from `@earendil-works/pi-*` or outside
`packages/chord`, application-neutral vocabulary, Node-specific loading kept out of the
platform-neutral runtime — plus a **banned-word list**: *Session, Harness, AgentLane, server,
client, attachment, TUI, model, tool, hook, provider credential, workspace* must not become Chord
concepts.

Its headline feature (symmetric RPC) is still planned, not implemented.

### Modules

| Path | What |
|---|---|
| `src/api.ts` | `defineFacet`, `defineService`, `createFacetHost`, `replicatedState`, `createRemoteServiceBinding`. `defineService` throws on the reserved `$chord.` prefix. |
| `src/services/` | 10 files. `provider.ts` (authoritative side), `consumer.ts`, `state.ts` (replication core), `state-codec.ts`, `wire.ts`, `instances.ts`, `handle.ts`, `loopback.ts`, `errors.ts`, `state-internals.ts`. |
| `src/context/index.ts` | Go-style `Context`: `BACKGROUND_CONTEXT`, `createContextKey`, `withContextValue`, `withAbortSignal` (via `AbortSignal.any`), `withCancel`, `awaitWithContext`. `toString()` mirrors Go (`"[Context BACKGROUND_CONTEXT]"`). |
| `src/delta/` | The JSON delta engine plus a **normative** `src/delta/README.md` with a mutation-rights table. |
| `src/facets/` | `host.ts` (`FacetKernel`, lifecycle + generation state machines), `loader.ts`, `README.md`. |
| `src/node/`, `src/bundler.ts` | esbuild-to-CJS facet bundling, `chord-facets.json` manifest (format 2, per-entry SHA-256), `node:vm` loading that bypasses Node's module cache. |
| `src/json.ts` | `copyJson` — a strict validator rejecting non-finite numbers, cycles, non-plain prototypes, symbol keys, sparse arrays, and accessors. |

### Three Chord design details worth knowing

- **`MemberSlot` is a `Proxy` over a callable** (`services/consumer.ts:24-64`). Applying it invokes
  the remote method, `.value` returns local replicated state, `.subscribe` returns an unsubscribe
  function, and `then` / `Symbol.toStringTag` are handled so the proxy never looks thenable. That
  is how a remote service keeps a **stable facade** across disconnects and replacements.
- **The delta engine refuses unknown verbs loudly.** `src/delta/index.ts:187-189` and `:273-275`:
  *"Silently skipping an unknown verb is how a newer producer's op vanishes."* It also blocks
  prototype-pollution paths (`RESERVED_SEGMENTS = {__proto__, constructor, prototype}`, checked on
  every decode *and* every apply, because paths are data that can echo model output) and rejects
  out-of-range indices that would allocate huge sparse arrays. Writes use `Object.defineProperty`
  so an inherited setter cannot run.
- **Facet reload keeps service handles connected.** `src/facets/README.md:213-221`: candidates are
  activated and validated *while current providers stay routed*, so stable service handles never
  go disconnected during an ordinary reload.

### How `pi-agent-core` uses it — two very different ways

1. **Context is fully replaced.** `packages/agent/src/harness/context.ts:14-25` re-exports eight
   primitives straight from `@earendil-works/chord/context` and adds one pi-specific key
   (`TELEMETRY_CONTEXT_KEY`). All of `pi-agent-core`'s cancellation and context propagation *is*
   Chord's implementation.
2. **Pico3 speaks Chord service semantics.**
   `packages/agent/src/harness/pico3/chord.ts:42-43` defines `pi.harness` (local-only) and
   `pi.conversation` (remote-exposable) services. `attachChordView` (`:81-150`) bridges Pico's
   commit-granular watch envelopes to one Chord publication per commit: the raw listener only
   **enqueues**, and a microtask drain applies ops and publishes with *"no await between them"*.
   The queue is bounded (`WATCH_CAPACITY`); overflow deliberately **fails the bridge** and tells
   the owner to respawn its keyed service instance rather than silently dropping updates.

Chord's delta engine is also used directly by `pico3/{view,memory,session,jsonl,types}.ts` and
`pico3/legacy-tracker.ts`.

## `protocol`, `client`, `server` — the experimental remote stack

These three exist together. `PROTOCOL_VERSION = 8`, and `README.md` says outright: *"The
protocol is experimental and has no compatibility guarantees."* Peer authentication is **not**
implemented.

**`protocol`** — `src/index.ts` is 22 lines over four modules: hand-written CBOR (`src/cbor/`, no
`cbor` dependency), `codec.ts`, `framing.ts`. The route is
`{serverId, sessionId, attachmentId}` — a combined durable address fencing a call to one server,
one durable session, and one live presentation attachment. The frame is a 4-byte big-endian length
plus one definite-length CBOR item, and decoders accept arbitrary fragmentation and coalescing.
Strictness: all envelope schemas reject unknown properties, codecs recursively reject non-JSON
payloads, and limits are 16 MiB payload / 1,000,000 entries / 64 nesting levels.
`attach()` / `detach()` return **no** routing ids — the server publishes the live route out of band.

**`client`** — `Client`, `createClientServiceTransport`, `ByteTransport`, `ConnectionState`.
It verifies the physical endpoint reports the expected logical `serverId` at handshake. It
**deliberately does not construct typed service proxies** (`README.md:30`) — typed APIs come from
application-owned Chord bindings. And there is **no auto-recovery** (`README.md:34`): on
disconnect, pending requests reject locally but accepted work may still complete remotely. Callers
must `reconnect()`, re-attach, and repeat only safe operations. The `./unix` subpath
(`createUnixTransportFactory`, `discoverUnixServers`) derives each expected server id from the
socket filename and verifies via handshake, skipping malformed/stale entries.

**`server`** — `RoutedServerServiceHost.attachClient()` gives a connection-scoped server endpoint;
`RoutedSessionHandle.attachClient()` gives a presentation-scoped Session capability whose
`invokeService()` **forwards the opaque envelope without server-side business decoding**. Repeat
`attach` from one connection is idempotent. Each attachment gets a server-generated id delivered
only as routing control data. **The real `Session` and `AgentHarness` never cross the process
boundary.**

All three are consumed only from `packages/coding-agent/src/experimental/`
(`server.ts`, `client-runtime.ts`, `client-tui.ts`, `session-worker.ts`,
`session-worker-manager.ts`, `radius-relay.ts`) plus two tests each. They are devDependencies of
`coding-agent`, not runtime dependencies.

## `durable` — unwired

`@earendil-works/pi-durable`: *"Durable conversation, task, and document runtime for Pi… This
package contains the Pico runtime."*

**No package in the monorepo imports it.** Grep finds only its own tests, its own vitest alias,
the root `tsconfig.json` path map, and `scripts/durable-browser-smoke-entry.ts`. It is a parallel
storage stack that `pi-agent-core`'s own `harness/session/*` reimplements.

Worth knowing if you touch it:

- The root entry is runtime-neutral. SQLite adapters for Bun / Cloudflare Durable Objects
  implement a **synchronous** `SqliteDatabase` facade (`README.md:30-32`); remote async APIs like
  Cloudflare D1 *"cannot implement this synchronous facade; they require a dedicated `Storage`
  backend."*
- Durability guarantees are deliberately weakened: JSONL `fsync` defaults to `false`; SQLite uses
  WAL + `synchronous = NORMAL` and checkpoints on close, so *"acknowledged commits survive
  process crashes, but the newest commits may be lost after a power or host failure."*
- Single-owner only: cross-process locking and ID allocation are not supported.

Design docs: `packages/durable/docs/pico-v5.md`, `pico-v5-handoff.md`, `pico-v5-chord-usage.md`.

## `session-backends/sqlite-node` — dead

`@earendil-works/pi-session-backend-sqlite-node`. `src/index.ts` wraps `node:sqlite`'s
`DatabaseSync` and exports a self-contained `src/sqlite/` implementation. No consumers outside its
own directory (only `tsconfig.json` and `package-lock.json` reference it). It was renamed from
`@earendil-works/pi-storage-sqlite-node` (`CHANGELOG.md:37`) — a leftover of a superseded storage
design.

It is **not** the same thing as `pi-durable`'s `storage/sqlite/node`.

## `evals` — private, dev-only

`"private": true`, and the only workspace with no `dependencies` at all. Behavioral evals for the
coding agent: `*.docs.eval.ts` (documentation lift) and `*.host.eval.ts`. Runner in `src/`
(`cli.ts`, `docker.ts`, `plan.ts`, `report.ts`, `harness.ts`). No `src/index.ts` — it is an
application, not a library.

Methodology (`README.md:71-107`): build **two** Docker images (`without_docs` / `with_docs`),
pack the workspace, plan every `(case, variant, model, runNumber)` arm up front, run each arm in a
fresh container, and only count a pair when **both** arms produce exactly one score — otherwise the
pair is blocked and headline lift is withheld. Repetition order alternates by run number to reduce
order bias. Requires `PI_PROVIDER` and `PI_MODEL`.

Cost note: artifacts may contain prompts, responses, generated code, and tool output. The README
also notes that Docker alone cannot prove agent-written code never touches the network.

## Related

- [architecture.md](architecture.md) — the two-stack diagram and dependency order
- [development.md](development.md) — build order, which is this dependency graph
