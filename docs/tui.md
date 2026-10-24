# `packages/tui` — terminal UI library

`@earendil-works/pi-tui`. Runtime deps: `get-east-asian-width`, `marked`, and
`marked-cjk-friendly`. The only package using `node:test` instead of vitest, and the home of
the repo's committed native addons.

## The two renderers

| | `TuiMainScreen` (`mode: "regular"`) | `TuiAltScreen` (`mode: "fullscreen"`, implements `ViewportTUI`) |
|---|---|---|
| Buffer | Main buffer + native scrollback; `ENTER_ALT_SCREEN` never sent | `\x1b[?1049h/l`, autowrap disabled |
| Document | Unbounded, grows downward | Fixed `terminal.rows` viewport; app owns scrolling |
| Repaint | Line-range diff vs `previousLines` | Row-addressed: `ESC[{row+1};1H` per changed row |
| Mouse | Not captured (terminal owns scrollback) | Owns mouse: selection, scrollbars, wheel chaining, OSC-8 clicks |
| On stop | Prints a trailing newline (unless `preserveScreen`) | **Restores the main buffer and prints the entire final document** so history is not lost |

`pi` picks one in `packages/coding-agent/src/modes/interactive/tui-renderer.ts:22-56`.

## Core model

`src/tui.ts` (1543 lines):

- `Component` (`:116-141`): `render(width): string[]` and `invalidate()` are required;
  `handleInput?`, `handleMouse?`, `wantsKeyRelease?` are optional.
- `Container` (`:358-418`): flat children list, concatenates child line arrays. It caches
  per-child heights in `mouseLayout` so hit-testing does not re-render children.
- `TUI` (`:463-492`) is the public interface. `ViewportTUI` (`:496-503`) adds
  `setLayoutRoot()` and is keyed by `Symbol.for("@earendil-works/pi-tui/viewport")`.
- `TuiBase extends Container` (`:505`) is the shared engine. `abstract doRender()` (`:551`) is the
  **only** renderer-specific method; focus, overlays, cursor extraction, and the render scheduler
  live here. `TuiBase` is exported from `src/tui.ts` but **not** re-exported by `src/index.ts`.

`CURSOR_MARKER = "\x1b_pi:c\x07"` (`tui.ts:196`) is a zero-width APC sequence the focused component
emits. The TUI strips it and moves the real hardware cursor there for **IME candidate-window
positioning**. Fake cursors use `FAKE_CURSOR_START/END` (`:199-200`, `renderFakeCursor` `:206-208`).

## Differential rendering

`src/tui-main-screen.ts` (`doRender` `:247-616`). The whole sequence:

1. `render(width)` → resolve fake cursors → composite overlays → `extractCursorPosition` →
   `applyLineResets`.
2. First render → `fullRender(false)`, no clear.
3. **Width change → always a full redraw**, because wrapping changed.
4. **Height change → full redraw, except under Termux** (`:345-352`): *"Termux changes height when
   the software keyboard shows or hides… a full redraw causes the entire history to replay on
   every toggle."*
5. `setClearOnShrink` (default `false`) forces a full redraw when content shrinks below
   `maxLinesRendered` **and** no overlays are active (`:354-361`).
6. The diff is a single linear scan computing `firstChanged` / `lastChanged`. If nothing changed,
   no line content is written, but cursor-positioning escapes may still be emitted.
7. Kitty images force the changed range to expand to cover their reserved rows.
8. If `firstChanged < prevViewportTop` (a change above the visible area) → full redraw.
9. Otherwise move the cursor by a relative delta, emit `\r\n` only for appended content, and
   render **only the changed lines**, each preceded by `\x1b[2K`.
10. Everything is wrapped in **synchronized output** `ESC[?2026h` … `ESC[?2026l`.

`BoundedTerminalWriter` (`:18-74`) streams output in 1 MiB **UTF-16 character** chunks so a huge
render never builds a string past V8's max length, and never splits a surrogate pair.

**A too-wide line is a hard crash in the main screen**: if `visibleWidth(line) > width`, the
renderer writes `pi-tui-crash.log` with all rendered lines, calls `stop()`, then throws telling you
to use `visibleWidth()` / `truncateToWidth()` (`:518-545`). The alt screen clamps instead.

Debug switches: `PI_TUI_DEBUG_REDRAW=1` logs full-redraw reasons to `<logDirectory>/pi-tui-debug.log`;
`PI_TUI_DEBUG=1` dumps per-frame state to `/tmp/tui/render-*.log`.

### Scheduling

`MIN_RENDER_INTERVAL_MS = 16` (`tui.ts:517`) — renders are throttled to ~60 fps.
`requestRender()` coalesces into a `process.nextTick` + `setTimeout`. **Keyboard input bypasses the
throttle** via `requestImmediateRender()` (`:1038-1053`; call/comments at `:1154-1156`):
*"Keyboard input is latency-sensitive. Avoid the throttled timer path, where even setTimeout(0) can
take a full 16 ms tick on Windows."*

## Layout

`src/layout-node.ts` (52 lines) is the opt-in protocol: a component opts in by implementing
`[LAYOUT_NODE](): LayoutNode` with `LAYOUT_NODE = Symbol.for("@earendil-works/pi-tui/layout-node")`.

**Only the alt-screen renderer uses it.** `layout.ts:106-247` `layoutComponent()` recursively
produces a `LayoutBox {rect, clip, children, lines, scrollView, layer}` tree each frame.

- Non-layout (leaf) components cache `render(width)` per width in `LayoutContext.renderCache`, so
  a component measured by a `Stack` and then painted is only rendered once per frame.
- For a scroll node, the child is laid out at `y - scrollTop`, then
  `state.updateLayout(contentHeight, viewportHeight, requestRender)` is called, then the box is
  translated — this is how scrolling does not re-render children.
- `paintBox` (`:329-376`) has a fast path: a full-width box on an untouched row assigns the source
  string directly instead of compositing through ANSI/grapheme segmentation.
- Geometry is rebuilt from scratch every frame; only `render()` output is cached.

## Terminal I/O

`src/terminal.ts` (630 lines). `ProcessTerminal.start()` (`:199-230`): raw mode, utf8, bracketed
paste `ESC[?2004h`, `resize` listener, self-`SIGWINCH` (POSIX only; swallows `EACCES` from
seccomp/LSM), Windows VT input, then **Kitty keyboard protocol negotiation**.

Kitty negotiation (`:234-430`) sends `ESC[>7u ESC[?u ESC[c` — push flags 1|2|4, then a Device
Attributes query as a sentinel. If DA returns first, the terminal doesn't know Kitty, so
modifyOtherKeys is enabled immediately with no startup timeout. Responses can arrive split, hence
a 150 ms fragment-flush timer. A bare `\r` is rewritten to `\x1b[13;2u` for native Shift+Enter on
Apple Terminal/Windows.

Other details:

- `drainInput()` (`:434-470`) disables Kitty first, then drains stdin so Kitty key-**releases**
  don't leak to the parent shell over slow SSH.
- `stop()` (`:472-524`) pauses stdin — the comment at `:515-517` says this fixes a race where a
  buffered `Ctrl+D` could close the parent shell over SSH.
- Escape-sequence timeout is 10 ms, **100 ms over SSH** (`SSH_CONNECTION`/`SSH_TTY`),
  overridable with `PI_TUI_ESC_TIMEOUT`.
- `PI_TUI_WRITE_LOG` mirrors all writes to a file (or a timestamped file when it names a directory).
- OSC 9;4 terminal progress with a 1 s keepalive, and OSC 7501 program status.
- Input reassembly across stdin chunks lives in `src/stdin-buffer.ts`; high bytes are converted
  to `ESC + (byte-128)` for legacy Meta compatibility.

## Keys

`src/keys.ts` (1455 lines) is **parsing**, not keybindings. `src/keybindings.ts` (320) is the
configurable layer.

- `KeyId` (`keys.ts:151`) is a type-safe union built recursively from `ModifiedKeyId`, so a typo
  is a compile error. `Key.ctrl("c")` helpers give autocomplete.
- Three input vocabularies: **Kitty CSI-u**, **xterm modifyOtherKeys**, and **legacy** sequences.
- **Non-Latin layouts**: Kitty flag 4 reports a `baseLayoutKey` (PC-101 position), so `Ctrl+С`
  (Cyrillic) matches `ctrl+c`. But the base-layout key is only used when the codepoint is not a
  recognized a-z letter or known symbol — otherwise Dvorak/Colemak/xremap would produce false
  matches (`:699-716`).
- `LOCK_MASK` (Caps Lock + Num Lock) is masked out of every modifier comparison.
- Ambiguity is resolved by **global protocol state, not platform**: when Kitty is active `\x1b\r`
  means shift+enter; when it is off, `\x1b\r` is alt+enter.
- `isKeyRelease` / `isKeyRepeat` are string-sniffing helpers that first bail if the data contains
  the bracketed-paste marker, so pasted Bluetooth MACs like `90:62:3F:A5` aren't misread.

### Keybindings

`TUI_KEYBINDINGS` (`keybindings.ts:71`) is the default table. `KeybindingsManager` provides
`matches`, `getKeys`, `getConflicts`, `setUserBindings`, `getUserBindings`, and
`getResolvedBindings`. `Keybindings` lists every binding id as `true` so downstream packages can
declaration-merge new ids.

**All components call `getKeybindings().matches(...)`, never `matchesKey` directly.** That is the
extension point, and it is why `AGENTS.md` forbids hardcoding key checks. (`TuiBase` itself uses
`matchesKey` only for the Shift+Ctrl+D debug key.)

Some shipped defaults are deliberately **unbound** (`defaultKeys: []`):
`tui.editor.historyPrevious/Next`, `tui.altScreen.halfPage*`, `lineUp`, `lineDown` — those are
host policy. The alt screen also ships search bindings (`ctrl+shift+f`, enter/ctrl+g,
shift+enter/ctrl+shift+g, escape) and prompt navigation.

## Platform-specific modifier handling — three layers

1. **Protocol level** (no platform checks): Kitty vs modifyOtherKeys vs legacy, plus the
   Kitty-active ambiguity rules.
2. **Terminal-emulator checks**: `isAppleTerminalSession()` (`TERM_PROGRAM === "Apple_Terminal"`),
   `isWindowsTerminalSession()` (`WT_SESSION` and not SSH), tmux/Zellij/Sty, Termux
   (`TERMUX_VERSION`), WezTerm, VS Code.
3. **Native OS key state** via `src/native-modifiers.ts`:
   `isNativeModifierPressed("shift" | "command" | "control" | "option")`. Used only to
   disambiguate native Shift+Enter on macOS/Windows, because those terminals send a bare `\r`
   with no modifier information.

## Native modules

Three hand-written N-API addons, with prebuilt `.node` binaries committed under
`native/<platform>/prebuilds/`:

| Platform | Source | Language | Provides |
|---|---|---|---|
| darwin | `native/darwin/src/darwin-platform.m` | Objective-C (AppKit/CoreGraphics) | modifier keys, async text/image/file clipboard |
| linux | `native/linux/src/linux-platform-x11.c` | C (`libxcb.so.1`) | async X11 clipboard **reads only** |
| win32 | `native/win32/src/win32-platform.c` | C (links system libs; no Node headers) | console input setup, modifier keys, async clipboard |

They exist because Node has no API for reading physical modifier-key state, doing asynchronous
clipboard I/O, or enabling Windows VT input. `NativeClipboard` (`src/native-platform.ts:9-15`)
distinguishes `undefined` (unavailable) from `null` (no text/image) and adds `getFilePaths?`;
Linux "uses command-line tools to retain clipboard ownership instead" of a native write.

Loading (`native-platform.ts:27-66`, `native-module-path.ts:13-36`): x64/arm64 only, tries several
candidate paths because standalone single-file binaries have no installed `pi-tui` package.
`getNativeClipboard()` requires `DISPLAY` on Linux.

Build: `npm run build:native:{darwin,linux,win32}`. Cross-compiling darwin requires osxcross plus
a macOS SDK under Apple's license (`native/darwin/README.md:13-24`).

## Components (`src/components/`, 18 files)

| Component | Notes |
|---|---|
| `Text` | Multi-line; `paddingX=1`/`paddingY=1` defaults; caches by `(text, width)`. Tabs → 3 spaces. |
| `TruncatedText` | Single-line, truncates and pads to exact width. For status lines. |
| `Box` | Padding + background over all children; revalidates its cache by **sampling** `bgFn("test")`. |
| `Spacer` | Fixed blank vertical space. |
| `VStack` / `HStack` | Flexbox-ish `basis`/`grow`/`shrink`/`minSize`/`maxSize`; shared `Stack` base in `stack.ts`. |
| `ScrollView` | Vertical only (throws otherwise); exactly one child. `follow: "none" \| "end"`, `overscroll: "chain" \| "contain"`, transient `scrollbar: "auto"`. |
| `Input` | Single-line `Focusable`. Bracketed paste, Emacs kill ring, undo, reverse-video fake cursor. |
| `Editor` | Multi-line `Focusable`; word wrap, autocomplete, slash-command + file completion, **paste markers**: pastes become `[paste #N …]` marker spans merged into one atomic segment (`editor.ts:48-100`). |
| `Markdown` | `marked` + `marked-cjk-friendly` + a `MarkdownTheme`, optional `highlightCode`, LaTeX via `renderLatex` with fallback to raw source. |
| `SelectList` | Prefix filter, mouse. The visible window is **centered** on the selection. |
| `SettingsList` | Label/value rows with cycling `values[]` and chainable submenus. |
| `Image` | Inline via Kitty/iTerm2, `imageFallback` text otherwise. |
| `Loader` / `CancellableLoader` | Braille spinner. `setIndicator` distinguishes "not configured" from "configured". Escape aborts. |
| `MouseRegion` | Adds `handleMouse` without changing rendering. |

Also `src/editor-component.ts` (`EditorComponent` interface for custom editors),
`src/components/alt-screen-flash.ts` (transient reverse-video messages), and
`src/alt-screen-search.ts` (the search panel/index used by the alt screen).

## Notable extras

- `src/latex.ts` (1507 lines) — a **from-scratch LaTeX → Unicode math renderer** with no
  dependency. The parser returns `undefined` on unsupported syntax so callers fall back to raw
  source. Two-pass layout: private-use markers, then composition into physical lines with a real
  baseline. Fractions use `───`; `cases`, matrices, `align*`/`aligned`/`gather`/`multline`/`split`,
  `array`, and the `pmatrix` family are supported.
- `src/terminal-image.ts` (770 lines) — graphics protocol plus capability detection. **tmux/screen
  deliberately disable features**: images `null` under tmux, hyperlinks off under `screen`, and
  under tmux hyperlinks only if `tmux display-message -p '#{client_termfeatures}'` lists them.
  Unknown terminals get `hyperlinks: false` because OSC 8 renders invisibly as plain text
  (`:141-144`). Header-only dimension parsers for PNG/JPEG/GIF/WebP — no image decoding.
  Overridable with `PI_IMAGE_PROTOCOL`, `PI_TRUE_COLOR`, `PI_HYPERLINKS`.
- `src/autocomplete.ts` — `CombinedAutocompleteProvider` handles slash commands, slash-command
  **arguments**, `@` file mentions, and paths. Fuzzy `@` search shells out to `fd`. Smart prefix
  extraction handles `~/`, `./`, `../`, absolute, and quoted paths.
- `CURSOR_MARKER` / fake cursor markers: zero-width APC sequences used for IME positioning while
  the component draws a fake inverse-video cursor.
- `SEGMENT_RESET = "\x1b[0m\x1b]8;;\x07"` is appended to every line — SGR *and* OSC-8 hyperlink
  reset.

## Alt-screen specifics

`src/tui-alt-screen.ts` (1797 lines) adds: selection with character/word/line granularity and
500 ms double-click, edge auto-scroll, `copyOnSelect` (default true) via OSC 52; the built-in
search panel (`src/alt-screen-search.ts`, top-right, query + match navigation);
`scrollToPrompt` scanning for `OSC133_PROMPT_START` markers.

Three deliberate workarounds worth knowing:

- Under TMUX/ZELLIJ/STY it enables button-motion tracking instead of all-motion, *"because
  multiplexers can lag when every pointer movement is forwarded."*
- **iTerm2 images are disabled in alt screen**: capabilities are temporarily overridden so images
  render as text, because the iTerm2 protocol has no delete/crop operation.
- For WezTerm, a frame that redraws images clears the affected rows first, draws text rows, then
  draws image rows last, because WezTerm erases Kitty image cells when a later `EL` clears a
  covered row.

Clipboard honesty (`:1479-1499`): if the host injects `copySelection`, it is used and "Copied!" is
only reported on real success — a bare OSC 52 write can claim success while leaving the clipboard
untouched (macOS Terminal.app, tmux without passthrough).

## Tests

`node --test --test-reporter=dot --test-reporter-destination=stdout test/*.test.ts`, importing
`src/*.ts` directly. Works because Node ≥22.19 strips types natively.

Two `Terminal` test doubles:

1. `test/virtual-terminal.ts` — real `@xterm/headless`, always reports
   `kittyProtocolActive === true`, adds test-only `sendInput()` and `resize()`.
2. Hand-rolled terminals when you need to inspect raw bytes — e.g. `BoundedWriteTerminal`
   (`test/tui-render.test.ts:45-66`) records `writes[]`.

`setCapabilities` / `resetCapabilitiesCache` / `setCellDimensions` exist so tests can exercise both
image-protocol paths.

Runnable demos: `test/chat-simple.ts` (131 lines — the smallest complete example),
`test/key-tester.ts`, `test/image-test.ts`, plus `render-churn-bench.ts` and
`alt-screen-large-transcript-bench.ts`.

## Consumer example

Smallest real in-repo usage, `packages/coding-agent/src/cli/config-selector.ts:20-61`:

```ts
const ui: TUI = new TuiMainScreen(
  new ProcessTerminal(),
  options.settingsManager.getShowHardwareCursor(),
  options.agentDir,
);
ui.setClearOnShrink(...);
ui.addChild(selector);
ui.setFocus(selector.getResourceList());
ui.start();
// on close: ui.stop(); resolve();
```
