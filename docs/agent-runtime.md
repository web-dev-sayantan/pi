# Agent Runtime: `packages/ai` and `packages/agent`

## `packages/ai` — unified multi-provider LLM API

### Layering

```
providers/<id>.ts        Provider factory: id, name, baseUrl, auth, models, api
  └─ api/<id>.lazy.ts    4-line lazy wrapper: () => lazyApi(() => import("./<id>.ts"))
       └─ api/<id>.ts    the wire protocol: stream() + streamSimple()
            └─ vendor SDK (or hand-rolled SSE)
providers/data/<id>.json generated catalog (gitignored)
  └─ providers/<id>.models.ts   generated typed shard (committed)
       └─ models.generated.ts   generated aggregator (committed)
```

### The two load-time invariants

**1. The root barrel pulls zero SDKs.** `src/index.ts:4-8` is explicit: no generated catalogs, no
provider factories, no api-registry, no OAuth implementations, no compat. Enforced by
`test/lazy-module-load.test.ts`, which spawns a process and asserts the count. `scripts/check-browser-smoke.mjs:100-142`
re-checks it through an esbuild browser bundle.

**2. `stream()` is synchronous; setup is asynchronous.** `src/api/lazy.ts` is the keystone:
`lazyStream(model, setup)` returns the stream *now* and pushes any setup failure as an `error`
event, which is what lets `Models.stream()` be sync while auth resolution is async.

### Core types (`src/types.ts`, 1133 lines)

| Type | Line | Note |
|---|---|---|
| `Api` | 17-37 | `KnownApi \| (string & {})` — the `& {}` keeps autocomplete while allowing custom strings. |
| `Context` | 697-714 | Caller-facing input: `{ systemPrompt?, messages, tools? }`. |
| `TranscriptContext` | 697-714 | `{ messages }` **branded with a `unique symbol`**. Only `normalizeContext()` produces it, so a raw `Context` cannot reach a provider. Breaking change in 0.86.0. |
| `SystemMessage` | 515-531 | Carries `sections: Record<string, string \| null>` plus `toolsAdded` / `toolsRemoved`. **The leading system message *is* the system prompt + tool set**; later ones mutate it. |
| `AssistantMessage` | 539-561 | `content: (Text \| Thinking \| ToolCall)[]`, `usage`, `stopReason`, `errorMessage?`, `diagnostics?`. |
| `AssistantMessageEvent` | 716-748 | `start \| text_start/delta/end \| thinking_* \| toolcall_* \| done \| error`. Every non-terminal event carries `partial` — a **shared live accumulator, not a snapshot**. Events for different blocks are not guaranteed contiguous. |
| `StreamOptions` | 187-237 | `temperature`, `maxTokens`, `transport`, `cacheRetention`, `sessionId`, `onProviderStreamEvent`, `thinkingBudgets`, … |
| `Model<TApi>` | 1061-1133 | `compat` is a **conditional type keyed on the api id** — `hasApi(model, "anthropic-messages")` is what unlocks full option typing. `type?: "chat"` is optional so existing catalogs keep working; use `getModelType()` / `isModelType()`, never `model.type === "chat"`. |
| `Usage` | 420-441 | `reasoning` is a **subset of `output`**; `cacheWrite1h` a subset of `cacheWrite`. |

`ProviderStreams` (278-299) has exactly two methods — `stream` and `streamSimple` — plus optional
`fetchDeferred` / `cancelDeferred`.

### Providers are plain objects, not a class hierarchy

There is no base class. Every factory is 6-15 lines of
`createProvider({ id, name, baseUrl, auth, models, api })`. `createProvider` accepts a single
`ProviderStreams` or a `Partial<Record<TApi, ProviderStreams>>` map and dispatches on `model.api`
(`src/models.ts:1029-1070`). 43 providers live in `src/providers/`; `radius.ts` is the one
hand-written `Provider` because it has custom refresh logic.

`OpenAICompletionsCompat` alone has ~28 capability flags (`types.ts:754-1019`): `thinkingFormat`
with 11 encodings, `sessionAffinityFormat`, `vllmPriority`, `supportsStrictMode`, … Providers
differ almost entirely through these flags plus their `auth`.

### Auth resolution (`src/auth/resolve.ts:27-93`)

1. `overrides.apiKey` wins outright.
2. **A stored credential owns its provider.** If one exists, only that path is taken — no env
   fallback, ever, not even after a failed refresh. This is a deliberate departure from the
   usual "env first, then stored".
3. Only when nothing is stored does `apiKey.resolve()` consult ambient sources (env vars, AWS
   profiles, ADC files).

OAuth refresh is **double-checked locked** (`resolve.ts:110-162`): a token with <5 min validity
triggers `CredentialStore.modify()`, which re-checks expiry *under the lock* so concurrent
processes refresh exactly once. `CredentialStore.modify` is the only write path
(`auth/types.ts:78-90`).

### Two independent retry layers

| Layer | File | Scope |
|---|---|---|
| HTTP | `src/utils/provider-retry.ts` | one request, inside the adapter. Every adapter calls the SDK with `maxRetries: 0` and wraps it. Retryable = `x-should-retry` header, else 408/409/429/≥500. If the server asks for more than `maxRetryDelayMs` (default 60 s) it **throws immediately** rather than sleeping silently. |
| Turn | `src/utils/retry.ts` | whole assistant turns, via `retryAssistantCall`. Classification is **string-pattern based on `errorMessage`** — a non-retryable list (quota/billing) is checked before a retryable one (`overloaded`, 429/5xx, `ENOTFOUND`, …). Aborts are terminal. |

### Context overflow

`src/utils/overflow.ts` holds ~24 provider-specific regexes plus three non-error overflow signals
(z.ai silent overflow, Xiaomi MiMo `length` + zero output + context >99%, Cerebras bodyless
400/413). `isContextOverflow` is how compaction gets triggered.

### Other things worth knowing

- `src/providers/faux.ts` is the **in-repo test provider** and is part of the public API
  (`index.ts:35`). It simulates streaming and simulates prompt caching by diffing the common
  prefix of consecutive prompts.
- Prompt caching is four separate mechanisms selected by `compat`: `cache_control` breakpoints
  (Anthropic, applied even on OpenAI-shaped endpoints), OpenAI `prompt_cache_key` (clamped to 64
  chars) + `prompt_cache_retention`, `cachePoint` (Bedrock), and session-affinity **headers**.
  Every adapter duplicates a 6-line `resolveCacheRetention()`.
- `calculateCost` (`src/models.ts:1187-1207`) mutates `usage.cost` in place, applies
  `cost.tiers[]`, and special-cases Anthropic 1h cache writes at 2× the input rate.
- `src/compat.ts` is the **old global API preserved verbatim**, scheduled for deletion with the
  coding-agent ModelManager migration. Its `registerBuiltInApiProviders()` is non-clobbering so
  test/extension overrides survive. `src/legacy-api-aliases.ts` is 16 deprecated `streamX` consts
  reachable only from `/compat`.
- `src/cli.ts` (the `pi-ai` bin) is a standalone OAuth login tool writing `./auth.json` in the
  cwd. It is not a REPL. pi-ai does **not** own credential persistence — that is
  `CredentialStore`, injected by the app.
- Model data generation is transactional (`scripts/generate-models.ts:3383-3507`): stage into a
  temp dir, validate, then `rename` — and on failure the previous shards, aggregator, *and* data
  dir are restored. `test/generate-models-strict.test.ts` asserts exactly that.

## `packages/agent` — three runtimes

See [architecture.md](architecture.md#three-runtimes-in-packagesagent) for the map. This
section covers the core agent, which is the one the `pi` binary uses.

### `src/types.ts` (500 lines) — read this first

| Type | Line | Note |
|---|---|---|
| `StreamFn` | 33-37 | Must not throw. |
| `AgentTool` | 443-468 | Extends pi-ai's `Tool` with `label`, `prepareArguments?`, `execute(toolCallId, params, signal?, onUpdate?)`, `executionMode?`, and `replay?: "never" \| "safe"`. |
| `AgentState` | 378-417 | `systemPrompt` is a **read-only replay of the transcript** — you change it by appending a system message. `tools`/`messages` are accessors that copy on assignment. `pendingToolCalls` is a replaced `Set` per event, never mutated. |
| `AgentEvent` | 485-500 | 10 variants: `agent_start/end`, `turn_start/end`, `message_start/update/end`, `tool_execution_start/update/end`. |
| `AgentLoopConfig` | 189-338 | Requires `model` + `convertToLlm`; everything else optional. |
| `QueueMode` | 55 | `"all" \| "one-at-a-time"`. |

`AgentTool.replay` is **unused by `agent-loop.ts`** — grep confirms no reads. It exists for the
harness's `effect_pending` recovery policy (`packages/agent/docs/harness.md`).

### The turn loop (`src/agent-loop.ts`)

`runLoop` (162-320) is an outer loop for follow-ups wrapping an inner loop for tool batches and
steering. Per iteration:

1. `prepareNextTurn` if a turn just finished, then `turn_start`. The steering re-poll at `:203-205`
   is **conditional** on an empty queue so `one-at-a-time` mode never delivers two messages in
   one turn.
2. Emit pending messages.
3. `prepareRequest` — may replace context/model/thinking for this and all later requests.
   It may *not* append messages: `AgentRequestUpdate = Omit<AgentLoopTurnUpdate, "messages">`.
4. `streamAssistantResponse`.
5. Hard exit on `stopReason === "error" | "aborted"`.
6. `executeToolCalls`.
7. `finishTurn` → `turn_end`. `{action:"end"}` exits without polling queues.
8. Poll steering; then follow-ups; then an `explicitContinuation` fallback that forces exactly
   one context-only turn.

### Five loop behaviors that will surprise you

1. **`agent_end` ≠ idle.** It is the last emitted event, but the agent is not idle until all
   awaited `agent_end` listeners settle, so `isStreaming` stays true and `waitForIdle()` has not
   resolved (`types.ts:478-484`).
2. **Parallel tool mode splits ordering.** `tool_execution_end` fires in *completion* order;
   the toolResult messages are emitted afterwards in *assistant source* order
   (`agent-loop.ts:646-651`).
3. **`stopReason: "length"` fails every tool call in that message** instead of executing them
   (`:468-500`). Streamed tool args are finalized with a best-effort salvage parser, so truncated
   args can validate while being silently incomplete.
4. **`terminate` is unanimous-only** (`:685-687`): early termination requires *every* finalized
   call in the batch to set it. It is runtime-only and never reaches the emitted `toolResult`.
5. **Tool declaration drift is announced.** Before each request the loop diffs transcript-declared
   tools against `context.tools` and injects a `SystemMessage` with `toolsAdded`/`toolsRemoved`
   (`:332-362`).

Error runs are not special-cased: `handleRunFailure` (`agent.ts:532-548`) fabricates a synthetic
assistant message with `stopReason: "error" | "aborted"` and emits the normal
`message_start`/`message_end`/`turn_end`/`agent_end` tail.

`Agent` maintains its own `messages` array (pushing on `message_end`); the loop's `context.messages`
is a snapshot copy made at `agent.ts:460-465`. They diverge by design — only the `Agent` copy is
authoritative for the next run.

### `src/stream-fn.ts` — 20 lines

A one-slot mutable module-global default, deliberately (`stream-fn.ts:5-10`): hosts install a stream
function without making `pi-agent-core` depend on a provider catalog. `Agent` copies it into a
public mutable field, so the `?? getDefaultStreamFn()` fallback in the loop is only reachable for
direct `runAgentLoop` callers.

### `src/proxy.ts` — server-routed streaming

`streamProxy` is a `StreamFn` that POSTs to `${proxyUrl}/api/stream`. The server strips the
`partial` field from deltas to cut bandwidth; the client rebuilds it, re-parsing tool-call args
through `parseStreamingJson` on every delta. `buildProxyRequestOptions` is an explicit allow-list —
`apiKey` and `signal` are deliberately not serialized.

### The harness (durable runtime)

Normative spec: `packages/agent/docs/harness.md`. Read §0.9 first — it is the
implementation-status ledger (J1, C1, R11, R12, T1, S3, H1 are known-incomplete).

Four parts of a session: an immutable **entry tree**; mutable **values and lists** at bound typed
addresses; **branches and lanes** (a lane = a named data path with a movable tip, a config,
queues, and at most one operation); and an append-only **usage ledger**.

| Concept | File | Note |
|---|---|---|
| Public contract | `src/harness/agent-harness.ts` | `AgentHarness`, `AgentLane`, 11 hooks, `HarnessEvent`. |
| Single-writer guarantee | `src/harness/session/mutation-line.ts` | A 23-line promise-tail queue. |
| Bound addresses | `src/harness/session/values.ts` | `value<T>(ns, key)` / `list<T>(ns, key)`; list reads default to limit 1,000 and clamp at 10,000. |
| `OperationState` | `src/harness/session/types.ts:316-329` | **13 flat leaves** discriminated by `at`, not a class hierarchy. |
| Reducer | `src/harness/runtime/reducer.ts` | Pure in-place mutation of a caller-owned `LaneSnapshot`. `navigation_end` returns `"rebase"` because the reducer cannot derive a branch move. |
| Node IO | `src/harness/env/nodejs.ts` | 924 lines. errno→`FileErrorCode`, bounded output with adaptive spill to a temp file, `128 + signal` exit codes, process-tree kill. |
| Storage | `src/harness/session/jsonl/` | `JSONL_FORMAT_VERSION = 4`, with a v3 legacy importer. |

`docs/runtime-simplification.md:50` is the design constraint to internalize:
**"Do not introduce a generic Procedure interface, runner, scheduler, graph, callback plan, or
dependency facade."** The 13-leaf `OperationState` is a hand-written exhaustive switch, and
`drive.ts:102-104` throws if a procedure makes no progress.

### Pico3

`src/harness/pico3/`, exported **only** via the `./experimental/pico3` subpath. Agent behavior is
composed from replaceable **task kinds** (`generation`, `tool`, `postTools`, `collapse`, `job`,
`plugin`) rather than a hard-coded scheduler. User kinds may not shadow a built-in and `pi.*` is
reserved (`harness.ts:156-164`). Design lineage: `packages/agent/docs/pico-v3.md`.

### Agent tests

Three vitest configs: `vitest.config.ts` (everything), `vitest.harness.config.ts`
(`test/harness/**`, plus coverage over `src/agent.ts` and `src/agent-loop.ts`),
`vitest.benchmark.config.ts`. `test/e2e.test.ts` uses the faux provider, never a real API.
`test/harness/pico3/types.compile.ts` is compile-time only — deliberately not a `.test.ts`.

## Related

- [architecture.md](architecture.md) — package map and request lifecycle
- [development.md](development.md) — check/test/build commands
- `packages/ai/README.md` — provider list, options, auth
- `packages/agent/docs/harness.md` — the normative harness spec
- `.pi/skills/add-llm-provider.md` — provider addition checklist
