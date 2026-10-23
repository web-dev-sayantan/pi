# `packages/tui` — terminal UI library

`@earendil-works/pi-tui`. Runtime deps: only `get-east-asian-width` and `marked`. The only
package using `node:test` instead of vitest, and the only one with compiled native code committed
to the repo.

## The two renderers

| | `TuiMainScreen` (`mode: "regular"`) | `TuiAltScreen` (`mode: "fullscreen"`, implements `ViewportTUI`) |
|---|---|---|
| Buffer | Main buffer + native scrollback; `ENTER_ALT_SCREEN` never sent | `\x1b[?1049h/l`, autowrap disabled |
| Document | Unbounded, grows downward | Fixed `terminal.rows` viewport; app owns scrolling |
| Repaint | Line-range diff vs `previousLines` | Row-addressed: `ESC[{row+1};1H` per changed row |
| Mouse | Not captured (terminal owns scrollback) | Owns mouse: selection, scrollbars, wheel chaining, OSC-8 clicks |
| On stop | Prints a trailing newline | **Restores the main buffer and prints the entire final document** so history is not lost |

`pi` picks one at `packages/coding-agent/src/modes/interactive/tui-renderer.ts:18-48`.

## Core model

`src/tui.ts` (1473 lines):

- `Component` interface (`:111-136`): `render(width): string[]` is required; `handleInput?`,
  `handleMouse?`, `wantsKeyRelease?` optional; `invalidate()` required.
- `Container` (`:319-379`): flat children list, concatenates child line arrays. It caches
  per-child heights in `mouseLayout` so hit-testing does not re-render children.
- `TuiBase extends Container` (`:466`) is the shared engine. `abstract doRender()` (`:510`) is the
  **only** renderer-specific method; everything else — focus, the overlay stack, overlay
  compositing, cursor extraction, the render scheduler — lives here. `TuiBase` is *not* exported;
  `Container` is a class, `TUI` is an interface.

## Differential rendering

`src/tui-main-screen.ts:247-616`. The whole sequence:

1. `render(width)` → composite overlays → `extractCursorPosition` → `applyLineResets`.
2. First render → `fullRender(false)`, no clear.
3. **Width change → always a full redraw**, because wrapping changed.
4. **Height change → full redraw, except under Termux** (`:344-351`): *"Termux changes height when
   the software keyboard shows or hides… a full redraw causes the entire history to replay on
   every toggle."*
5. `setClearOnShrink` (default `false`) forces a full redraw when content shrinks below
   `maxLinesRendered`.
6. The diff itself is a single linear scan computing `firstChanged` / `lastChanged`. If nothing
   changed, only the cursor is repositioned — **no write at all**.
7. Kitty images force the changed range to expand to cover their reserved rows.
8. If `firstChanged < prevViewportTop` (a change above the visible area) → full redraw.
9. Otherwise move the cursor by a relative delta, emit `\r\n` only for appended content, and
   render **only the changed lines**, each preceded by `\x1b[2K`.
10. Everything is wrapped in **synchronized output** `ESC[?2026h` … `ESC[?2026l`.

`BoundedTerminalWriter` (`:18-74`) streams output in 1 MiB chunks so a huge render never builds a
string past V8's max length, and never splits a surrogate pair.

**A too-wide line is a hard crash**: if `visibleWidth(line) > width`, the renderer writes
`pi-tui-crash.log` with all rendered lines, calls `stop()`, then throws telling you to use
`visibleWidth()` / `truncateToWidth()` (`:517-544`).

### Scheduling

`MIN_RENDER_INTERVAL_MS = 16` — renders are throttled to ~60 fps. `requestRender()` coalesces
into a `process.nextTick` + `setTimeout`. **Keyboard input bypasses the throttle** via
`requestImmediateRender()` (`:1087`): *"Keyboard input is latency-sensitive. Avoid the throttled
timer path, where even setTimeout(0) can take a full 16 ms tick on Windows."*

## Layout

`src/layout-node.ts` (51 lines) is the opt-in protocol: a component opts in by implementing
`[LAYOUT_NODE](): LayoutNode` with `LAYOUT_NODE = Symbol.for("@earendil-works/pi-tui/layout-node")`.

**Only the alt-screen renderer uses it.** `layout.ts:106-247` `layoutComponent()` recursively
produces a `LayoutBox {rect, clip, children, lines, scrollView, layer}` tree each frame.

- Non-layout (leaf) components cache `render(width)` per width in `LayoutContext.renderCache`, so
  a component measured by a `Stack` and then painted is only rendered once per frame.
- For a scroll node, the child is laid out at `y - scrollTop`, then
  `state.updateLayout(contentHeight, viewportHeight, requestRender)` is called, then the box is
  translated — this is how scrolling does not re-render children.
- `paintBox` (`:330-377`) has a fast path: a full-width box on an untouched row assigns the
  source string directly instead of compositing through ANSI/grapheme segmentation.
- Geometry is rebuilt from scratch every frame; only `render()` output is cached.

## Terminal I/O

`src/terminal.ts` (547 lines). `ProcessTerminal.start()` (`:172-203`): raw mode, utf8,
bracketed paste `ESC[?2004h`, `resize` listener, self-`SIGWINCH` (POSIX only; swallows `EACCES`
from seccomp/LSM), Windows VT input, then **Kitty keyboard protocol negotiation**.

Kitty negotiation (`:243-343`) sends `ESC[>7u ESC[?u ESC[c` — push flags 1|2|4, then a Device
Attributes query as a sentinel. If DA returns first, the terminal doesn't know Kitty, so
modifyOtherKeys is enabled immediately with no startup timeout. Responses can arrive split, hence
a 150 ms fragment-flush timer.

Other details:

- `drainInput()` (`:384-420`) disables Kitty first, then drains stdin so Kitty key-**releases**
  don't leak to the parent shell over slow SSH.
- `stop()` (`:422-468`) pauses stdin — the comment at `:459-461` says this fixes a race where a
  buffered `Ctrl+D` could close the parent shell over SSH.
- Escape-sequence timeout is 10 ms, **100 ms over SSH** (`SSH_CONNECTION`/`SSH_TTY`),
  overridable with `PI_TUI_ESC_TIMEOUT`.
- `PI_TUI_WRITE_LOG` mirrors all writes to a file.
- Input reassembly across stdin chunks lives in `src/stdin-buffer.ts`; high bytes are converted
  to `ESC + (byte-128)` for legacy Meta compatibility.

## Keys

`src/keys.ts` (1401 lines) is **parsing**, not keybindings. `src/keybindings.ts` (320) is the
configurable layer.

- `KeyId` (`keys.ts:152`) is a type-safe union built recursively from `ModifiedKeyId`, so a typo
  is a compile error. `Key.ctrl("c")` helpers give autocomplete.
- Three input vocabularies: **Kitty CSI-u**, **xterm modifyOtherKeys**, and **legacy** sequences.
- **Non-Latin layouts**: Kitty flag 4 reports a `baseLayoutKey` (PC-101 position), so `Ctrl+С`
  (Cyrillic) matches `ctrl+c`. But the base-layout key is only used when the codepoint is not a
  recognized a-z letter or known symbol — otherwise Dvorak/Colemak/xremap would produce false
  matches (long comment at `:678-685`).
- `LOCK_MASK` (Caps Lock + Num Lock) is masked out of every modifier comparison.
- Ambiguity is resolved by **global protocol state, not platform**: when Kitty is active `\x1b\r`
  means shift+enter; when it is off, `\x1b\r` is alt+enter.
- `isKeyRelease` / `isKeyRepeat` are string-sniffing helpers that first bail if the data contains
  the bracketed-paste marker, so pasted Bluetooth MACs like `90:62:3F:A5` aren't misread.

### Keybindings

`TUI_KEYBINDINGS` (`keybindings.ts:71-210`) is the default table. `KeybindingsManager` provides
`matches`, `getKeys`, and `getConflicts()` (detects two user bindings claiming the same key).
`Keybindings` lists every binding id as `true` so downstream packages can declaration-merge new
ids.

**All components call `getKeybindings().matches(...)`, never `matchesKey` directly.** That is the
extension point, and it is why `AGENTS.md` forbids hardcoding key checks.

Some shipped defaults are deliberately **unbound** (`defaultKeys: []`):
`tui.editor.historyPrevious/Next`, `tui.altScreen.halfPage*`, `lineUp`, `lineDown` — those are
host policy.

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
| darwin | `native/darwin/src/darwin-platform.m` | Objective-C (AppKit) | modifier keys, async text/image clipboard |
| linux | `native/linux/src/linux-platform-x11.c` | C (`libxcb.so.1`) | async X11 clipboard **reads only** |
| win32 | `native/win32/src/win32-platform.c` | C (links `kernel32`, `user32`; no Node headers) | console input setup, modifier keys, async clipboard |

They exist because Node has no API for reading physical modifier-key state, doing asynchronous
clipboard I/O, or enabling Windows VT input. `NativeClipboard` (`src/native-platform.ts:9-16`)
deliberately distinguishes `undefined` (unavailable) from `null` (no text/image), and Linux
"uses command-line tools to retain clipboard ownership instead" of a native write.

Loading (`native-platform.ts:26-51`, `native-module-path.ts:14-31`): x64/arm64 only, tries
several candidate paths because standalone single-file binaries have no installed `pi-tui`
package. `getNativeClipboard()` requires `DISPLAY` on Linux.

Build: `npm run build:native:{darwin,linux,win32}`. Cross-compiling darwin requires osxcross plus
a macOS SDK under Apple's license (`native/darwin/README.md:15-22`).

## Components (`src/components/`, 17 files)

| Component | Notes |
|---|---|
| `Text` | Multi-line; `paddingX=1`/`paddingY=1` defaults; caches by `(text, width)`. Tabs → 3 spaces. |
| `TruncatedText` | Single-line, truncates and pads to exact width. For status lines. |
| `Box` | Padding + background over all children; revalidates its cache by **sampling** `bgFn("test")` rather than tracking function identity. |
| `VStack` / `HStack` | Flexbox-ish `basis`/`grow`/`shrink`/`minSize`/`maxSize`. `VStack` produces an unbounded document. `HStack` composites with `compositeTuiLine`. |
| `ScrollView` | Vertical only (throws otherwise); exactly one child (`addChild`/`clear` throw). `follow: "none" \| "end"`, `overscroll: "chain" \| "contain"`, transient `scrollbar: "auto"`. |
| `Input` | Single-line `Focusable`. Bracketed paste, Emacs kill ring, undo with word-typing coalescing, reverse-video fake cursor. |
| `Editor` | Multi-line `Focusable`; requires the `TUI` for height-aware scrolling. Word wrap, autocomplete, slash-command + file completion, **paste markers**: a `>10`-line paste is merged into one atomic grapheme so cursor motion and deletion treat it as a unit (`editor.ts:48-100`). |
| `Markdown` | `marked` + a 16-slot `MarkdownTheme`, optional `highlightCode`, LaTeX via `renderLatex` with fallback to raw source. |
| `SelectList` | Prefix filter, mouse. The visible window is **centered** on the selection, so hover must not change selection. |
| `SettingsList` | Label/value rows with cycling `values[]` and chainable submenus. Optional fuzzy header. |
| `Image` | Inline via Kitty/iTerm2, `imageFallback` text otherwise. |
| `Loader` / `CancellableLoader` | Braille spinner. `setIndicator` distinguishes "not configured" (colorize) from "configured" (render verbatim). Escape aborts. |
| `MouseRegion` | Adds `handleMouse` without changing rendering. |

Also `src/editor-component.ts` (`EditorComponent` interface for custom editors) and
`src/components/alt-screen-flash.ts` (transient reverse-video messages).

## Notable extras

- `src/latex.ts` (1506 lines) — a **from-scratch LaTeX → Unicode math renderer** with no
  dependency. ~200 symbol tables and a recursive-descent parser that returns `undefined` on any
  unsupported syntax so callers fall back to raw source. Two-pass layout: pass 1 emits private-use
  markers referencing layout nodes, pass 2 composes them into multiple physical lines with a real
  baseline. Fractions get a `───` rule, `cases` uses `⎧⎨⎩`, matrices use the `⎛⎞⎜⎟⎝⎠` families.
  Environments: `equation`, `align*`/`aligned`/`gather`/`multline`/`split`, `cases`, `array`, and
  the `pmatrix` family.
- `src/terminal-image.ts` — graphics protocol plus capability detection. **tmux/screen
  deliberately disable features**: images `null` under tmux, hyperlinks off under `screen`, and
  under tmux hyperlinks only if `tmux display-message -p '#{client_termfeatures}'` actually lists
  them. Unknown terminals get `hyperlinks: false` because OSC 8 renders invisibly as plain text on
  terminals that swallow it (`:129-132`). Header-only dimension parsers for PNG/JPEG/GIF/WebP — no
  image decoding. Overridable with `PI_IMAGE_PROTOCOL`, `PI_TRUE_COLOR`, `PI_HYPERLINKS`.
- `src/autocomplete.ts` — `CombinedAutocompleteProvider` handles slash commands, slash-command
  **arguments**, `@` file mentions, and paths. Fuzzy `@` search shells out to `fd` and `SIGKILL`s
  the child on `AbortSignal`. Smart prefix extraction handles `~/`, `./`, `../`, absolute, and
  **quoted** paths, stripping opening wrappers only when the closer is absent.
- `CURSOR_MARKER = "\x1b_pi:c\x07"` (`tui.ts:168`) — a zero-width APC sequence the focused
  component emits. The TUI strips it and moves the real hardware cursor there for **IME
  candidate-window positioning**, while the component draws a fake inverse-video cursor.
- `SEGMENT_RESET = "\x1b[0m\x1b]8;;\x07"` is appended to every line — SGR *and* OSC-8 hyperlink
  reset.

## Alt-screen specifics

`src/tui-alt-screen.ts` (1455 lines) adds: selection with char/word/line granularity and 500 ms
double-click, auto-scroll at the edges every 50 ms, `copyOnSelect` via OSC 52; the built-in search
panel (top-right, 40% width, binary search over the visible row range); `scrollToPrompt` scanning
for `OSC133_PROMPT_START` markers.

Three deliberate workarounds worth knowing:

- Under TMUX/ZELLIJ/STY it enables button-motion tracking instead of all-motion, *"because
  multiplexers can lag when every pointer movement is forwarded."*
- **iTerm2 images are disabled in alt screen**: capabilities are temporarily overridden so images
  render as text, because the iTerm2 protocol has no delete/crop operation.
- For WezTerm, frames containing images clear all rows before drawing, because WezTerm erases
  Kitty image cells when a later `EL` clears a covered row.

Clipboard honesty (`:1451-1468`): if the host injects `copySelection`, it is used and "Copied!" is
only reported on real success — a bare OSC 52 write can claim success while leaving the clipboard
untouched (macOS Terminal.app, tmux without passthrough).

## Tests

`node --test --test-reporter=dot test/*.test.ts`, importing `src/*.ts` directly. Works because
Node ≥22.19 strips types natively.

Two `Terminal` test doubles:

1. `test/virtual-terminal.ts` — real `@xterm/headless`, always reports
   `kittyProtocolActive === true`, adds test-only `sendInput()` and `resize()`.
2. Hand-rolled terminals when you need to inspect raw bytes — e.g. `BoundedWriteTerminal`
   (`test/tui-render.test.ts:43-63`) records `writes[]`.

`setCapabilities` / `resetCapabilitiesCache` / `setCellDimensions` exist so tests can exercise both
image-protocol paths.

Runnable demos: `test/chat-simple.ts` (130 lines — the smallest complete example),
`test/key-tester.ts`, `test/image-test.ts`, plus `render-churn-bench.ts` and
`alt-screen-large-transcript-bench.ts`.

## Consumer example

Smallest real in-repo usage, `packages/coding-agent/src/cli/config-selector.ts:20-61`:

```ts
const ui: TUI = new TuiMainScreen(new ProcessTerminal(), showHardwareCursor, agentDir);
ui.setClearOnShrink(...);
ui.addChild(selector);
ui.setFocus(selector.getResourceList());
ui.start();
// on close: ui.stop(); resolve();
```

`pi-tui` has 60+ import sites in `packages/coding-agent/src`.
