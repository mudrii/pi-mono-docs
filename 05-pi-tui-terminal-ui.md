# pi-tui — Terminal UI Library

> Status: pi-mono **v0.74.0** (tag `v0.74.0`, commit `1eee081e`). npm: [`@earendil-works/pi-tui`](https://www.npmjs.com/package/@earendil-works/pi-tui) `0.74.0`, published 2026-05-07.

Source: [`packages/tui/`](https://github.com/earendil-works/pi-mono/tree/v0.74.0/packages/tui) in the pi-mono monorepo.

---

## 1. Summary

`pi-tui` is a self-contained TypeScript library for building terminal applications around a **scroll-friendly differential renderer**. Unlike full-screen frameworks (Ink, blessed, ratatui), pi-tui writes into the existing terminal scrollback: new content is appended at the bottom and only the changed tail is repainted on each frame. Overlays float on top of that base content using a separate composition pass.

Distinguishing characteristics for v0.74.0:

- **Zero hard runtime deps on AI/agent packages.** It has only `chalk`, `get-east-asian-width`, `marked`, `mime-types`, plus the optional `koffi` peer used for Windows VT input (`packages/tui/package.json:38-47`).
- **Sub-line differential rendering** with three rebuild strategies: incremental diff (default), width-or-Termux-height change → full redraw, viewport scroll → full redraw (`packages/tui/src/tui.ts:953`).
- **CSI 2026 synchronized output** wraps each frame to eliminate flicker.
- **Kitty keyboard protocol** (flags 1+2+4: disambiguate, event types, alternate keys) with a 150 ms fallback to xterm `modifyOtherKeys` mode 2 (`packages/tui/src/terminal.ts:192-208`).
- **Hardware-cursor IME support** via the bespoke APC marker `\x1b_pi:c\x07` (`packages/tui/src/tui.ts:90`).
- **Inline images** via Kitty and iTerm2 protocols, with image-line tracking so the renderer never partially overwrites a graphic (`packages/tui/src/terminal-image.ts`, `packages/tui/src/components/image.ts`).
- **Anchored overlay stack** with focus restoration and percentage sizing (`packages/tui/src/tui.ts:141-195, 329`).
- **Rich editing surface** — `Editor` is 2 292 lines: word-wrap with paste markers, kill ring, undo stack, character jump, history, slash-command and `@file` autocomplete with `fd` (`packages/tui/src/components/editor.ts`).

The full source surface is small enough to read end-to-end: 11 206 lines across `src/` (counted at v0.74.0).

---

## 2. Install

```bash
npm install @earendil-works/pi-tui
# or
pnpm add @earendil-works/pi-tui
```

Node ≥ 20 is required (`packages/tui/package.json:34`). The package ships as ESM only (`"type": "module"`) and exposes:

```ts
import { TUI, ProcessTerminal, Editor, Markdown, Image, getKeybindings } from "@earendil-works/pi-tui";
```

`koffi` is listed under `optionalDependencies` — install failures are ignored on platforms that don't need it. It is consumed only on Windows to call `kernel32.dll!SetConsoleMode` and enable `ENABLE_VIRTUAL_TERMINAL_INPUT` (`packages/tui/src/terminal.ts:210`).

### Install context: the CLI

`pi-tui` is the rendering layer behind the **`pi-coding-agent`** CLI (`packages/coding-agent`). When you `npm install -g @earendil-works/pi-coding-agent`, pi-tui is pulled in transitively and used to drive the chat UI, slash-command palette, settings dialogs, and inline image rendering. The reference integration is the `chat-simple.ts` test harness (§ 10).

---

## 3. Architecture

### 3.1 Event loop

The TUI runs cooperatively on the Node event loop with one scheduled render per 16 ms (60 fps cap):

```
stdin (raw) ─┐
             ├─► StdinBuffer ─► split sequences ─► input listeners ─► focused component
SIGWINCH ────┘                                                         │
                                                                       ▼
                                                                requestRender()
                                                                       │
                                                                       ▼
                                                            setTimeout(16 ms − Δ)
                                                                       │
                                                                       ▼
                                              compositeOverlays() ──► doRender() ──► terminal.write()
```

Key constants and entry points:

- `MIN_RENDER_INTERVAL_MS = 16` (`packages/tui/src/tui.ts:253`) — minimum gap between renders. `requestRender()` coalesces all calls within the window into a single repaint.
- `start()` puts stdin in raw mode, enables bracketed paste (`\x1b[?2004h`), queries Kitty support, and starts listening (`packages/tui/src/terminal.ts:63+`).
- `drainInput()` temporarily disables the keyboard protocol while resuming a child process, so a shelled-out command sees plain VT input (`packages/tui/src/terminal.ts:233`).
- `onDebug` fires on `Shift+Ctrl+D` before input is dispatched (`packages/tui/src/tui.ts:249`).

### 3.2 Terminal abstraction

`Terminal` is the I/O port (`packages/tui/src/terminal.ts:16-58`):

| Method | Purpose |
|---|---|
| `start()` / `stop()` | enter/leave raw mode, install handlers |
| `drainInput()` | flush pending stdin, disable protocols, return data for child processes |
| `write(data)` | raw write; optionally logged via `PI_TUI_WRITE_LOG` |
| `columns` / `rows` | dimensions, with `COLUMNS`/`LINES` env fallback for headless harnesses |
| `kittyProtocolActive` | live flag set by `queryAndEnableKittyProtocol` |
| `moveBy(dx, dy)` | relative cursor move (clamps at edges) |
| `hideCursor` / `showCursor` | wraps `\x1b[?25l/h` |
| `clearLine` / `clearFromCursor` / `clearScreen` | targeted erasure |
| `setTitle(s)` | OSC 0 |
| `setProgress(state, percent)` | OSC 9;4 — feeds Windows Terminal and ConEmu progress badges |

`ProcessTerminal` is the production implementation that wires `process.stdin`, `process.stdout`, and `SIGWINCH`. Tests use a headless variant from `@xterm/headless`.

### 3.3 Differential rendering pipeline

`doRender()` (`packages/tui/src/tui.ts:953`) implements the diff. Per frame:

1. Compose children + overlays into a flat string array, padded to viewport width and ANSI-reset terminated.
2. Determine the **rebuild strategy**:
   - **First render** (no `previousLines`): full output.
   - **Width changed**, or **height changed on Termux**: full redraw, clear and reprint from the top of the viewport. `fullRedrawCount` increments — exposed via `tui.fullRedraws` for tests/telemetry (`tui.ts:281`).
   - **Viewport top scrolled up** (`firstChanged < viewportTop`): full redraw.
   - Otherwise **incremental diff**: walk from line 0, find the first line whose rendered string differs from `previousLines[i]`, jump the cursor there, and rewrite the tail.
3. If new output is shorter than old (`clearOnShrink`, env `PI_CLEAR_ON_SHRINK=1`), emit clear-to-end-of-line for trailing rows.
4. Wrap the write in `\x1b[?2026h` … `\x1b[?2026l` so the terminal applies it atomically.
5. Update `previousLines`, `previousWidth`, `previousHeight`, `maxLinesRendered`, and Kitty image-id set.
6. If a focused `Focusable` emitted `CURSOR_MARKER`, strip it, then `positionHardwareCursor()` (`tui.ts:1287`) moves the real cursor with `\x1b[<row>;<col>H` so IME candidate popups appear where the user is typing.

The renderer also tracks Kitty image IDs (`extractKittyImageIds`, `tui.ts:16`) so it can issue `\x1b_Gq=1,d=I,i=<id>\x1b\\` deletions for images that scrolled out (`packages/tui/src/terminal-image.ts:177`).

A crash logger writes line-width violations to `~/.pi/agent/pi-crash.log` to catch component bugs in the wild.

---

## 4. Component interface contract

### 4.1 `Component`

```ts
// packages/tui/src/tui.ts:39-63
export interface Component {
  render(width: number): string[];
  handleInput?(data: string): void;
  wantsKeyRelease?: boolean;
  invalidate(): void;
}
```

Contract notes:

- `render(width)` **must** return strings whose visible width (per `visibleWidth`) is ≤ `width`. Violations are logged to the crash log and corrupt the diff.
- `handleInput` only fires when the component has focus (raw or via overlay). `data` is a **single** escape sequence already split out by `StdinBuffer`.
- `wantsKeyRelease` defaults to `false`; the TUI drops Kitty release events for components that don't opt in (`tui.ts:53`, filter in `keys.ts:527`).
- `invalidate()` is called by the framework on theme swaps, focus changes, and full redraws — components clear render caches here.

### 4.2 `Focusable`

```ts
// packages/tui/src/tui.ts:74-77
export interface Focusable {
  focused: boolean;
}
```

When `focused === true`, the component should emit `CURSOR_MARKER` at the logical cursor position in its rendered output. The TUI removes the marker pre-write and uses its coordinates to place the hardware cursor.

`isFocusable()` is a published type guard (`tui.ts:80`).

### 4.3 `Container`

```ts
// packages/tui/src/tui.ts:200-234
export class Container implements Component {
  children: Component[] = [];
  addChild(c: Component): void;
  removeChild(c: Component): void;
  clear(): void;
  invalidate(): void;
  render(width: number): string[];  // concatenates children
}
```

`TUI extends Container` — the root TUI **is** the base layer. Add components with `tui.addChild()`; remove with `tui.removeChild()`.

### 4.4 `TUI`

```ts
// packages/tui/src/tui.ts:239+
class TUI extends Container {
  constructor(terminal: Terminal, showHardwareCursor?: boolean);
  start(): Promise<void>;
  stop(): void;
  requestRender(force?: boolean): void;
  setFocus(c: Component | null): void;
  showOverlay(c: Component, opts?: OverlayOptions): OverlayHandle;
  addInputListener(fn: (data: string) => InputListenerResult): () => void;
  getShowHardwareCursor(): boolean;
  setShowHardwareCursor(enabled: boolean): void;
  getClearOnShrink(): boolean;
  setClearOnShrink(enabled: boolean): void;
  fullRedraws: number;        // read-only counter
  onDebug?: () => void;       // Shift+Ctrl+D
}
```

### 4.5 Lifecycle

```
new ProcessTerminal()
  └─► new TUI(terminal)
        └─► tui.addChild(...) / showOverlay(...)
              └─► await tui.start()    // raw mode, alt protocols, render loop
                    └─► ... event handling ...
                          └─► tui.stop()  // restore stdin, clear protocols
```

`stop()` is idempotent and restores cooked mode, disables bracketed paste, disables Kitty/`modifyOtherKeys`, shows the cursor, and clears any progress badge.

---

## 5. Overlays

Defined in `packages/tui/src/tui.ts:97-195, 329, 623, 758`.

### 5.1 API

```ts
showOverlay(component, {
  // sizing — number or "<n>%"
  width?: SizeValue,
  minWidth?: number,
  maxHeight?: SizeValue,

  // anchor-based positioning
  anchor?:
    | "center" | "top-left" | "top-right" | "bottom-left" | "bottom-right"
    | "top-center" | "bottom-center" | "left-center" | "right-center",
  offsetX?: number,
  offsetY?: number,

  // absolute / percentage positioning (alternative to anchor)
  row?: SizeValue,
  col?: SizeValue,

  // edge margins
  margin?: number | { top?, right?, bottom?, left? },

  // visibility / focus
  visible?: (cols, rows) => boolean,
  nonCapturing?: boolean,
})
  → OverlayHandle { hide(), setHidden(h), isHidden(), focus(), unfocus(), isFocused() }
```

### 5.2 Semantics

- Each `showOverlay` call pushes onto an internal stack with a monotonic `focusOrder` (`tui.ts:264-271`). The topmost **visible, capturing** overlay receives focus.
- `nonCapturing: true` keeps the previous focus target — useful for status bars, completion popups, toasts.
- `visible(cols, rows)` is re-evaluated each frame, so overlays can auto-hide on narrow terminals.
- `resolveOverlayLayout()` (`tui.ts:623`) reduces sizing/positioning into a clipped rect; `compositeOverlays()` (`tui.ts:758`) writes the overlay over the base lines, padded and reset.
- `hide()` permanently removes the overlay and restores the saved `preFocus` (or the next visible overlay below). Use `setHidden(true)` for temporary occlusion (e.g., while showing a child dialog).
- `focus()` / `unfocus()` bring an overlay to the top of the focus stack without re-stacking it.

### 5.3 Common patterns

- **Modal settings dialog:** `tui.showOverlay(new SettingsList(...), { anchor: "center", width: "60%", maxHeight: "80%" })`.
- **Autocomplete popup:** `nonCapturing: true`, `anchor: "bottom-left"`, `offsetY: -1` — the editor keeps focus and forwards Tab/Enter.
- **Confirm before exit:** `tui.showOverlay(confirm, { anchor: "center", width: 40, minWidth: 20 })`; the saved `preFocus` is restored when `hide()` runs.

---

## 6. Editor & input components

### 6.1 `Editor` (multi-line)

`packages/tui/src/components/editor.ts:217` (2 292 lines total).

```ts
new Editor(theme: EditorTheme, options?: EditorOptions)
// EditorOptions { paddingX?, autocompleteMaxVisible? }
// EditorTheme   { borderColor, selectList: SelectListTheme }
```

Features (file-mapped):

- **Word wrap aware of bracketed-paste markers** — `segmentWithMarkers` (line 30) splits text into segments that may not be broken across lines, `wordWrapLine` (line 101) wraps with these constraints. Lets a pasted multi-line string survive resize without reflow corruption.
- **Vertical scrolling** with `maxVisibleLines = max(5, floor(rows * 0.3))` so the editor never eats more than ~30 % of the terminal.
- **Bracketed paste**: any paste > 0 chars is stored as an opaque marker that renders as a summary (`[Pasted #N — 42 lines]`) and can be expanded with `getExpandedText()`.
- **Slash-command autocomplete** and **`@`-prefix file path autocomplete** with 20 ms debounce. File search invokes `fd` via a worker — falls back gracefully if missing.
- **Kill ring** (`packages/tui/src/kill-ring.ts`) — Emacs `Ctrl+K`/`Ctrl+U`/`Ctrl+W` deletes push, `Ctrl+Y` yanks, `Alt+Y` cycles.
- **Undo stack** (`packages/tui/src/undo-stack.ts`) — `Ctrl+_` reverts.
- **Character jump mode** — `Ctrl+]` enters a single-character forward jump, `Ctrl+Alt+]` backward.
- **History** — up to 100 prior submissions, recallable with up/down at the top/bottom of the buffer.
- **Sticky column** for arrow navigation across lines of varying length.

### 6.2 `Input` (single-line)

`packages/tui/src/components/input.ts:18` (503 lines). Same kill ring and undo, horizontal scrolling, bracketed paste, no autocomplete by default but pluggable.

### 6.3 `EditorComponent` interface

`packages/tui/src/editor-component.ts` defines the protocol that lets host CLIs (e.g. pi-coding-agent) swap in alternate editors:

```ts
interface EditorComponent extends Component {
  getText(): string;
  setText(text: string): void;
  handleInput(data: string): void;
  onSubmit: (text: string) => void;
  onChange?: (text: string) => void;

  // optional
  addToHistory?(text: string): void;
  insertTextAtCursor?(text: string): void;
  getExpandedText?(): string;
  setAutocompleteProvider?(p: AutocompleteProvider | null): void;
  borderColor?: (s: string) => string;
  setPaddingX?(n: number): void;
  setAutocompleteMaxVisible?(n: number): void;
}
```

### 6.4 Keybindings

`packages/tui/src/keybindings.ts:54` exports `TUI_KEYBINDINGS`, a layered map keyed by action id. Defaults at v0.74.0 (abbreviated; see source for the full table):

| Action | Default |
|---|---|
| `tui.editor.cursorUp` / `cursorDown` | `up` / `down` |
| `tui.editor.cursorLeft` / `cursorRight` | `[left, ctrl+b]` / `[right, ctrl+f]` |
| `tui.editor.cursorWordLeft` / `cursorWordRight` | `[alt+left, ctrl+left, alt+b]` / `[alt+right, ctrl+right, alt+f]` |
| `tui.editor.cursorLineStart` / `cursorLineEnd` | `[home, ctrl+a]` / `[end, ctrl+e]` |
| `tui.editor.jumpForward` / `jumpBackward` | `ctrl+]` / `ctrl+alt+]` |
| `tui.editor.deleteCharBackward` / `deleteCharForward` | `backspace` / `[delete, ctrl+d]` |
| `tui.editor.deleteWordBackward` / `deleteWordForward` | `[ctrl+w, alt+backspace]` / `[alt+d, alt+delete]` |
| `tui.editor.deleteToLineStart` / `deleteToLineEnd` | `ctrl+u` / `ctrl+k` |
| `tui.editor.yank` / `yankPop` | `ctrl+y` / `alt+y` |
| `tui.editor.undo` | `ctrl+-` |
| `tui.input.newLine` / `submit` / `tab` / `copy` | `shift+enter` / `enter` / `tab` / `ctrl+c` |
| `tui.select.up` / `down` / `pageUp` / `pageDown` / `confirm` / `cancel` | arrows / `enter` / `[escape, ctrl+c]` |

`KeybindingsManager` (`keybindings.ts:155`) supports layered overrides via `getKeybindings()`. Users can rebind any action by merging a partial map.

---

## 7. Keyboard protocol

`packages/tui/src/keys.ts` (1 400 lines) and `packages/tui/src/terminal.ts:192-208`.

### 7.1 Protocol negotiation

On `start()`, `queryAndEnableKittyProtocol` (`terminal.ts:192`) sends `\x1b[?u` and races a 150 ms timer:

- **Kitty response** (`\x1b[?<flags>u`) → enable flags `1 + 2 + 4` via `\x1b[>7u` (disambiguate, event types, alternate keys). `terminal.kittyProtocolActive = true`.
- **Timeout** → enable xterm `modifyOtherKeys` mode 2 with `\x1b[>4;2m`, which encodes Ctrl/Alt combinations as `CSI 27;mod;keycode~`.
- **No protocol** → fall back to legacy VT100 (single byte plus escape-prefix for Alt).

`stop()` always disables both modes so the terminal is left in a sane state.

### 7.2 Encodings handled

`parseKey()` (`keys.ts:1251`) and `parseKittySequence()` (`keys.ts:587`) cover:

- CSI-u: `CSI <keycode> ; <modifiers> [: <event>] [: <alternate_key>] u`
- CSI-tilde with mods: `CSI <keycode> ; <mods> ~`
- xterm `modifyOtherKeys` mode 2: `CSI 27 ; <mods> ; <keycode> ~`
- Legacy arrows `CSI A/B/C/D`, function keys `CSI 11~`…`24~`, SS3 `\x1bO[A-D]`
- Alt-prefix `\x1b<char>` for legacy terminals
- Mouse SGR `CSI < … M/m` (parsed but not exposed by default)
- Kitty keypad codes (`KP_0` … `KP_DIVIDE`) normalized to logical keys

Modifier bitmask: `shift=1, alt=2, ctrl=4, super=8` (Kitty subtracts 1 from the wire value).

### 7.3 Key release / repeat

When `wantsKeyRelease === true`, the TUI passes through `CSI <code>:3u` (release) and `CSI <code>:2u` (repeat). `isKeyRelease` and `isKeyRepeat` (`keys.ts:527, 557`) test those event-type fields. Components that don't opt in never see them.

### 7.4 Layout independence

`matchesKey()` (`keys.ts:820`) honors the **alternate (US-layout) key code** when Kitty supplies one, so `Ctrl+]` works on AZERTY or Greek layouts. Latin letters, digits, and ASCII punctuation are excluded from this remap so native characters keep typing through.

### 7.5 Other input infrastructure

- **Bracketed paste**: enabled via `\x1b[?2004h`; markers `\x1b[200~` … `\x1b[201~`. `StdinBuffer` synthesizes a `paste` event so editors don't see each pasted character as a keypress.
- **`StdinBuffer`** (`packages/tui/src/stdin-buffer.ts:251`, 411 lines): splits batched input into individual sequences, holding incomplete CSI (until terminator `0x40–0x7E`), OSC (until ST or BEL), DCS, APC, and SS3 with a 10 ms timeout. It also deduplicates the case where Kitty emits both a CSI-u sequence and the printable codepoint for one key (fixed in v0.70.3).
- **Windows VT input** (`terminal.ts:210`): uses optional `koffi` to call `SetConsoleMode` with `ENABLE_VIRTUAL_TERMINAL_INPUT` so the same parser works on Windows Terminal.

### 7.6 Public key helpers

```ts
import {
  Key, KeyId, KeyEventType,
  parseKey, matchesKey,
  decodeKittyPrintable,
  isKeyRelease, isKeyRepeat,
  isKittyProtocolActive, setKittyProtocolActive,
} from "@earendil-works/pi-tui";
```

Use `matchesKey(data, "ctrl+]")` for ad-hoc checks, or `parseKey(data)` for full structured access.

---

## 8. Built-in components

All under `packages/tui/src/components/`. Sizes are v0.74.0 line counts.

| Component | File | Notes |
|---|---|---|
| `Text` | `text.ts` (106) | Static styled lines. `setText(string \| string[])`. Caches per-width render. |
| `TruncatedText` | `truncated-text.ts` (65) | Single-line, ellipsis when overflowing. |
| `Spacer` | `spacer.ts` (28) | N empty lines. |
| `Box` | `box.ts` (137) | Bordered container with title and theming; uses Unicode box drawing characters. |
| `Markdown` | `markdown.ts` (852) | CommonMark via `marked` with custom `StrictStrikethroughTokenizer` requiring `~~text~~` with non-whitespace boundaries. Cached by `(text, width)`. ANSI styling, OSC 8 links (skipped on tmux/screen/unknown terminals), code blocks, lists, tables. |
| `SelectList` | `select-list.ts` (229) | Scrolling list with `setFilter` fuzzy match, wraparound on arrow keys, configurable `minPrimaryColumnWidth` / `maxPrimaryColumnWidth`, `truncatePrimary` hook for custom truncation. Items: `{ value, label, description }`. |
| `SettingsList` | `settings-list.ts` (250) | List of `SettingItem { id, label, description, currentValue, values, submenu }`. Enter/Space cycles the current value or opens a submenu. `enableSearch` adds an embedded `Input`. |
| `Loader` | `loader.ts` (86) | Extends `Text`. `setIndicator({ frames, intervalMs })`. `DEFAULT_FRAMES = "⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏"`, `DEFAULT_INTERVAL_MS = 80`. Schedules its own ticks via `setInterval`. |
| `CancellableLoader` | `cancellable-loader.ts` (40) | `Loader` + `AbortController`. Cancel via the `tui.select.cancel` keybinding. |
| `Editor` | `editor.ts` (2 292) | See § 6.1. |
| `Input` | `input.ts` (503) | See § 6.2. |
| `Image` | `image.ts` (112) | See § 9. |

Top-level re-exports — see `packages/tui/src/index.ts` (106 lines).

### Utility modules

- **`utils.ts`** (1 140 lines): `visibleWidth` (East Asian width + flag handling, ASCII fast path, 512-entry cache), `truncateToWidth`, `wrapTextWithAnsi` (preserves SGR state and re-opens OSC 8 hyperlinks at line breaks), `sliceByColumn`, `sliceWithWidth`, `extractSegments`, `normalizeTerminalOutput`.
- **`fuzzy.ts`** (137 lines): `fuzzyMatch`, `fuzzyFilter`, `FuzzyMatch` — the ranker driving SelectList filters and slash-command suggestions. v0.73.0 updated ranking heuristics.
- **`autocomplete.ts`** (783 lines): `AutocompleteProvider` interface, `CombinedAutocompleteProvider` (stacks multiple sources), `SlashCommand` provider, `AutocompleteItem`, `AutocompleteSuggestions`.

---

## 9. Image support

`packages/tui/src/terminal-image.ts` (423 lines) and `packages/tui/src/components/image.ts` (112 lines).

### 9.1 Capability detection

`detectCapabilities()` (`terminal-image.ts:42`) reads env vars:

| Env var | Result |
|---|---|
| `KITTY_WINDOW_ID` | `images = "kitty"` |
| `TERM_PROGRAM = ghostty` | `images = "kitty"` |
| `WEZTERM_PANE` | `images = "kitty"` |
| `ITERM_SESSION_ID` (and not WezTerm) | `images = "iterm2"` |
| Inside tmux/screen | `images = null`, `hyperlinks = false` |
| Inside `cmux` | images disabled (v0.73.1 fix) |

The capability struct also exposes `cellWidthPx` / `cellHeightPx`, queried with `\x1b[16t` and cached.

### 9.2 Encoding

- **Kitty**: `encodeKitty()` (`terminal-image.ts:127`) chunks base64 at 4 096 chars and emits `\x1b_Ga=T,f=100,i=<id>,m=1;…\x1b\\`. Image IDs are random 32-bit ints from `allocateImageId()` to avoid collision with other Kitty consumers.
- **iTerm2**: `encodeITerm2()` (line 189) emits `\x1b]1337;File=…;inline=1:<base64>\x07`.

`renderImage()` returns `{ sequence, rows, cols, imageId? }`. `calculateImageRows()` (line 214) converts pixel dimensions to terminal cells using the cell-size query.

Image dimensions are parsed natively for **PNG**, **JPEG**, **GIF**, and **WebP** (no external deps).

### 9.3 The `Image` component

`packages/tui/src/components/image.ts:23`:

```ts
new Image(base64Data, mimeType, theme, options?, dimensions?)
// ImageOptions { maxWidthCells?, maxHeightCells?, filename?, imageId? }
// ImageTheme   { fallbackColor }
```

Render strategy (`image.ts:59`):

- If the terminal supports images, allocate a Kitty image id on first render, then emit `(rows-1)` empty lines, a cursor-up `CSI <n>A`, the Kitty/iTerm2 sequence, then for Kitty a cursor-down `CSI <n>B`. This keeps the TUI's logical cursor model in lockstep with the visible content while the terminal places the graphic.
- If no protocol is available, render a `[image: image/png 800×600 filename.png]` text fallback colored via `theme.fallbackColor`.
- `invalidate()` clears the cached width/lines; the TUI calls this when the viewport resizes.
- `deleteKittyImage(id)` (`terminal-image.ts:177`) is issued automatically when an image scrolls out of the diff window.

v0.73.1 fixed a redraw bug where shrinking the window left ghost images on screen.

---

## 10. Examples

### 10.1 Reference harness — `test/chat-simple.ts`

This 130-line test fixture is the canonical "minimum working chat UI" and the closest pi-tui has to a runnable example:

```ts
import { Container, Editor, Loader, Markdown, ProcessTerminal, TUI,
         CombinedAutocompleteProvider, SlashCommand } from "@earendil-works/pi-tui";

const terminal = new ProcessTerminal();
const tui = new TUI(terminal);

const messages = new Container();
tui.addChild(messages);

const editor = new Editor(
  { borderColor: (s) => s, selectList: theme.selectList },
  { paddingX: 1 },
);
editor.setAutocompleteProvider(new CombinedAutocompleteProvider([
  new SlashCommand([
    { name: "/clear", description: "Clear chat" },
    { name: "/delete", description: "Delete last message" },
  ]),
]));
editor.onSubmit = (text) => {
  messages.addChild(new Markdown(text, theme.markdown));
  // ... show Loader, call LLM, replace with response Markdown
  tui.requestRender();
};
tui.addChild(editor);
tui.setFocus(editor);

await tui.start();
```

See `packages/tui/test/chat-simple.ts` for the full version with the spinner swap-out pattern.

### 10.2 Modal overlay

```ts
const dialog = new SettingsList(items, theme.settings, { enableSearch: true });
const handle = tui.showOverlay(dialog, {
  anchor: "center",
  width: "60%",
  maxHeight: "70%",
});
dialog.onConfirm = () => handle.hide();   // restores prior focus
```

### 10.3 Inline image

```ts
import { readFile } from "node:fs/promises";
import { Image } from "@earendil-works/pi-tui";

const data = await readFile("logo.png");
const img = new Image(
  data.toString("base64"),
  "image/png",
  { fallbackColor: (s) => s },
  { maxWidthCells: 40, filename: "logo.png" },
);
tui.addChild(img);
tui.requestRender();
```

### 10.4 Custom keybinding override

```ts
import { KeybindingsManager, TUI_KEYBINDINGS } from "@earendil-works/pi-tui";

const mgr = new KeybindingsManager(TUI_KEYBINDINGS);
mgr.override({ "tui.editor.cursorWordLeft": ["alt+left"] });   // drop ctrl+left
const editor = new Editor(theme, { keybindings: mgr });
```

---

## 11. DeepWiki discrepancies

The DeepWiki snapshot at `https://deepwiki.com/badlogic/pi-mono` (last indexed 30 April 2026, commit `156a9052`) predates the pi-mono ownership move and v0.74.0. Material differences to be aware of:

| Topic | DeepWiki | v0.74.0 reality |
|---|---|---|
| **npm scope** | `@mariozechner/pi-tui` | `@earendil-works/pi-tui` — the old scope is no longer published. |
| **GitHub org** | `github.com/badlogic/pi-mono` | `github.com/earendil-works/pi-mono` (see `packages/tui/package.json:30`). |
| **`tui.ts` line numbers** | references `tui.ts:17-41`, `:68`, `:214-236`, `:404-416` | v0.74.0: `Component` interface `tui.ts:39-63`, `CURSOR_MARKER` at `:90`, `TUI` class at `:239`, `setFocus()` at `:311`. Use this doc's citations. |
| **Differential rendering** | Mentions `previousLines` + content-shrink clear | Also covers width change, Termux height change, viewport-top scroll → full redraw (`tui.ts:953`), plus Kitty image-id tracking. |
| **Keyboard protocol** | Lightly described | v0.74.0: Kitty flags 1+2+4, 150 ms timeout, `modifyOtherKeys` mode 2 fallback, layout-aware `matchesKey` (see § 7). |
| **Built-in components** | Lists Editor, Input, SelectList, Container, Markdown, Image | v0.74.0 ships those plus `Text`, `TruncatedText`, `Spacer`, `Box`, `SettingsList`, `Loader`, `CancellableLoader` (see § 8). |
| **Overlay system** | DeepWiki page 5.3 returns no content at fetch time | Fully documented in § 5 here. |
| **DeepWiki page 5.4** | Returns content about keyboard protocols, not components | Treat as misindexed. § 8 here is the authoritative component inventory. |
| **DeepWiki utilities page** | Doesn't mention `fuzzy` or `autocomplete` modules | Both ship in v0.74.0 — `packages/tui/src/fuzzy.ts`, `packages/tui/src/autocomplete.ts`. |

When DeepWiki and this doc conflict, **trust the source under `packages/tui/` at tag `v0.74.0`**; DeepWiki is a stale auto-extracted summary.

---

## 12. References

### Source files (pi-mono v0.74.0)

- `packages/tui/package.json` — manifest, dependencies, Node version requirement
- `packages/tui/src/index.ts` — public API surface
- `packages/tui/src/tui.ts` — `Component`, `Focusable`, `Container`, `TUI`, overlay system, render pipeline
- `packages/tui/src/terminal.ts` — `Terminal`, `ProcessTerminal`, protocol negotiation, OSC 9;4 progress
- `packages/tui/src/keys.ts` — key parsing, Kitty / CSI-u / `modifyOtherKeys` decoders, `matchesKey`
- `packages/tui/src/keybindings.ts` — `TUI_KEYBINDINGS`, `KeybindingsManager`
- `packages/tui/src/stdin-buffer.ts` — input chunking, bracketed paste, dedupe
- `packages/tui/src/terminal-image.ts` — Kitty / iTerm2 image protocols, PNG/JPEG/GIF/WebP dimension parsing
- `packages/tui/src/utils.ts` — `visibleWidth`, `truncateToWidth`, `wrapTextWithAnsi`, slicing helpers
- `packages/tui/src/fuzzy.ts` — fuzzy match used by SelectList and autocomplete
- `packages/tui/src/autocomplete.ts` — autocomplete providers and `SlashCommand`
- `packages/tui/src/kill-ring.ts`, `undo-stack.ts` — Emacs-style editing helpers
- `packages/tui/src/editor-component.ts` — pluggable editor interface
- `packages/tui/src/components/*.ts` — built-in components
- `packages/tui/test/chat-simple.ts` — reference chat harness
- `packages/tui/README.md` — upstream readme (780 lines)
- `packages/tui/CHANGELOG.md` — historical changes

### External

- npm: https://www.npmjs.com/package/@earendil-works/pi-tui (v0.74.0)
- Repo: https://github.com/earendil-works/pi-mono/tree/v0.74.0/packages/tui
- Marked parser: https://github.com/markedjs/marked (v15)
- Kitty keyboard protocol: https://sw.kovidgoyal.net/kitty/keyboard-protocol/
- Kitty graphics protocol: https://sw.kovidgoyal.net/kitty/graphics-protocol/
- iTerm2 image protocol: https://iterm2.com/documentation-images.html
- CSI 2026 synchronized output: https://gist.github.com/christianparpart/d8a62cc1ab659194337d73e399004036
- OSC 8 hyperlinks: https://gist.github.com/egmontkob/eb114294efbcd5adb1944c9f3cb5feda
- OSC 9;4 progress: ConEmu / Windows Terminal extension
- xterm `modifyOtherKeys`: https://invisible-island.net/xterm/modified-keys.html
