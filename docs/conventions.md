# Conventions

Everything here is enforced either by `npm run check` or by a test. Read this before your first
commit.

## Formatting and linting

Biome 2.3.5, not eslint/prettier (`biome.json`).

- **Tabs, indent width 3, line width 120.** `biome.json:22-24`.
- `noNonNullAssertion` is **off** — `!` is acceptable.
- `noExplicitAny` is **off** in biome, but `AGENTS.md` still bans `any` unless absolutely
  necessary. That one is a human rule.
- `useNodejsImportProtocol` off, `noEmptyInterface` off (needed for declaration merging —
  `packages/ai/src/types.ts` uses it for `CustomAgentMessages`).
- Never hand-edit `packages/ai/src/models.generated.ts` or `*.models.ts`; both are excluded from
  biome (`biome.json:37-38`) and generated. Regenerate via
  `npm run generate:models`, then include the diff.

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
`./foo.js` specifier in a non-declaration `.ts` file, including inside `import type`.

`erasableSyntaxOnly` is why you will see a `unique symbol` phantom-type trick to carry type
information with zero runtime cost — e.g. `packages/ai/src/types.ts:697-714`, where
`TranscriptContext` is branded so a raw `Context` cannot reach a provider by accident.

## Imports

- **No inline/dynamic imports** (`AGENTS.md`). Top-level imports only.
- The deliberate exceptions, all in `packages/ai`, exist to defeat bundlers or survive browser
  builds. Do not "fix" them:
  - `src/auth/oauth/load.ts` — variable-specifier `import()` so a bundler cannot follow it into
    `node:http`/`node:crypto`. `src/bun-oauth.ts` registers static loaders for single-file
    binaries.
  - `src/api/bedrock-converse-stream.lazy.ts` — variable specifier, to keep the AWS SDK out of
    browser bundles.
  - `src/env-api-keys.ts:1-24` — dynamic `node:fs`/`os`/`path` with a comment saying converting
    these to top-level imports breaks browser/Vite builds.
  - `src/compat.ts`, `src/legacy-api-aliases.ts` — a compatibility surface scheduled for removal.
- Extension code loads TypeScript at runtime via `jiti`
  (`packages/coding-agent/src/core/extensions/jiti-loader.ts`). That is extension loading, not
  source imports.

## Entry points are cost contracts

`scripts/check-entry-graphs.mjs:3-11` states the rule: *one stray `export *` can make a narrow
entry drag an entire barrel; importing a 1-file pure function through a barrel cost ~37 MB of
evaluated module graph, and nothing fails until someone measures a process.*

Budgets are declared in `check-entry-graphs.mjs:34-44`:

| Entry | Budget |
|---|---|
| `ai/utils/*` | ≤3 files, must not reach `providers/`, `api/`, or `index.ts` |
| `agent/harness/runtime/reducer` | ≤1 file |
| `agent/harness/context` | ≤6 files, must not reach `harness/runtime/`, `harness/execution/`, or `ai/` |
| `agent/harness/env/nodejs` | ≤5 files, must not reach `ai/` or `harness/runtime/` |
| `agent/harness/session` | ≤25 files, must not reach `harness/runtime/`, `harness/execution/`, or `ai/index.ts` |

`.` and `./node` are deliberately unbounded ("batteries-included"). The same thinking shows up in
`packages/ai/test/lazy-module-load.test.ts` and `scripts/check-browser-smoke.mjs`.

When you add a barrel export, run `npm run check` and see whether a budget broke.

## Errors: return, don't throw, at the boundaries

Two contracts you must not violate:

- **`StreamFn` must never throw or reject** (`packages/agent/src/types.ts:19-37`). Failures are
  in-band: a terminal `done`/`error` event carrying an `AssistantMessage` with
  `stopReason: "error" | "aborted"`. This is why `packages/agent/src/proxy.ts:232-243` turns a
  clean EOF with no terminal event into a synthetic `error` event instead of hanging forever.
- **`ExecutionEnv` / `FileSystem` methods must never throw**
  (`packages/agent/src/harness/types.ts:270-274`). Every filesystem failure, including unexpected
  backend errors, is encoded in a `Result`. `NodeExecutionEnv`
  (`packages/agent/src/harness/env/nodejs.ts:105-129`) maps Node errno codes onto eight stable
  `FileErrorCode` values.

Errors that *are* exceptions use tagged shapes from `packages/agent/src/harness/result.ts`
(`TaggedError` factory, `HarnessFault`, `HarnessClosed`, `matchError`) or `ModelsError` with
codes `model_source | model_validation | provider | stream | auth | oauth`
(`packages/ai/src/utils/models-error.ts`).

## Naming and structure

- `create*` for factories, `is*`/`has*`/`assert*` for predicates, `parse*` for parsing.
- `*Operations` interfaces are pluggable IO seams on tools, e.g.
  `packages/coding-agent/src/core/tools/read.ts` — `examples/extensions/ssh.ts` swaps them to run
  every tool over SSH.
- Tool files export **two** things: a rich `create*ToolDefinition` (name, typebox `parameters`,
  `promptSnippet`, `promptGuidelines`, `renderCall`, `renderResult`, `execute`) and a thin
  `create*Tool` returning a plain `AgentTool`. The CLI stores definitions and adapts them
  (`src/core/tools/tool-definition-wrapper.ts`).
- Inline single-line helpers that have only one call site (`AGENTS.md`).
- Never hardcode a key check like `matchesKey(keyData, "ctrl+x")`. Add the default to
  `TUI_KEYBINDINGS` (`packages/tui/src/keybindings.ts:71-210`) or the coding-agent's default
  tables so it stays configurable.

## Git

`AGENTS.md` — multiple pi sessions share this cwd. Stage explicit paths, never `git add -A`.
Never run `git reset --hard`, `git checkout .`, `git clean -fd`, `git stash`,
`git commit --no-verify`, or force-push; each destroys another session's work.

Message format: `{feat,fix,docs}[(ai,tui,agent,coding-agent)]: <summary>`. No emojis.
Never commit unless asked.

## Testing conventions

- Test names describe behavior. Regression tests encode the issue number
  (`packages/coding-agent/test/suite/regressions/`) or the bug
  (`packages/tui/test/regression-*.test.ts`).
- Add a comment with the GitHub issue number next to a regression test.
- `packages/agent/src/harness/session/testing/` exports runner-independent conformance cases
  (`ConformanceCase`, `StorageFixture`) so any backend can be validated against the same suite.
  `StorageFixture extends AsyncDisposable` — dispose with `Symbol.asyncDispose`.
- No network, no real API keys, no paid tokens in tests. Use the faux provider.
- If you add a provider, `packages/ai/README.md:1848-1865` lists the required cross-cutting test
  files; `.pi/skills/add-llm-provider.md` §5 restates it as a checklist.

## Comment style

The codebase uses precise "why" comments with citations to files, line numbers, issues, and
upstream bugs. Examples worth imitating:

- `packages/ai/src/utils/provider-retry.ts:97-104` — why every adapter must pass `maxRetries: 0`.
- `packages/tui/src/tui-main-screen.ts:344-351` — why Termux height changes skip the full redraw.
- `packages/coding-agent/src/core/extensions/loader.ts:712` — why extension discovery is one
  level deep.
- `scripts/check-entry-graphs.mjs:3-11` — why entry graphs are budgeted.

Do not write comments that restate the code.
