# Development

## Setup

Requires **Node >= 22.19.0** (`package.json:66-68`; CI uses Node 22). Node's native TypeScript
stripping is load-bearing — there is no compile step for running or testing from source.

```bash
npm install --ignore-scripts
```

Always pass `--ignore-scripts`. `.npmrc` sets `save-exact=true` and `min-release-age=2`.

System packages needed on Linux (from `.github/workflows/ci.yml:26-30`): `libcairo2-dev`,
`libpango1.0-dev`, `libjpeg-dev`, `libgif-dev`, `librsvg2-dev`, plus `fd` and `ripgrep` (pi
downloads managed copies, but CI symlinks `fdfind` to `fd`).

## Model data must be hydrated first

`packages/ai/src/providers/data/` is **gitignored** (`.gitignore:11`) and absent in a fresh
checkout. Without it, `packages/ai` tests and `build:offline` fail.

```bash
npm run hydrate:model-data   # fetches provider catalogs into src/providers/data/
```

`npm run build` does this for you. `npm run build:offline` does not — use it only when the data
directory already exists.

## Build

```bash
npm run build            # refresh model data, then build every package in order
npm run build:offline    # same, without network access
npm run build:native:darwin   # or :linux / :win32 — only needed for pi-tui native addons
```

`build` is a `cd`-chained sequence in `package.json:16`; the order *is* the dependency graph.
The coding-agent's final step (`scripts/build-coding-agent-bundle.mjs`) produces the `pi` binary
at `packages/coding-agent/dist/bundle/cli.js`. `npm run build:binary` additionally compiles a
standalone executable with `bun build --compile`.

## Check — this is the gate

`npm run check` runs, in order (`package.json:21`):

| Step | What it enforces |
|---|---|
| `biome check --write --error-on-warnings .` | Lint + format. Tabs, width 3, line width 120. |
| `check:pinned-deps` | Every **external** dependency in every `package.json` is an exact version. Workspace deps use `^`. |
| `check:runtime-deps` | Every non-relative, non-`node:` import in `src/` is declared in that package's manifest. |
| `check:ts-imports` | No relative `./foo.js` specifiers in `.ts` files. Use `./foo.ts`. |
| `check:entry-graphs` | Each narrow package entry point stays under a declared module budget. |
| `check:shrinkwrap` | `packages/coding-agent/npm-shrinkwrap.json` matches the root lockfile. |
| `check:install-lock:coding-agent` | The generated install lock is current. |
| `tsc --noEmit` | Type check (root `tsconfig.json` covers `packages/*/src`, `packages/*/test`, `packages/coding-agent/examples`). |
| `check:browser-smoke` | esbuild browser bundles: `pi-ai` pulls zero SDKs, agent pulls only `@anthropic-ai/sdk`, `pi-durable` browser bundle excludes its Node adapters. |

`biome check --write` **mutates your files**. Re-read the diff before committing.

`npm run check` does not run tests.

The pre-commit hook (`.husky/pre-commit`) runs `check-lockfile-commit`, then the full `check`,
then `check:browser-smoke` if any staged file touches `packages/ai` or the root manifests, then
re-stages the files biome reformatted. Lockfile changes are blocked unless
`PI_ALLOW_LOCKFILE_CHANGE=1` (`scripts/check-lockfile-commit.mjs:4-6`).

## Test

```bash
./test.sh            # the full suite — use this
```

`test.sh` builds a temp `HOME`/`TMPDIR`/`XDG_*`, clears the environment down to `PATH`, `PWD`,
`TZ=UTC`, `GIT_TERMINAL_PROMPT=0` and friends, then runs `env -i … npm test`. This is what stops
e2e tests from firing on your real credentials.

Never run `npm test` at the root directly — `npm run test` fans out to
`npm run test --workspaces --if-present`, and e2e tests self-enable when auth/endpoint env vars
are present (e.g. `packages/ai/test/anthropic-long-cache-retention-e2e.test.ts:7,122`).

Target a single test while iterating:

```bash
# vitest packages (most of them) — run from the package root
node "$(git rev-parse --show-toplevel)/node_modules/vitest/dist/cli.js" --run test/foo.test.ts

# packages/tui uses node:test, not vitest
node --test test/foo.test.ts
```

Test layering inside a package:

| Command | Scope |
|---|---|
| `npm test` | everything, default vitest config |
| `agent: npm run test:harness` | `test/harness/**` only (`vitest.harness.config.ts:14`) |
| `agent: npm run test:session:conformance` | storage-backend conformance cases |
| `tui: npm test` | `node --test test/*.test.ts` |

`packages/coding-agent` also sets `PI_OFFLINE=1` in `vitest.config.ts:11-12`. Opt into network in
a test via `allowNetwork()` from `test/test-network-env.ts`.

### Writing tests

- Import source directly with explicit extensions: `import { x } from "../src/foo.ts"`. Never
  the built package.
- `vitest.base.ts` aliases every `@earendil-works/*` specifier to its `src/index.ts`, so tests
  always run against TypeScript source, never `dist`.
- `packages/coding-agent/test/suite/harness.ts` is the mandated harness. `test/suite/README.md`:
  use it, use the faux provider (`packages/ai/src/providers/faux.ts`), no real APIs, keys, network
  or paid tokens. `harness.ts:143-190` constructs `Agent` + `AgentSession` directly rather than
  going through `createAgentSession()`.
- Regression tests carry the issue number:
  `packages/coding-agent/test/suite/regressions/<issue#>-<slug>.test.ts`.
  `packages/tui/test/` uses descriptive names (`regression-sigwinch-kill-eacces.test.ts`).
- Fault injection for storage: `GatingStorage` (`arm` / `waitPending(n)` / `next(n)` / `discard`)
  and `InstrumentedStorage` reproduce "kill the process between any two commits"
  (`packages/agent/src/harness/session/testing/`).

## Run pi from source

```bash
./pi-test.sh                     # from any directory
./pi-test.sh --no-env            # unset all provider API keys first
./pi-test.sh -- --model foo/bar  # extra args are forwarded
```

`pi-test.sh:58-59` runs node with `--import` pointing at
`packages/coding-agent/src/experimental/source-resolver.ts`. That resolver reads
`compilerOptions.paths` from the root `tsconfig.json` and registers a `node:module` hook, so
`@earendil-works/*` resolve to source instead of a stale `dist`. It also gates everything behind
`PI_EXPERIMENTAL=1` (`src/core/experimental.ts:1-3`), which `src/experimental/cli.ts` sets.

Mini variants (see `src/experimental/{mini,micro}/README.md`): `mini-test.sh`, `pi-test.ps1`,
`pi-test.bat`.

To test interactive mode under tmux, follow `.pi/skills/interactive-testing.md`.

## Release

Full procedure: `.pi/skills/release.md`. Summary of the machinery:

- `npm run release:{patch,minor,major}` → `scripts/release.mjs`.
- `scripts/sync-versions.js` keeps all package versions in lockstep (`^0.87.1` internally).
- `prepublishOnly` = clean + build + check. `npm run publish:dry` is the safe rehearsal.
- `npm run release:local` builds, packs, and creates isolated npm and Bun installs outside the
  repo before tagging.
- Generated artifacts: `npm run shrinkwrap:coding-agent` and `npm run install-lock:coding-agent`
  (both have `--check` modes wired into `npm run check`).
  `scripts/generate-coding-agent-shrinkwrap.mjs` has an explicit allowlist for dependency
  lifecycle scripts; a new one fails the check until reviewed.

## Adding an LLM provider

Follow `.pi/skills/add-llm-provider.md` — a 7-step checklist covering `packages/ai/src/types.ts`,
the provider factory, lazy registration, `scripts/generate-models.ts`, the required test matrix
(`stream.test.ts` plus eight cross-cutting files), coding-agent wiring, and docs.

Enforced by tests, not convention: `packages/ai/test/lazy-module-load.test.ts` spawns a process and
asserts that importing the root barrel, `providers/all`, or `compat` loads **zero** provider
SDKs, and that one stream loads exactly one.

## Changelog

Entries go under `## [Unreleased]` in `packages/*/CHANGELOG.md`, appended to the existing
`Breaking Changes` / `Added` / `Changed` / `Fixed` / `Removed` subsections.

Conflict to be aware of: `AGENTS.md` documents this workflow for in-repo agents, while
`CONTRIBUTING.md:69` tells external contributors *"Do not edit CHANGELOG.md. Changelog entries are
added by maintainers."* Follow `CONTRIBUTING.md` for outside PRs. Released version sections are
immutable.
