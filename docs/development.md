# Development

## Setup

Requires **Node >= 22.19.0** (`package.json:71-73`; CI uses Node 22). Node's native TypeScript
stripping is load-bearing: there is no compile step for running or testing from source.

```bash
npm install --ignore-scripts
```

Always pass `--ignore-scripts`. `.npmrc` sets `save-exact=true` and `min-release-age=2`.

Linux CI installs `libcairo2-dev`, `libpango1.0-dev`, `libjpeg-dev`, `libgif-dev`,
`librsvg2-dev`, plus managed `fd`/`ripgrep` (CI symlinks `fdfind` to `fd`)
(`.github/workflows/ci.yml:26-30`).

## Model data must be hydrated first

`packages/ai/src/providers/data/` is **gitignored** (`.gitignore:11`) and absent in a fresh
checkout. Without it, `packages/ai` tests and builds fail.

```bash
npm run hydrate:model-data   # fetch provider catalogs into src/providers/data/
npm run check:model-data     # validate the generated data
```

`npm run build` hydrates first. `npm run build:offline` does not; it runs `check:model-data`
and then builds from the existing data.

## Build

```bash
npm run build            # refresh model data, then build every package in order
npm run build:offline    # same, without network access
npm run build:native:darwin   # or :linux / :win32 — only needed for pi-tui native addons
```

`build` is a `cd`-chained sequence in `package.json:16`:

```
telemetry → chord → tui → codemode → mcp → ai → durable → env → agent → coding-agent
```

The order is the dependency graph. The coding agent's final step
(`scripts/build-coding-agent-bundle.mjs`) produces the `pi` binary at
`packages/coding-agent/dist/bundle/cli.js`. For a standalone executable, run
`npm --prefix packages/coding-agent run build:binary` (Bun compiles `dist/bun/cli.js`).

## Check — this is the gate

`npm run check` runs, in order (`package.json:21`):

| Step | What it enforces |
|---|---|
| `node scripts/biome-check.mjs --write` | Biome lint/format with `--error-on-warnings`; writes fixes. Falls back to the WASM build on Termux/unsupported platforms. |
| `check:pinned-deps` | Every registry dependency in every manifest is an exact version. Workspace deps are exact too; only `workspace:`, `file:`, `link:`, `portal:`, `git+`, `github:`, `git:`, `https:`, `ssh:`, `git://` are exempt. |
| `check:runtime-deps` | Every bare import in a package's build graph is declared in its manifest. Relative paths, `node:`, and `cloudflare:` are exempt. |
| `check:ts-imports` | No relative `./foo.js` specifiers in `.ts` files. Use `./foo.ts`. |
| `check:entry-graphs` | Each narrow public entry point stays under its declared file budget and forbidden paths. |
| `check:install-lock:coding-agent` | The generated install lock is current. |
| `tsc --noEmit` | Type check. Root `tsconfig.json` covers `packages/*/src`, `packages/*/test`, `packages/durable/test-workers`, the model-catalog scripts, and coding-agent/agent examples. |
| `check:browser-smoke` | Three esbuild browser bundles: durable excludes its Node adapters; the agent bundle excludes all providers except one catalog and one SDK (`@anthropic-ai/sdk`). |

`biome-check --write` **mutates your files**. Re-read the diff before committing.
`npm run check` does not run tests.

The pre-commit hook (`.husky/pre-commit`) runs `check-lockfile-commit`, then the full `check`,
then `check:browser-smoke` when staged files touch `packages/ai`, `packages/web-ui`, or the
root manifests, then re-stages staged files that still exist. Lockfile changes are blocked
unless `PI_ALLOW_LOCKFILE_CHANGE=1` (also `true`/`yes`).

## Test

```bash
./test.sh            # the full suite — use this
```

`test.sh` creates a marked temp root and refuses to delete it unless it owns the path. It sets
an isolated `HOME`/`TMPDIR`/`XDG_*`, points git/npm at empty configs, sets `GIT_ASKPASS` to
`false`, `PI_NO_LOCAL_LLM=1`, `AWS_EC2_METADATA_DISABLED=true`, and runs `env -i … npm test`.
This is what stops e2e tests from firing on real credentials.

Root `npm test` runs `test:scripts` (`scripts/*.test.mjs`, `*.test.ts`) and then
`npm run test --workspaces --if-present`.

Never run `npm test` at the root directly. E2E tests self-enable when auth or endpoint env
vars are present (for example `packages/ai/test/anthropic-long-cache-retention-e2e.test.ts`).

Target a single test while iterating:

```bash
# vitest packages (most of them) — run from the package root
node "$(git rev-parse --show-toplevel)/node_modules/vitest/dist/cli.js" --run test/foo.test.ts

# packages/tui uses node:test, not vitest
node --test test/foo.test.ts
```

Per-package runners:

| Package | Command |
|---|---|
| `tui` | `node --test --test-reporter=dot --test-reporter-destination=stdout test/*.test.ts` |
| `ai`, `agent`, `coding-agent`, `codemode`, `mcp`, `chord`, `telemetry`, `env`, `durable` | `vitest --run` |
| `durable` | runs the suite twice (with and without document retention) and then worker tests |
| `evals` | `vitest run --config vitest.test.config.ts` |

`packages/coding-agent` sets `PI_OFFLINE=1` in `vitest.config.ts:8-11`. Opt into network in a
test via `allowNetwork()` from `packages/coding-agent/test/test-network-env.ts`.

Root extras: `npm run test:mcp-conformance` runs the MCP conformance harness.

### Writing tests

- Import source directly with explicit extensions: `import { x } from "../src/foo.ts"`. Never
  the built package.
- `vitest.base.ts` aliases the shared workspace packages to their `src/*.ts`, so tests run
  against TypeScript source, never `dist`.
- `packages/coding-agent/test/suite/harness.ts` is the mandated suite harness
  (`createHarness()` starts at ~150; it constructs `Agent` and `AgentSession` directly, without
  `createAgentSession()`). Use the faux provider
  (`packages/ai/src/providers/faux.ts`); no real APIs, keys, network, or paid tokens.
- `packages/coding-agent/test/suite/README.md` also says not to extend the legacy
  `test/test-harness.ts` unless a missing capability forces it.
- Regression tests carry the issue number:
  `packages/coding-agent/test/suite/regressions/<issue#>-<slug>.test.ts`.
  `packages/tui/test/` uses descriptive `regression-*.test.ts` names.
- Storage and environment backends get shared conformance suites from
  `packages/durable/src/testing/` (`createStorageConformance`, `createEnvConformance`).

## Run pi from source

```bash
./pi-test.sh                     # from any directory
./pi-test.sh --no-env            # unset all provider credential env vars first
./pi-test.sh -- --model foo/bar  # extra args are forwarded
```

`pi-test.sh:58-59` runs `node --import <source-resolver> packages/coding-agent/src/cli.ts`.
`src/experimental/source-resolver.ts` reads `compilerOptions.paths` from the root
`tsconfig.json` and registers a `node:module` resolve hook so `@earendil-works/*` resolve to
source instead of stale `dist`. It has no `PI_EXPERIMENTAL` gate; that flag only controls
application-level UI features (`core/experimental.ts`). Windows variants: `pi-test.ps1`,
`pi-test.bat`.

To test interactive mode under tmux, follow `.pi/skills/interactive-testing.md`.

## Release

Full procedure: `.pi/skills/release.md`. Summary of the machinery:

- `npm run release:{patch,minor,major}` → `scripts/release.mjs`. Patch = fixes and additions;
  minor = breaking changes; there are no major releases.
- `npm run version:{patch,minor,major}` bumps every workspace, runs `scripts/sync-versions.js`,
  and refreshes the lockfile. All workspace packages stay in lockstep at `1.4.0`.
- `prepublishOnly` = clean + build + check. `npm run publish:dry` is the safe rehearsal.
- `npm run release:local` builds, packs, and creates isolated npm and Bun installs outside the
  repo before tagging.
- Generated artifacts: `npm run install-lock:coding-agent` (with `--check` wired into
  `npm run check`). Lifecycle-script exceptions are reviewed in
  `scripts/generate-coding-agent-install-lock.mjs`.
- The Nix build pins a model-catalog revision in `nix/model-catalog.json`; refresh it with
  `npm run update:model-catalog-pin` after provider or model changes.

## Adding an LLM provider

Follow `.pi/skills/add-llm-provider.md`. It covers core types, the provider factory, lazy
registration, `scripts/generate-models.ts`, the required test matrix (one `stream.test.ts` plus
nine cross-cutting files), coding-agent wiring, and docs.

Enforced by tests, not convention: `packages/ai/test/lazy-module-load.test.ts` spawns a process
and asserts that importing the root barrel, `providers/all`, or `compat` loads **zero** provider
SDKs, and that one stream loads exactly one.

## Changelog

Entries go under `## [Unreleased]` in `packages/*/CHANGELOG.md`, appended to the existing
`Breaking Changes` / `Added` / `Changed` / `Fixed` / `Removed` subsections (create a missing
subsection). Released version sections are immutable.

Conflict to be aware of: `AGENTS.md` documents this workflow for in-repo agents, while
`CONTRIBUTING.md:68` tells external contributors *"Do not edit CHANGELOG.md. Changelog entries
are added by maintainers."* Follow `CONTRIBUTING.md` for outside PRs.

## Removed commands

Do not look for these; they no longer exist: `check:shrinkwrap`, `shrinkwrap:coding-agent`,
root `build:binary`, `packages/agent` `test:harness` / `test:session:conformance`, and
`mini-test.sh`.
