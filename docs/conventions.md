# Conventions

Everything here is enforced by `npm run check`, a test, or `AGENTS.md`. Read this before your
first commit.

## Formatting and linting

Biome 2.3.5, not eslint/prettier (`biome.json`).

- **Tabs, indent width 3, line width 120** (`biome.json:22-24`).
- `noNonNullAssertion` is **off** — `!` is acceptable.
- `noExplicitAny` is **off** in Biome, but `AGENTS.md` still bans `any` unless absolutely
  necessary. That one is a human rule.
- `useNodejsImportProtocol` off, `noEmptyInterface` off (needed for declaration merging),
  `noControlCharactersInRegex` off.
- Never hand-edit `packages/ai/src/models.generated.ts` or `*.models.ts`; both are excluded from
  Biome (`biome.json:36-37`) and generated. Regenerate via `npm run generate:models` and include
  the diff.
- A GritQL plugin (`scripts/biome/model-type-comparison.grit`) rejects direct comparisons like
  `model.type === "chat"`; use `isModelType()` / `getModelType()`.
- `npm run check` runs `scripts/biome-check.mjs --write`, which uses the native Biome binary
  where available and the WASM build on Termux/other platforms.

## TypeScript

`tsconfig.base.json`:

| Option | Value | Consequence |
|---|---|---|
| `erasableSyntaxOnly` | `true` | **No** `enum`, `namespace`/`module`, parameter properties, `import =`, `export =`, or anything needing JS emit. |
| `verbatimModuleSyntax` | `true` | Type-only imports **must** say `import type` / `export type`. |
| `allowImportingTsExtensions` + `rewriteRelativeImportExtensions` | `true` | Write `./foo.ts`; tsc rewrites to `.js` on emit. |
| `target` / `lib` | `ES2024` | |
| `strict` | `true` | |

`check:ts-imports` (`scripts/check-ts-relative-imports.mjs`) fails the build on any relative
`./foo.js` specifier in a non-declaration `.ts` file, including inside `import()` type nodes.

`erasableSyntaxOnly` is why you will see a `unique symbol` phantom-type trick — for example
`TranscriptContext` in `packages/ai/src/types.ts:795-803`, branded so a raw `Context` cannot
reach a provider by accident.

## Imports

- **No inline/dynamic imports** (`AGENTS.md`). Top-level imports only.
- The deliberate exceptions live in dedicated modules; do not add new ones casually:
  - `packages/ai/src/auth/oauth/load.ts` — variable-specifier `import()` so a bundler cannot
    follow it into `node:http`/`node:crypto`.
  - `packages/ai/src/api/bedrock-converse-stream.lazy.ts` — variable specifier, to keep the AWS
    SDK out of browser bundles.
  - `packages/ai/src/env-api-keys.ts` — dynamic `node:fs`/`os`/`path` so browser/Vite builds stay
    working.
  - `packages/ai/src/providers/images/register-builtins.ts` — its own lazy loader instead of
    `lazyApi()`.
  - `packages/coding-agent/src/**/*.lazy.ts` — one-line loaders for optional MCP/codemode
    runtimes.
  - `packages/coding-agent/src/utils/photon.ts` — lazy optional Photon module.
  - `packages/chord/src/loader.ts` — cache-busting plugin import for hot reload.
- Extension code loads TypeScript at runtime via `jiti`
  (`packages/coding-agent/src/core/extensions/jiti-loader.ts`). That is extension loading, not a
  source import.

## Entry points are cost contracts

`scripts/check-entry-graphs.mjs:2-10` states the rule: *one stray `export *` can make a narrow
entry drag an entire barrel; importing a 1-file pure function through a barrel cost ~37 MB of
evaluated module graph, and nothing fails until someone measures a process.*

Budgets are declared in `scripts/check-entry-graphs.mjs:36-78`:

| Package | Entry | Budget |
|---|---|---|
| `ai` | `./models` | ≤20 files; must not reach `providers/`, `models.generated.ts`, `index.ts`, `utils/validation.ts`, or `utils/typebox-helpers.ts` |
| `ai` | `./utils/*` | ≤3 files; must not reach `providers/`, `api/`, or `index.ts` |
| `codemode` | `.` | ≤12 files; must not reach `cloudflare.ts` |
| `codemode` | `./portable` | ≤8 files; must not reach the Node host, worker, wasm loader, or root entry |
| `codemode` | `./cloudflare` | ≤9 files; same forbidden set |
| `durable` | `.` | ≤73 files; must not reach `ai/index.ts`, `ai/utils/typebox-helpers.ts`, or `durable/src/images/` |
| `durable` | `./tools` | ≤17 files; must not reach `durable/src/images/` |

Unbudgeted entries are deliberately unbounded ("batteries-included"). The check also fails when
a budget names an export the manifest does not declare. When you add a barrel export, run
`npm run check` and see whether a budget broke.

## Errors: return, don't throw, at the boundaries

Three contracts you must not violate:

- **`StreamFn` must never throw or reject** (`packages/agent/src/types.ts:19-37`). Failures are
  in-band: a terminal `done`/`error` event carrying an `AssistantMessage` with
  `stopReason: "error" | "aborted"`. This is why `packages/agent/src/proxy.ts` turns a clean EOF
  with no terminal event into a synthetic `error` event instead of hanging forever.
- **`ExecutionEnv` / `FileSystem` operations return `Result` and do not throw**
  (`packages/durable/src/env/index.ts:200-285`). `NodeExecutionEnv`
  (`packages/durable/src/env/node.ts:612`) maps errno onto eight stable `FileErrorCode` values:
  `aborted`, `not_found`, `permission_denied`, `not_directory`, `is_directory`, `invalid`,
  `not_supported`, `unknown`. `Shell.exec` returns `ExecutionErrorCode` values
  (`aborted`, `timeout`, `shell_unavailable`, `spawn_error`, `callback_error`, `unknown`).
- **Storage failures are fatal to an open Durable session**, while invalid reads fail only that
  read (`packages/durable/src/errors.ts`).

Errors that *are* exceptions use tagged shapes:

- Chord: `ChordError` with a `code` from `CHORD_ERROR_CODES` plus `forbidden()` / `invalid()`
  helpers (`packages/chord/src/errors.ts`).
- pi-ai: `ModelsError` with codes `model_source | model_validation | provider | stream | auth |
  oauth` (`packages/ai/src/utils/models-error.ts`).
- Durable: `SessionFailed`, `ReadOnlySession`, `ConversationBusy`, `ReadAfterWrite`,
  `StorageRequestError` (`packages/durable/src/errors.ts`).

## Naming and structure

- `create*` for factories, `is*`/`has*`/`assert*` for predicates, `parse*` for parsing.
- `*Operations` interfaces are pluggable IO seams on tools, e.g.
  `packages/coding-agent/src/core/tools/read.ts:48` (`ReadOperations`) and
  `bash.ts:75` (`BashOperations`). `examples/extensions/ssh.ts` swaps them to run every tool
  over SSH.
- Built-in tool files export **two** things: a rich `create*ToolDefinition` (name, label,
  description, typebox `parameters`, `promptSnippet`, `promptGuidelines`, `renderCall`,
  `renderResult`, `execute`) and a thin `create*Tool` returning a plain `AgentTool`. The CLI
  stores definitions and adapts them (`src/core/tools/tool-definition-wrapper.ts`).
- `ToolDefinition` also carries `outputSchema` / `structuredContent` for programmatic callers,
  `exposure` (`direct | model-only | codemode | deferred | hidden`), `namespace`,
  `defaultActive`, `prepareLoadout`, and `annotations` (`core/extensions/types.ts:579-668`).
- Inline single-line helpers that have only one call site (`AGENTS.md`).
- Never hardcode a key check like `matchesKey(keyData, "ctrl+x")`. Add the default to
  `TUI_KEYBINDINGS` (`packages/tui/src/keybindings.ts:71-210`) or to the coding agent's
  `KEYBINDINGS` table (`packages/coding-agent/src/core/keybindings.ts:180`) so it stays
  configurable.

## Git

`AGENTS.md` — multiple pi sessions share this cwd. Stage explicit paths, never `git add -A`.
Never run `git reset --hard`, `git checkout .`, `git clean -fd`, `git stash`,
`git commit --no-verify`, or force-push; each destroys another session's work.

Message format: `{feat,fix,docs}[(ai,tui,agent,coding-agent)]: <summary>`. No emojis.
Never commit unless asked.

## Testing conventions

- Test names describe behavior. Regression tests encode the issue number
  (`packages/coding-agent/test/suite/regressions/<issue#>-<slug>.test.ts`) or the bug
  (`packages/tui/test/regression-*.test.ts`).
- Add a comment with the GitHub issue number next to a regression test.
- `packages/durable/src/testing/` exports shared conformance suites (`createStorageConformance`,
  `createEnvConformance`, `registerStorageConformance`, `registerEnvConformance`) so every
  storage backend and execution environment is validated against the same cases.
- `packages/coding-agent/test/suite/README.md`: use `test/suite/harness.ts`, use the faux
  provider, no real APIs/keys/network/paid tokens, keep tests deterministic. Do not extend the
  legacy `test/test-harness.ts` unless a missing capability forces it.
- If you add a provider, `.pi/skills/add-llm-provider.md` §5 lists the required test matrix:
  one `stream.test.ts` plus nine cross-cutting files.

## Comment style

The codebase uses precise "why" comments with citations to files, issues, and upstream bugs.
Examples worth imitating:

- `scripts/check-entry-graphs.mjs:2-10` — why entry graphs are budgeted.
- `packages/ai/src/utils/provider-retry.ts` — why SDK-backed adapters pass `maxRetries: 0`
  and wrap the request.
- `packages/tui/src/tui-main-screen.ts` — why Termux height changes skip the full redraw.
- `packages/coding-agent/src/core/extensions/loader.ts` — why extension discovery is one level
  deep.

Do not write comments that restate the code.
