# `packages/coding-agent` — the `pi` CLI

`@earendil-works/pi-coding-agent`, `bin.pi = dist/bundle/cli.js`. Builds on
`pi-agent-core`'s core agent, `pi-ai`, `pi-tui`, `pi-codemode`, and `pi-mcp`.

## Bootstrap

`src/cli.ts` is 6 lines and matters:

```ts
setupCli();
main(process.argv.slice(2));
```

`setupCli()` (`src/cli/setup.ts:4-13`) runs before any module does IO: sets `process.title`,
`PI_CODING_AGENT=true`, `AI_AGENT=pi`, silences `process.emitWarning`, and configures the undici
dispatcher before any provider SDK issues a request.

`src/rpc-entry.ts` duplicates that by hand because it must inject `--mode rpc` into argv.
`src/bun/cli.ts` adds sandbox and Bun runtime side effects for the compiled binary.

`src/main.ts` (1001 lines) is the single entry for all modes. Order:

offline flag (`:579`) → `pi auth` early exit (`:585`) → Windows self-update cleanup → managed
install cleanup → bootstrap `SettingsManager` → package commands (`:600`) → `pi config` (`:613`) →
`pi mcp` (`:617`) → `parseArgs` (`:623`) → `--version` → `--export` → `resolveAppMode` (`:654`) →
`takeOverStdout` for non-interactive → migrations (`:669`) → first-time setup (`:677`) → session-dir
and session selection (`:696`) → `createRuntime` factory (`:733`) → `createAgentSessionRuntime`
(`:866`) → `--help` / `--list-models` → piped stdin (`:900`) → initial message → theme init
(`:909`) → mode dispatch (`:952`, `:954`, `:988`).

Argument parsing is hand-written (`src/cli/args.ts`, a single `for` loop with an `else if` chain).
Unknown `--flags` land in `result.unknownFlags` and are resolved against extension-registered
flags (`core/agent-session-services.ts`).

## Four modes

`resolveAppMode` (`main.ts:112-124`): explicit `--mode rpc|json` wins; otherwise any non-TTY on
stdin/stdout, or `--print`, selects `print`; anything TTY selects `interactive`. A **second**
downgrade happens later (`:901-903`): piped stdin content flips interactive to print.

| Mode | File | Behavior |
|---|---|---|
| `print` | `modes/print-mode.ts` | Single-shot; prints only the final assistant text. Exit 1 on `stopReason: "error" \| "aborted"`. SIGTERM/SIGHUP handlers kill tracked detached children then exit 143/129. |
| `json` | same file, `mode:"json"` | One session-header line, then one `toJsonEvent(event)` per line. `modes/json-event.ts` strips cumulative snapshots: `message_start` gives the initial message, `message_update` carries deltas, `message_end` is authoritative. |
| `rpc` | `modes/rpc/rpc-mode.ts` | `Promise<never>`. JSONL commands on stdin, responses/events on stdout. `RpcCommand` union in `rpc-types.ts`. |
| `interactive` | `modes/interactive/interactive-mode.ts` | The TUI (7105 lines). |

## Three-layer session wiring

| File | Role |
|---|---|
| `src/core/sdk.ts` (479 lines) | `createAgentSession()` — builds the `pi-agent-core` `Agent` (`:403-453`), resolves model/thinking/tools, constructs `AgentSession`. |
| `src/core/agent-session-services.ts` (233 lines) | `createAgentSessionServices()` — cwd-bound `ModelRuntime` + `SettingsManager` + `DefaultResourceLoader`; applies extension-pending providers and flags. |
| `src/core/agent-session-runtime.ts` (447 lines) | Owns the current session and services; rebuilds both on cwd change, new session, fork, or switch. |

**The non-obvious core idea is deferred cwd resolution.** `--session` and `--resume` may select a
session from a *different* project, so project-local settings, resources, provider registrations,
and models must not be resolved until the target session cwd is known (`main.ts:685-718`). Hence
`createRuntime: CreateAgentSessionRuntimeFactory` is a closure that can rebuild everything for a
new cwd. When `--session <id>` resolves to another project, interactive mode prompts *"Fork this
session into current directory?"*.

`AgentSession` (`src/core/agent-session.ts`, 4458 lines) is the class **shared by all four modes** —
modes add only their own I/O layer. `AgentSessionEvent` (`:196-239`) extends `AgentEvent` with
session-only events: `agent_settled`, `queue_update`, `compaction_start/end`, `entry_appended`,
`auto_retry_start/end`, `bash_execution_update`, etc.

`ModelRuntime` (`core/model-runtime.ts`) implements `pi-ai`'s `Models` for the CLI. It composes
built-in providers, configured custom providers, virtual models, remote catalogs, per-user model
filters, and `AuthStorage`; `streamSimple` is what `sdk.ts` installs on the `Agent`.

## Tools

`src/core/tools/` exports **8** built-in tools: `read, bash, powershell, edit, write, grep, find,
ls` (`tools/index.ts`). Convenience groups: `createCodingToolDefinitions` (read/bash/edit/write,
`:164`), `createReadOnlyToolDefinitions` (read/grep/find/ls, `:173`), `createAllToolDefinitions`
(`:182`), plus `createCodingTools`/`createReadOnlyTools`/`createAllTools` for plain `AgentTool`s.

Each tool file exports **two** things: a rich `create*ToolDefinition` (name, label, description,
typebox `parameters`, `promptSnippet`, `promptGuidelines`, `renderShell`, `renderCall`,
`renderResult`, `execute`, optional `outputSchema`/`exposure`/`namespace`) and a thin
`create*Tool` returning a plain pi-agent-core `AgentTool`. The registry stores **definitions**;
`tools/tool-definition-wrapper.ts` adapts them to `AgentTool`. A test can inject a plain
`Record<string, AgentTool>` via `baseToolsOverride`, and it is synthesized back into a
`ToolDefinition`.

`*Operations` interfaces are pluggable IO seams — `read.ts:48`, `bash.ts:75`; for example
`examples/extensions/ssh.ts` swaps them to run every tool over SSH.

`tools/renderers/` is a deliberate bundle split: it re-exports only the renderers so a
rendering-only process does not load typebox (~17 MB of module graph).

Registration is `AgentSession._buildRuntime` (`agent-session.ts:3666`). When no tools are passed,
the SDK resolves `settings.defaultTools` (or `["read", "bash", "edit", "write"]`), applies
`+name`/`-name` modifiers and `excludeTools`, and passes the result as `initialActiveToolNames`.
`AgentSession` falls back to `["read", "bash", "edit", "write"]` (`:3710-3712`).

## Hooks → extensions

There is no `src/core/hooks/` directory. Hooks were renamed to extensions; the only trace is the
deprecation warning in `src/migrations.ts`. The replacement is `src/core/extensions/` (8 files) —
the engine — plus `src/extensions/`, the shipped built-ins.

| File | Role |
|---|---|
| `core/extensions/types.ts` (2308 lines) | `ExtensionAPI`, `ExtensionContext`, `ToolDefinition`, event interfaces, `defineTool`, `ToolExposure`, `ToolNamespace`. |
| `core/extensions/loader.ts` (879 lines) | jiti-based TS loading, discovery, transactional init. |
| `core/extensions/runner.ts` (1565 lines) | `ExtensionRunner` — dispatches events and binds core actions. |
| `core/extensions/jiti-loader.ts` / `jiti-static-loader.ts` | jiti loading for Node vs embedded/Bun builds. |
| `core/extensions/virtual-modules.ts` / `wrapper.ts` | Embedded module aliases and extension wrapping. |

`ExtensionEvent` is unioned at `types.ts:1405`, exposed through typed `pi.on(...)` overloads.
Groups: trust/resources, session lifecycle, context/system prompt, provider (`before_provider_request`,
`before_provider_headers`, `after_provider_response`, `provider_stream_event`,
`cache_warming_decision`), agent loop, UI, input, and message/tool events.

**Built-ins** (`src/extensions/index.ts`):

- `llama.cpp` — local llama.cpp server management.
- `codemode` — model-written JavaScript calling other tools (inactive by default; MCP may activate
  it).
- `tool-search` — deferred tool search/load (inactive by default).
- `mcp` — MCP servers from `mcp.json`, `pi.registerMcpServer()`, `/mcp`, and `pi mcp`.

The last three are `replaceable`: an extension that registers the same tool or command takes over.

**Discovery** (`loader.ts:831-879`), in order: `<cwd>/.pi/extensions/`, `<agentDir>/extensions/`,
then explicit `-e` paths. Directory scanning is **one level deep** — a subdirectory resolves via
`package.json` with a `pi` manifest field (for example `pi.extensions`), else `index.ts`/`index.js`
(`loader.ts:747-825`).

**Loading** picks a jiti strategy (`loader.ts:577-584`): virtual modules for a Bun binary/SEA,
`tsconfigPaths` when running `.ts` sources, or a concrete alias map for unbundled dist.
`getAliases()` maps `@earendil-works/*` (and legacy `@mariozechner/*`) to entry files, and maps bare
`@earendil-works/pi-ai` to the **`compat` entrypoint**, so extensions written against the old
global API keep working.

**Extension load is transactional.** `createExtensionAPI()` returns `{api, commit, discard}`
(`loader.ts:247-550`): flag defaults, provider registrations, and MCP servers made during the
factory call are queued and applied only on `commit()`. On throw, `discard()` unsubscribes
everything and the API is permanently dead.

Every `ExtensionAPI` method calls `assertActive()` first. After `ctx.newSession()`, `ctx.fork()`,
`ctx.switchSession()`, or `ctx.reload()`, a captured context is **stale** — the loader invalidates
subscriptions because actions like `newSession` are only safe in user-initiated commands.

## Package manager and MCP commands

- `pi install|remove|list|update [--local]` (`package-manager-cli.ts`) manages Pi packages from
  npm, git, URLs, or local paths. Personal declarations go to `~/.pi/agent/settings.json`; `-l`
  writes `.pi/settings.json` and requires project trust. `pi update` updates Pi; `--extensions`,
  `--models`, and `--all` select package or catalog updates.
- `pi config [--local]` opens the resource/config selector.
- `pi mcp add|remove|list|login|logout` manages MCP servers in `~/.pi/agent/mcp.json` and, in
  trusted projects, `.pi/mcp.json`. Inside a session, `/mcp` provides the same controls.
- Startup tolerates a configured package that fails to install: it warns, skips that package's
  resources, and retries on the next start or `/reload`.

## System prompt assembly

`src/core/system-prompt.ts` builds the prompt as a **map of named sections, not a flat string**:

```ts
const sections: SystemPromptSections = { preamble: promptSections.preamble };
for (const [name, content] of Object.entries(promptSections)) {
  if (name !== "preamble") sections[name] = `<${name}>\n${content}\n</${name}>`;
}
```

Order: `preamble` → `tools` → `rules` → `docs` → (`addendum`) → (`project_context`) → (`skills`)
→ `cwd` → custom sections (`:128-190`). A `customPrompt` replaces everything but the addendum.

Two consequences:

- `diffSystemPromptSections` (`:217`) produces a sparse patch, and
  `AgentSession._preparePromptAndToolLoadout` (`agent-session.ts:1758`) emits a `SystemMessage`
  **only when the prompt actually changed**. An unchanged prompt adds nothing to the transcript.
- `_rebuildSystemPrompt` (`agent-session.ts:1720`) gathers snippets from the live tool registry, so
  **enabling a tool changes the system prompt**.

Context files (`AGENTS.md` / `CLAUDE.md`) load in `src/core/resource-loader.ts:211-300`, trying
`["AGENTS.override.md", "AGENTS.md", "AGENTS.MD", "CLAUDE.md", "CLAUDE.MD"]` and walking cwd →
root. `findShadowedContextFile` (`:241-275`) prevents a main-repo `AGENTS.md` from being applied
twice in nested git worktrees.

Skills are `src/core/skills.ts` (one flat file, not a directory). It honors `.gitignore` /
`.ignore` / `.fdignore` at every ancestor level via the `ignore` package.

## Assets and config

`src/config.ts` (656 lines) is the single source of truth for asset paths; `AGENTS.md` requires
using its helpers instead of `__dirname`. Every accessor is a **function**, because the right
answer differs per distribution:

| Helper | Bun binary | Node (source or dist) |
|---|---|---|
| `getPackageDir()` | `dirname(process.execPath)` | `findNodePackageDir(__dirname)`, `PI_PACKAGE_DIR` override |
| `getThemesDir()` | `<pkg>/theme` | `<pkg>/{src\|dist}/modes/interactive/theme` |
| `getExportTemplateDir()` | `<pkg>/export-html` | `<pkg>/{src\|dist}/core/export-html` |
| `getInteractiveAssetsDir()` | `<pkg>/assets` | `<pkg>/{src\|dist}/modes/interactive/assets` |

The `{src|dist}` choice is `existsSync(join(packageDir, "src")) ? "src" : "dist"`. `findNodePackageDir`
has a Bun-specific fix: `build:binary` puts Bun metadata inside `dist/`, so a directory literally
named `dist` with a `package.json` one level up resolves to the parent.

`config.ts` reads `package.json` for `PACKAGE_NAME`/`VERSION` and its `piConfig` block for
`APP_NAME` (default `pi`) and `CONFIG_DIR_NAME` (default `.pi`). It derives the
`PI_CODING_AGENT_DIR` / `PI_CODING_AGENT_SESSION_DIR` env names from `APP_NAME`. A fork that
changes `piConfig.name` gets its own namespace. `getPackageJsonPath`, `getReadmePath`, `getDocsPath`, `getExamplesPath`,
`getChangelogPath`, `getAuthPath`, `getSettingsPath`, `getModelsPath`, `getBinDir`, and
`getSessionsDir` are the other accessors.

## Interactive mode

`src/modes/interactive/interactive-mode.ts` is the mode class;
`src/modes/interactive/tui-renderer.ts` is the composition root. `createInteractiveTui(options)`
builds either `TuiMainScreen` (`tuiMode: "regular"`) or `TuiAltScreen`
(`tuiMode: "fullscreen"`). `createInteractiveTuiReference(getTui)` returns a **`Proxy` that survives
renderer replacement**, so long-lived components hold one stable `TUI` reference while
`switchTuiMode()` swaps the underlying renderer. Components are created once and **remounted**
rather than rebuilt (`interactive-mode.ts:875-932`).

The mounted component tree is fixed (`interactive-mode.ts:984-992`): `documentContainer`,
`pendingMessagesContainer`, `statusContainer`, `widgetContainerAbove`, `editorContainer`,
`widgetContainerBelow`, `footerContainer`, plus `headerContainer` and a `TuiLayouts.ScrollView`
transcript from `chat-viewport.ts`.

Startup ordering has one real constraint: `ui.start()` runs **before** `rebindCurrentSession()` so
`session_start` handlers can use interactive dialogs (`:1000`, `:1102`).

`components/` has 48 files (selectors, transcript renderers, editors, footers, demos, bug report,
Mermaid, image/model/theme selectors). `theme/` defines the token set and ships `dark.json` and
`light.json`. Because pi renders **user-authored** themes, `setThemeJsonValidator(validateThemeJson)`
is called before any theme loads (`main.ts:577`) — see `theme/theme.ts`.

## Migrations

`src/migrations.ts` runs `runMigrations(cwd)` once per startup (`main.ts:669`).

| Function | Line | What |
|---|---|---|
| `migrateAuthToAuthJson` | 21 | Merges legacy `oauth.json` + `settings.json.apiKeys` into `auth.json` (mode 0600). No-op if it exists. Returns the migrated provider ids. |
| `migrateSessionsFromAgentRoot` | 84 | v0.30.0 bug: sessions went to `~/.pi/agent/*.jsonl` instead of `~/.pi/agent/sessions/<encoded-cwd>/`. |
| `migrateCommandsToPrompts` | 137 | Moves legacy `commands/` files into `prompts/`. |
| `migrateKeybindingsConfigFile` | 157 | Rewrites `keybindings.json` through `migrateKeybindingsConfig()`. |
| `migrateToolsToBin` | 177 | Moves managed `fd`/`rg` from `agentDir/tools/` to `agentDir/bin/`. |
| `migrateExtensionSystem` | 257 | Warns about leftover `hooks/` and non-binary `tools/`; returns deprecation warnings shown at startup. |

## Other directories

| Path | What |
|---|---|
| `src/experimental/` | Only `source-resolver.ts`, the tsconfig-paths resolver used by `pi-test.sh` and schema generation. |
| `src/bun/` | Compiled-binary support: sandbox env setup/restore, Bun runtime and QuickJS wasm shims. |
| `src/utils/` (37 files) | Paths (WSL, Git-Bash), child processes, images (Photon WASM resize worker), clipboard, markup, git, zip, syntax highlighting. |
| `src/cli/` (12 files) | Args, auth commands, credential printing, `@file` processing, session picker, project trust, startup UI, model listing. |
| `src/core/` (59 files) | Session, runtime, models, settings, trust, compaction, caching, extensions, tools, export. |
| `src/core/export-html/` | Session → single self-contained HTML. Vendored `marked.min.js` + `highlight.min.js`. Two asset layouts (Node `dist/core/export-html/`, Bun `export-html/`). |
| `src/modes/rpc/` | RPC server, client, JSONL framing, command types. |
| `src/extensions/` | The four built-ins: `llama`, `codemode`, `tool-search`, `mcp`. |

## Tests

- `vitest.config.ts` sets `PI_OFFLINE=1`; opt in with `allowNetwork()` from
  `test/test-network-env.ts`.
- `test/suite/harness.ts` is the mandated harness; `test/suite/README.md` spells out the rules
  (faux provider, no real APIs/keys/network/paid tokens, CI-safe and deterministic).
  `createHarness()` builds `Agent` + `AgentSession` **directly**, bypassing `createAgentSession()`,
  re-implementing enough of the SDK's `streamFn` / `onPayload` / `onResponse` / `transformContext`
  wiring to exercise extension events.
- 209 flat `*.test.ts` files, plus `test/session-manager/`, `test/fixtures/`, `test/mcp-conformance/`,
  and `test/suite/`.
- `test/suite/regressions/` has 84 files named `<github-issue#>-<slug>.test.ts`.
- `test/test-harness.ts` is the legacy path — do not extend it unless a missing capability forces it.

## Examples

`examples/sdk/` is a numbered progression (`01-minimal` … `14-codemode-mcp`). There are 70
single-file extension examples plus 9 directory examples, catalogued in
`examples/extensions/README.md`.

Two rules worth memorizing:

1. **Use `StringEnum` for string params** (`packages/ai/src/utils/typebox-helpers.ts`).
   `Type.Union([Type.Literal("a")])` does not work with the Google API.
2. **Persist state in the tool result `details`**, not a closure variable, so it survives forking.
   Reconstruct on `session_start` from `ctx.sessionManager.getBranch()`.

## Related

- [tui.md](tui.md) — the `pi-tui` library this mode renders with
- `packages/coding-agent/docs/` — 44 user- and extension-facing docs; `docs/extensions.md`,
  `docs/mcp.md`, `docs/codemode.md`, and `docs/packages.md` are the main ones
- `packages/coding-agent/examples/sdk/` — runnable SDK progression
