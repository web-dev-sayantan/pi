# `packages/coding-agent` — the `pi` CLI

`@earendil-works/pi-coding-agent`, `bin.pi = dist/bundle/cli.js`. Builds on
`pi-agent-core`'s core agent, `pi-ai`, `pi-tui`, and `chord`.

## Bootstrap

`src/cli.ts` is 6 lines and matters:

```ts
setupCli();
main(process.argv.slice(2));
```

`setupCli()` (`src/cli/setup.ts:4-13`) runs **before any module does IO**: sets `process.title`,
`PI_CODING_AGENT=true`, `AI_AGENT=pi`, silences `process.emitWarning` entirely, and calls
`configureHttpDispatcher()` so undici is configured before any provider SDK issues a request.

`src/rpc-entry.ts` duplicates that by hand because it must inject `--mode rpc` into argv.
`src/bun/cli.ts` adds sandbox + Bun runtime side effects for the compiled binary.

`src/main.ts` (985 lines) is the single entry for all four modes, in this order: offline flag →
`pi auth` early-exit → self-update cleanup → `SettingsManager` bootstrap → package commands →
`parseArgs` → `--version` → `--export` → **`resolveAppMode()`** → `takeOverStdout` (non-interactive)
→ `runMigrations` → first-time setup wizard → session dir resolution → `createRuntime` factory →
`createAgentSessionRuntime` → `--help`/`--list-models` → piped-stdin read → theme init → mode dispatch.

Argument parsing is hand-written (`src/cli/args.ts:71-259`, a single `for` loop with an
`else if` chain — no commander/yargs). Unknown `--flags` land in
`result.unknownFlags` and are resolved later against extension-registered flags
(`src/core/agent-session-services.ts:82-128`).

## Four modes

`resolveAppMode` (`src/main.ts:111-122`): explicit `--mode` wins; otherwise any non-TTY on
stdin/stdout ⇒ `print`; anything TTY ⇒ `interactive`. A **second** downgrade happens later at
`:881-883` — piped stdin content flips interactive to print.

| Mode | File | Behavior |
|---|---|---|
| `print` | `modes/print-mode.ts` | Single-shot; prints only the final assistant text. Exit 1 on `stopReason: "error" \| "aborted"`. SIGTERM/SIGHUP handlers kill tracked detached children then exit 143/129. |
| `json` | same file, `mode:"json"` | One session-header line, then one `toJsonEvent(event)` per line. Uses `modes/json-event.ts` to strip the cumulative snapshot: `message_start` gives the initial message, `message_update` carries deltas, `message_end` is authoritative. |
| `rpc` | `modes/rpc/rpc-mode.ts` | `Promise<never>`. JSONL commands on stdin, responses/events on stdout. `RpcCommand` union at `modes/rpc/rpc-types.ts:20+`. |
| `interactive` | `modes/interactive/interactive-mode.ts` | The TUI (4000+ lines). |

## Three-layer session wiring

| File | Role |
|---|---|
| `src/core/sdk.ts` (454) | `createAgentSession(options)` — builds the `pi-agent-core` `Agent`, resolves model/thinking/tools, constructs `AgentSession`. The `Agent` is created at `sdk.ts:380-415`. |
| `src/core/agent-session-services.ts` (221) | `createAgentSessionServices()` — cwd-bound `ModelRuntime` + `SettingsManager` + `DefaultResourceLoader`; applies extension-pending providers and flags. |
| `src/core/agent-session-runtime.ts` | Owns the current session + services and rebuilds them on cwd change / new session / fork / switch. |

**The non-obvious core idea is deferred cwd resolution.** `--session` and `--resume` may select a
session from a *different* project, so project-local settings, resources, provider registrations
and models must not be resolved until the target session cwd is known (`main.ts:670-680`).
Hence `createRuntime: CreateAgentSessionRuntimeFactory` (`main.ts:717-847`) is a closure that can
rebuild everything for a new cwd. When `--session <id>` resolves to another project, interactive
mode prompts *"Fork this session into current directory?"* (`main.ts:398-406`).

`AgentSession` (`src/core/agent-session.ts`) is the class **shared by all four modes** — modes add
only their own I/O layer. `AgentSessionEvent` (`:164-205`) extends `AgentEvent` with session-only
events: `agent_settled`, `queue_update`, `compaction_start/end`, `entry_appended`,
`auto_retry_start/end`, `bash_execution_update`, etc.

## Tools

`src/core/tools/` exports 8 built-in tools: `read, bash, powershell, edit, write, grep, find, ls`
(`tools/index.ts:95-105`). Three factory groups: `createCodingToolDefinitions` (read/bash/edit/write),
`createReadOnlyToolDefinitions` (read/grep/find/ls), `createAllToolDefinitions`.

Each tool file exports **two** things: a rich `create*ToolDefinition` (name, description, typebox
`parameters`, `promptSnippet`, `promptGuidelines`, `renderCall`, `renderResult`, `execute`) and a
thin `create*Tool` returning a plain pi-agent-core `AgentTool`. The registry stores
**definitions**; `tools/tool-definition-wrapper.ts:5-28` adapts them to `AgentTool`. A test can
inject a plain `Record<string, AgentTool>` via `baseToolsOverride` and it is synthesized back into
a `ToolDefinition` (`:36-47`).

`*Operations` interfaces are pluggable IO seams — `examples/extensions/ssh.ts` swaps them to run
every tool over SSH.

`tools/renderers/` is a deliberate bundle split (`renderers/index.ts:1-8`): it re-exports only
`renderCall`/`renderResult` so a rendering-only process does not load typebox (~17 MB of module
graph).

Registration is `AgentSession._buildRuntime` (`agent-session.ts:3237-3289`). Default active tools
are `["read", "bash", "edit", "write"]` (`:3281-3285`).

## Hooks → extensions

**There is no `src/core/hooks/` directory.** Hooks were renamed to extensions; the only trace is
the deprecation warning at `src/migrations.ts:222-252`. The `hooks` subpath in `package.json:40`
is vestigial.

The replacement is `src/core/extensions/` — 8 files, the engine — plus `src/extensions/`, the
one shipped built-in.

| File | Role |
|---|---|
| `core/extensions/types.ts` (~1900) | `ExtensionAPI`, `ExtensionContext`, `ToolDefinition`, every event interface, type guards like `defineTool`. |
| `core/extensions/loader.ts` (799) | jiti-based TS module loading, discovery, transactional init. |
| `core/extensions/runner.ts` | `ExtensionRunner` — dispatches events, binds core actions. |
| `core/extensions/jiti-loader.ts` | 3 lines: `export { createJiti } from "jiti"`. |

`ExtensionEvent` is 46 event interfaces unioned at `types.ts:1185-1216`, exposed through ~40 typed
`pi.on(...)` overloads. Groups: trust/resources, session lifecycle, context/system prompt,
provider (`before_provider_request`, `before_provider_headers`, `after_provider_response`,
`provider_stream_event`, `cache_warming_decision`), agent loop (`agent_start/end`,
`agent_before_settle`, `agent_settled`, `turn_start/end`), UI, and message/tool events.

**Discovery** (`loader.ts:751-799`), in order: `cwd/.pi/extensions/`, `<agentDir>/extensions/`,
then explicit `-e` paths. Directory scanning is **one level deep** — a subdirectory resolves via
`package.json` with a `pi.extensions` manifest, else `index.ts`/`index.js`
(`loader.ts:672-702`, comment at `:712`).

**Loading** picks one of three jiti strategies (`loader.ts:491-495`): virtual modules for a
Bun binary/SEA, `tsconfigPaths` when running `.ts` sources, or a concrete alias map for
unbundled dist. `getAliases()` (`:65-121`) maps `@earendil-works/*` (and legacy
`@mariozechner/*`) to entry files — and notably maps bare `@earendil-works/pi-ai` to the
**`compat` entrypoint**, so extensions written against the old global API keep working.

**Extension load is transactional.** `createExtensionAPI` returns `{api, commit, discard}`
(`loader.ts:228-469`): flag defaults and provider registrations made during the factory call are
queued and only applied on `commit()`. On throw, `discard()` unsubscribes everything and the API
is permanently dead (`:462-467`).

Every `ExtensionAPI` method calls `assertActive()` first. After `ctx.newSession()`,
`ctx.fork()`, `ctx.switchSession()` or `ctx.reload()`, a captured ctx is **stale** — the loader
invalidate path (`loader.ts:185-203`) tears down the event subscriptions and marks it, because
`ExtensionCommandContext` actions like `newSession` are only safe in user-initiated commands.

## System prompt assembly

`src/core/system-prompt.ts` builds the prompt as a **map of named sections, not a flat string**:

```ts
const sections: SystemPromptSections = { preamble: promptSections.preamble };
for (const [name, content] of Object.entries(promptSections)) {
  if (name !== "preamble") sections[name] = `<${name}>\n${content}\n</${name}>`;
}
```

Order: `preamble` → `tools` → `rules` → `docs` → (`addendum`) → (`project_context`) → (`skills`)
→ `cwd` → custom sections (`:142-179`). A `customPrompt` short-circuits everything but
`preamble`/`addendum`.

Two consequences worth internalizing:

- `diffSystemPromptSections` (`:204-216`) produces a sparse patch, and
  `AgentSession._preparePromptAndToolLoadout` (`agent-session.ts:1410-1424`) emits a
  `SystemMessage` **only when the prompt actually changed**. An unchanged prompt adds nothing to
  the transcript.
- `_rebuildSystemPrompt` (`:1374-1398`) gathers snippets from the live tool registry, so
  **enabling a tool changes the system prompt**.

Context files (`AGENTS.md` / `CLAUDE.md`) load in `src/core/resource-loader.ts:127-213`, trying
`["AGENTS.override.md", "AGENTS.md", "AGENTS.MD", "CLAUDE.md", "CLAUDE.MD"]` and walking cwd →
root. `findShadowedContextFile` (`:157-173`) prevents a main-repo `AGENTS.md` being applied twice
in nested git worktrees.

Skills are `src/core/skills.ts` (one flat file, not a directory). It honors `.gitignore` /
`.ignore` / `.fdignore` at every ancestor level via the `ignore` package.

## Assets and config

`src/config.ts` (579 lines) is the single source of truth for asset paths, and `AGENTS.md`
requires using its helpers instead of `__dirname`. Every accessor is a **function**, because the
right answer differs per distribution:

| Helper | Bun binary | Node (source or dist) |
|---|---|---|
| `getPackageDir()` | `dirname(process.execPath)` | `findNodePackageDir(__dirname)`, `PI_PACKAGE_DIR` override |
| `getThemesDir()` | `<pkg>/theme` | `<pkg>/{src\|dist}/modes/interactive/theme` |
| `getExportTemplateDir()` | `<pkg>/export-html` | `<pkg>/{src\|dist}/core/export-html` |
| `getInteractiveAssetsDir()` | `<pkg>/assets` | `<pkg>/{src\|dist}/modes/interactive/assets` |

The `{src\|dist}` choice is `existsSync(join(packageDir, "src")) ? "src" : "dist"`. `findNodePackageDir`
has a Bun-specific fix: `build:binary` puts Bun's metadata inside `dist/`, so a dir literally
named `dist` with a `package.json` one level up resolves to the parent.

`config.ts` also reads the `piConfig` block from `package.json` for `PACKAGE_NAME`, `APP_NAME`
(default `pi`), `APP_TITLE` (default `π`), `CONFIG_DIR_NAME` (default `.pi`), `VERSION`, and
derives env var names from `APP_NAME` — so a fork that changes `piConfig.name` gets its own
namespace.

## Interactive mode

`src/modes/interactive/interactive-mode.ts` is the mode class;
`src/modes/interactive/tui-renderer.ts` is the composition root that decides which pi-tui renderer
to build — `TuiMainScreen` (`tuiMode: "regular"`) or `TuiAltScreen` (`tuiMode: "fullscreen"`).

One non-obvious trick: `createInteractiveTuiReference` (`tui-renderer.ts:51-79`) returns a
**`Proxy` that survives renderer replacement**, so long-lived components can hold one stable `TUI`
reference while `switchTuiMode()` swaps the underlying renderer. Components are created once and
**remounted** rather than rebuilt (`:919`).

The mounted component tree is fixed (`interactive-mode.ts:935-943`): `documentContainer`,
`pendingMessagesContainer`, `statusContainer`, `widgetContainerAbove`, `editorContainer`,
`widgetContainerBelow`, `footerContainer`, plus `headerContainer` and a `TuiLayouts.ScrollView`
transcript.

Startup ordering has one real constraint: `ui.start()` runs **before** `rebindCurrentSession()` so
`session_start` handlers can use interactive dialogs (`:950`).

`components/` has 42 files (selectors, transcript renderers, editors, footers, plus demos like
`doom`-style overlays). `theme/` defines ~49 foreground `ThemeColor` tokens and ships `dark.json`
and `light.json`. Because pi renders **user-authored** themes, `setThemeJsonValidator(validateThemeJson)`
is called before any theme loads (`main.ts:890`) — see the comment at `theme/theme.ts:44-48`.

## Migrations

`src/migrations.ts` runs `runMigrations(cwd)` once per startup (`main.ts:653`).

| Function | Line | What |
|---|---|---|
| `migrateAuthToAuthJson` | 21-73 | Merges legacy `oauth.json` + `settings.json.apiKeys` into `auth.json` (mode 0600). No-op if it exists. |
| `migrateSessionsFromAgentRoot` | 84-131 | v0.30.0 bug: sessions went to `~/.pi/agent/*.jsonl` instead of `~/.pi/agent/sessions/<encoded-cwd>/`. |
| `migrateToolsToBin` | 177-216 | Moves managed `fd`/`rg` from `agentDir/tools/` to `agentDir/bin/`. |
| `migrateKeybindingsConfigFile` | 157-172 | Rewrites `keybindings.json` through `migrateKeybindingsConfig()`. |
| `migrateExtensionSystem` | 257-272 | `commands/` → `prompts/`; warns about leftover `hooks/` and non-binary `tools/`. |

## Other directories

| Path | What |
|---|---|
| `src/experimental/` | Gated on `PI_EXPERIMENTAL=1`. Four separate things: `micro/` (single-process Pico3), `mini/` (3-process tui/server/worker over unix socket), `services/` (Chord facet slices), and `plugins/`. Excluded from npm and the standalone binary. |
| `src/client/index.ts` | Literally `export * from "@earendil-works/pi-client"`. A re-export shim; excluded from npm via `"!dist/client"`. |
| `src/bun/` | Compiled-binary support: sandbox env setup/restore, Bun runtime shims. |
| `src/utils/` (39 files) | Paths (WSL, Git-Bash), child processes, images (photon WASM resize worker), clipboard, markup, git, zip. |
| `src/cli/` (13 files) | Args, auth commands, credential printing, `@file` processing, session picker, project trust, startup UI, experimental subcommand framework. |
| `src/core/export-html/` | Session → single self-contained HTML. Vendored `marked.min.js` + `highlight.min.js`. Two asset layouts (Node `dist/core/export-html/`, Bun `export-html/`). |

## Tests

- `vitest.config.ts` sets `PI_OFFLINE=1`; opt in with `allowNetwork()` from
  `test/test-network-env.ts`.
- `test/suite/harness.ts` is the mandated harness; `test/suite/README.md` spells out the rules
  (faux provider, no real APIs/keys/network/paid tokens, CI-safe and deterministic).
  `harness.ts:143-190` builds `Agent` + `AgentSession` **directly**, bypassing
  `createAgentSession()`, re-implementing just enough of the sdk's `streamFn` / `onPayload` /
  `onResponse` / `transformContext` wiring to exercise extension events.
- ~180 flat `*.test.ts` files plus `test/session-manager/` and `test/fixtures/`.
- `test/suite/regressions/` has ~80 files named `<github-issue#>-<slug>.test.ts`.
- `test/test-harness.ts` is the legacy path — do not extend it unless a missing capability forces
  it.

## Examples

`examples/sdk/` is a numbered progression (`01-minimal` … `13-session-runtime`).
`examples/extensions/` has 64 single-file examples plus 7 directory examples, catalogued in
`examples/extensions/README.md`.

Two rules from that README worth memorizing:

1. **Use `StringEnum` for string params** (`packages/ai/src/utils/typebox-helpers.ts`).
   `Type.Union([Type.Literal("a")])` does not work with the Google API.
2. **Persist state in the tool result `details`**, not a closure variable, so it survives forking.
   Reconstruct on `session_start` from `ctx.sessionManager.getBranch()`.

## Related

- [tui.md](tui.md) — the `pi-tui` library this mode renders with
- `packages/coding-agent/docs/` — 37 user- and extension-facing docs; `docs/extensions.md` is the
  extension authoring reference
- `packages/coding-agent/examples/sdk/` — runnable SDK progression
