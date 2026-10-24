# Agent Runtime: `packages/ai` and `packages/agent`

## `packages/ai` — unified multi-provider LLM API

### Layering

```
providers/<id>.ts          factory: id, name, baseUrl, auth, models, api/images/classifiers
  └─ api/<api>.lazy.ts     lazy wrapper: () => lazyApi(() => import("./<api>.ts"))
       └─ api/<api>.ts     wire protocol: stream() + streamSimple(), or generateImages(), or classify()
            └─ vendor SDK (or hand-rolled HTTP)

providers/data/<id>.json   generated catalog, gitignored
  └─ providers/<id>.models.ts   generated shard with chat, image, and classifier catalogs
       └─ models.generated.ts   generated aggregator (committed)
```

Each shard exports `*_MODELS`, `*_IMAGE_MODELS`, and `*_CLASSIFIER_MODELS`; the generator writes
all three. `src/model-catalog.ts` flattens them into typed catalogs.

### The two load-time invariants

**1. The root barrel pulls zero SDKs.** `src/index.ts:4-8` is explicit: no generated catalogs, no
provider implementations, no api registry, no OAuth implementations, no `compat`. It does export
the faux provider (`index.ts:34`), which is a provider factory but has no SDK dependency.
`test/lazy-module-load.test.ts` spawns a process, registers module resolve hooks for
`@anthropic-ai/sdk`, `openai`, `@google/genai`, and `@aws-sdk/client-bedrock-runtime`, and asserts
none load on root import, `providers/all`, or `compat`. It also asserts that one Anthropic stream
loads exactly `@anthropic-ai/sdk`.

**2. `stream()` is synchronous; setup is asynchronous.** `src/api/lazy.ts` is the keystone:
`lazyStream()` returns the stream now and pushes any setup failure as an `error` event. That is
what lets `Models.stream()` be sync while auth resolution is async. `Models.streamSimple()` is
also sync; the async part runs inside the returned stream.

### Core types (`src/types.ts`, 915 lines)

| Type | Line | Note |
|---|---|---|
| `KnownApi` | 64-74 | 10 chat APIs: OpenAI completions/responses, Azure responses, Codex responses, Anthropic messages, Bedrock converse, Google generative AI, Google Vertex, Mistral conversations, pi-messages. |
| `KnownImageApi` / `KnownClassifierApi` | 78-86 | `openrouter-images`; classifiers: `typesafe-system-one`, `cloudflare-workers-ai-system-one`, `llama-cpp-classify`, `openai-decisions`. |
| `Api` | 76 | `KnownApi \| (string & {})` — the `& {}` keeps autocomplete while allowing custom strings. |
| `Context` | 786-790 | Caller input: `{ systemPrompt?, messages, tools? }`. |
| `TranscriptContext` | 795-803 | `{ messages }` branded with a `unique symbol`. Only `normalizeContext()` produces it, so a raw `Context` cannot reach a provider. |
| `SystemMessage` | 561-578 | `sections`, `toolsAdded`, `toolsRemoved`. The leading system message **is** the system prompt and tool set; later ones patch it. |
| `AssistantMessage` | 585-621 | `content`, `usage`, `stopReason`, diagnostics, deferred handle, `durationMs`. |
| `AssistantMessageEvent` | 821-836 | `start`, text/thinking/tool-call events, `done`, `error`. Every non-terminal event carries `partial`, a shared live accumulator. |
| `StreamOptions` | 220-266 | Extends `ProviderRequestOptions`; adds `temperature`, `samplingParams`, `maxTokens`, `transport`, `cacheRetention`, `sessionId`, `onProviderStreamEvent`, `metadata`, … |
| `Model<TApi>` | 854-887 | `compat` is a conditional type keyed on the api id. `type?: "chat"` is optional; use `isModelType()` / `getModelType()`. |
| `ImageModel` / `ClassifierModel` | 890-904 | Both extend `BaseModel`; `type` is required here. |
| `Usage` | 459-487 | `reasoning` is a subset of `output`; `cacheWrite1h` is a subset of `cacheWrite`; `cost.totalFromProvider` marks provider-billed totals. |

`ProviderStreams` (`types.ts:319-331`) has `stream` and `streamSimple`, plus optional
`fetchDeferred` / `cancelDeferred`. Parallel contracts exist for images (`ProviderImages`,
`generateImages`) and classifiers (`ProviderClassifier`, `classify`).

### Providers are plain objects

There is no base class. `createProvider()` (`models.ts:1077`) accepts:

- one `ProviderStreams` or a `Partial<Record<TApi, ProviderStreams>>` map,
- an `images` map keyed by image API,
- a `classifiers` map keyed by classifier API,
- static `models`, optional `fetchModels`, `filterModels`, `filterAllModels`, and `auth`.

At least one implementation is required. Dispatch is on `model.api`; a model whose API has no
implementation produces an error stream.

`providers/all.ts` builds **42** provider factories. `radius.ts` is the one hand-written
`Provider` object because it has custom refresh logic. Most providers differ through `auth` and
the compat flags in `providers/compat-schema.ts`: `OpenAICompletionsCompat` alone has 27 flags,
including 11 `thinkingFormat` encodings.

### Auth resolution (`src/auth/resolve.ts:34-93`)

1. `overrides.apiKey` wins outright.
2. **A stored credential owns its provider.** If one exists, only that path is taken — no env
   fallback, not even after a failed refresh.
3. Only when nothing is stored does `apiKey.resolve()` consult ambient sources (env vars, AWS
   profiles, ADC files).

OAuth refresh is **double-checked locked** (`resolve.ts:118-160`): a token with less than five
minutes of validity triggers `CredentialStore.modify()`, which re-checks expiry under the lock so
concurrent processes refresh once. Once a refresh starts it ignores cancellation (bounded by a
15-second timeout), because the provider may already have rotated the refresh token.
`CredentialStore.modify` is the only write path (`auth/types.ts:86-90`).

### Two independent retry layers

| Layer | File | Scope |
|---|---|---|
| HTTP | `src/utils/provider-retry.ts` | One request inside the adapter. Honors `x-should-retry`, else retries 408/409/429/≥500. SDK-backed adapters pass `maxRetries: 0` and wrap the call. A server delay above `maxRetryDelayMs` (default 60 s) fails immediately instead of sleeping. |
| Turn | `src/utils/retry.ts` | Whole assistant turns via `retryAssistantCall`. Classification is string-pattern based on `errorMessage`; non-retryable quota/billing patterns are checked before retryable overload/network patterns. Aborts are terminal. |

### Context overflow

`src/utils/overflow.ts` holds 26 provider-specific overflow patterns, three non-match exclusions,
and three non-error signals:

- z.ai silent overflow (`usage.input + cacheRead > contextWindow`),
- Xiaomi MiMo `length` with zero output and >99% context fill,
- Cerebras bodyless 400/413.

`isContextOverflow()` is how compaction gets triggered.

### Images, classifiers, and the model store

- `Models.generateImages()` and `Models.classify()` resolve auth like `stream()` and never reject;
  failures come back as error-shaped results (`models.ts:990-1030`).
- `ImagesContext`, `AssistantImages`, `ClassifierContext`, `ClassifierResult` live in
  `types.ts:664-741`.
- `Providers` may expose `fetchModels()`; `Models.refresh()` refreshes configured dynamic
  providers concurrently and returns per-provider results instead of rejecting.
- `src/models-store.ts` persists catalogs (`models`, `lastModified`, `checkedAt`, `etag`). pi-ai
  owns catalog persistence, not credential persistence.

### Other things worth knowing

- `src/providers/faux.ts` is the in-repo test provider and part of the public API. It simulates
  streaming, deferred responses, token pacing, and prompt caching by diffing the common prefix of
  consecutive prompts; cache accounting is keyed by `sessionId` and disabled by
  `cacheRetention: "none"`.
- Prompt caching has four mechanisms selected by compat flags: `cache_control` breakpoints
  (Anthropic), OpenAI `prompt_cache_key` + `prompt_cache_retention`, Bedrock `cachePoint`, and
  session-affinity headers. `cacheRetention` is resolved per adapter.
- `calculateCost()` (`models.ts:1236-1260`) mutates `usage.cost` in place, applies `cost.tiers[]`,
  and charges Anthropic 1h cache writes at 2× the input rate.
- `src/compat.ts` preserves the old global API: API-dispatch `stream`/`complete`, env API keys,
  the API registry, generated catalog reads, image generation, and faux registration. It is
  scheduled for deletion with the coding-agent ModelManager migration. `legacy-api-aliases.ts`
  holds 16 deprecated `stream*` constants for eight APIs (not Bedrock or pi-messages).
- `src/cli.ts` (the `pi-ai` bin) is a standalone OAuth login tool that writes `./auth.json` in the
  current directory. It is not a REPL. pi-ai does not own credential persistence; the host injects
  a `CredentialStore`.
- `src/telemetry.ts` defines `AI_TELEMETRY_SCHEMA` and emits request spans/metrics. Exported
  telemetry attributes contain IDs and control metadata, never messages or tool arguments.
- `src/utils/identity.ts` defines pi's provider attribution (`PI_ATTRIBUTION`) and per-provider
  header mapping.
- Model generation is transactional (`scripts/generate-models.ts:3474-3590`): stage into a temp
  directory, validate, rename, and restore the previous shards, aggregator, and data directory on
  failure. `test/generate-models-strict.test.ts` covers the failure path.

## `packages/agent` — the core agent

`packages/agent` now contains only the core runtime: `agent.ts`, `agent-loop.ts`, `proxy.ts`,
`stream-fn.ts`, and `types.ts`. The durable harness lives in `packages/durable`; see
[infrastructure.md](infrastructure.md#durable--earendil-workspi-durable).

### `src/types.ts` (539 lines) — read this first

| Type | Line | Note |
|---|---|---|
| `StreamFn` | 33-37 | Must not throw or reject. |
| `QueueMode` | 55 | `"all" \| "one-at-a-time"`. |
| `AgentLoopConfig` | 192-343 | Requires `model` + `convertToLlm`; everything else optional. |
| `AgentState` | 382-422 | `systemPrompt` is a read-only replay of transcript system messages. `tools`/`messages` copy on assignment. `pendingToolCalls` is replaced per event. |
| `AgentToolResult` | 424-447 | `content`, `details`, optional `structuredContent`, `usage`, `isError`, `terminate`. |
| `AgentTool` | 466-499 | Extends pi-ai's `Tool` with `label`, `outputSchema`, `prepareArguments`, `execute`, `executionMode`, `replay`. |
| `AgentEvent` | 516-540 | 10 variants: agent/turn/message/tool-execution lifecycle. |

`AgentTool.replay` is unused by `agent-loop.ts`; the durable harness uses it for recovery.

### The turn loop (`src/agent-loop.ts`)

`runLoop` (`171-330`) is an outer loop for follow-ups wrapping an inner loop for tool batches and
steering. Per iteration:

1. `prepareNextTurn` if a turn just finished, then `turn_start`. The steering re-poll at `:206-209`
   is conditional on an empty queue, so `one-at-a-time` mode never delivers two messages in one
   turn.
2. Emit pending messages, announcing tool loadout changes.
3. `prepareRequest` — may replace context/model/thinking, but may not append messages
   (`AgentRequestUpdate = Omit<AgentLoopTurnUpdate, "messages">`).
4. `streamAssistantResponse`.
5. Hard exit on `stopReason === "error" | "aborted"`, after `finishTurn` and `turn_end`.
6. `executeToolCalls`.
7. `finishTurn` → `turn_end`. `{action:"end"}` exits without polling queues.
8. Poll steering; then follow-ups; then an `explicitContinuation` fallback that forces exactly one
   context-only turn.

### Five loop behaviors that will surprise you

1. **`agent_end` ≠ idle.** It is the last emitted event, but the agent is not idle until all
   awaited `agent_end` listeners settle, so `isStreaming` stays true and `waitForIdle()` has not
   resolved (`types.ts:410-415`).
2. **Parallel tool mode splits ordering.** `tool_execution_end` fires in *completion* order; the
   toolResult messages are emitted afterwards in *assistant source* order
   (`agent-loop.ts:596-691`).
3. **`stopReason: "length"` fails every tool call in that message** instead of executing them
   (`agent-loop.ts:488-516`). Streamed tool args are finalized with a best-effort salvage parser,
   so truncated args can validate while being silently incomplete.
4. **`terminate` is unanimous-only** (`agent-loop.ts:700-702`): early termination requires every
   finalized call in the batch to set it. It is runtime-only and never reaches the emitted
   `toolResult`.
5. **Tool declaration drift is announced.** Before each request the loop diffs transcript-declared
   tools against `context.tools` and injects a `SystemMessage` with `toolsAdded`/`toolsRemoved`
   (`agent-loop.ts:342-388`).

Error runs are not special-cased: `handleRunFailure` (`agent.ts:542-558`) fabricates a synthetic
assistant message with `stopReason: "error" | "aborted"` and emits the normal
`message_start`/`message_end`/`turn_end`/`agent_end` tail.

`Agent` maintains its own `messages` array (pushing on `message_end`); the loop's
`context.messages` is a snapshot copy made by `createContextSnapshot()` (`agent.ts:469-474`).
They diverge by design — only the `Agent` copy is authoritative for the next run.

### `src/stream-fn.ts` — 20 lines

A one-slot mutable module-global default. Hosts install a stream function without making
`pi-agent-core` depend on a provider catalog. `Agent` copies it into a public mutable field, so
the `?? getDefaultStreamFn()` fallback is only reachable for direct `runAgentLoop` callers.

### `src/proxy.ts` — server-routed streaming

`streamProxy` is a `StreamFn` that POSTs to `${proxyUrl}/api/stream`. The server strips the
`partial` field from deltas to cut bandwidth; the client rebuilds it, re-parsing tool-call
arguments through `parseStreamingJson` on every delta. `buildProxyRequestOptions` is an explicit
allow-list; `apiKey` and `signal` are deliberately not serialized.

### Agent tests

`packages/agent` has one vitest config (`vitest.config.ts`) and tests covering the agent loop,
steering/follow-up queues, tools, events, proxy, and the faux provider. There is no separate
harness config.

## Related

- [architecture.md](architecture.md) — package map and request lifecycle
- [development.md](development.md) — check/test/build commands
- [infrastructure.md](infrastructure.md) — Durable harness, Chord, telemetry
- `packages/ai/README.md` — provider list, options, auth
- `.pi/skills/add-llm-provider.md` — provider addition checklist
