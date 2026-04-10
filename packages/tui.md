# @mariozechner/pi-tui

**Version:** 0.66.1 · **License:** MIT · [npm](https://www.npmjs.com/package/@mariozechner/pi-tui)

Low-level terminal UI toolkit with differential rendering, Kitty graphics protocol, IME support, and a rich component library.

---

## Installation

```bash
npm install @mariozechner/pi-tui
```

**Peer dependencies:** None required.

**Optional:**
```bash
npm install koffi  # FFI bindings for native features
```

---

## Quick Start

```typescript
import { TUI, Text, Input, Box } from "@mariozechner/pi-tui";

const tui = new TUI();

const input = new Input({
  onSubmit: (value) => {
    tui.stop();
    console.log("You entered:", value);
  }
});

const layout = new Box({ padding: 1, children: [
  new Text({ text: "Type something and press Enter:" }),
  input
]});

tui.setRoot(layout);
tui.setFocus(input);
tui.start();
```

---

## TUI Class

The main class that owns the terminal, render loop, and overlay stack.

```typescript
class TUI extends Container {
  terminal: Terminal;

  // Focus management
  setFocus(component: Component | null): void;

  // Overlay management
  showOverlay(component: Component, options?: OverlayOptions): OverlayHandle;
  hideOverlay(): void;
  hasOverlay(): boolean;

  // Cursor (IME positioning)
  setShowHardwareCursor(enabled: boolean): void;

  // Debug hook
  onDebug?: () => void;  // Triggered by Shift+Ctrl+D

  // Render control
  requestRender(force?: boolean): void;

  // Lifecycle
  start(): void;
  stop(): void;

  // Input listeners
  addInputListener(listener: InputListener): () => void;
}
```

---

## Differential Rendering

Pi-tui uses **differential rendering** — only changed lines are written to the terminal, minimizing flicker and reducing I/O.

### Full Redraw Triggers

A full redraw happens when:
- First render
- Terminal dimensions change (resize)
- Content shrinks below the working area (configure via `PI_CLEAR_ON_SHRINK`)
- Content above the viewport has changed

### How Differential Updates Work

1. Previous render lines are cached
2. On each render cycle, new lines are computed
3. First and last changed lines are detected
4. Only lines in the changed range are written
5. Synchronized output mode (`\x1b[?2026h` / `\x1b[?2026l`) prevents flicker

Single-line changes (e.g., spinner animation) write a single line — minimal terminal I/O.

### Debug Environment Variables

| Variable | Effect |
|----------|--------|
| `PI_DEBUG_REDRAW=1` | Log full redraw reasons to `~/.pi/agent/pi-debug.log` |
| `PI_TUI_DEBUG=1` | Detailed render logs to `/tmp/tui/` |
| `PI_TUI_WRITE_LOG=<path>` | Directory path: creates `tui-<timestamp>-<pid>.log` per instance (v0.63.0+) |
| `PI_HARDWARE_CURSOR=1` | Show hardware cursor (for IME) |
| `PI_CLEAR_ON_SHRINK=0\|1` | Control screen-clear on content shrink |

### Container.render() Stack Overflow Fix (unreleased, on main)

A stack overflow in `Container.render()` caused by deeply nested component trees has been fixed on main but is not yet part of a release.

### Render Scheduling (v0.66.1+)

`requestRender()` coalesces to a 16ms frame budget under streaming load. `requestRender(true)` renders immediately.

### PI_TUI_WRITE_LOG (v0.63.0+)

Set to a directory path. Each instance creates `tui-<timestamp>-<pid>.log` for debugging multiple concurrent pi sessions.

### Keyboard Fixes (v0.64.0)

- **Kitty keypad normalization**: Kitty keyboard protocol keypad functional keys now normalize to logical digits, symbols, and navigation keys, fixing numpad input in terminals such as iTerm2.
- **CSI cell size parsing**: Cell size response handling now consumes only exact `CSI 6 ; height ; width t` replies, so bare `Escape` is no longer swallowed while waiting for terminal image metadata.

---

## Component System

All UI elements implement `Component`:

```typescript
interface Component {
  render(width: number): string[];   // Return rendered lines (ANSI ok)
  handleInput?(data: string): void;  // Called when focused
  wantsKeyRelease?: boolean;         // Opt in to key release events
  invalidate(): void;                // Signal re-render needed
}

interface Focusable {
  focused: boolean;
}
```

Components are composed into a tree. The TUI calls `render()` on the root and diffs the output.

---

## Built-in Components

### Container

Base layout component that stacks children vertically.

```typescript
new Container({ children: Component[] })
```

### Box

Container with optional padding and background color.

```typescript
new Box({
  padding?: number | { top, right, bottom, left },
  background?: string,   // ANSI color/chalk color
  children: Component[]
})
```

### Text

Multi-line text with optional word wrapping and padding.

```typescript
new Text({
  text: string,
  wordWrap?: boolean,
  padding?: number,
  color?: string
})
```

### Input

Single-line text input with editing features.

```typescript
new Input({
  value?: string,
  placeholder?: string,
  onSubmit?: (value: string) => void,
  onChange?: (value: string) => void,
  autocomplete?: AutocompleteProvider
})
```

**Features:**
- Horizontal scrolling for long text
- Emacs-style kill ring (`Ctrl+W`, `Alt+D`, `Ctrl+Y`, `Ctrl+K`)
- Undo/redo support
- Bracketed paste handling
- Autocomplete overlay integration

### Editor

Multi-line code/text editor.

```typescript
new Editor({
  value?: string,
  onSubmit?: (value: string) => void,
  onChange?: (value: string) => void,
  highlightCode?: (code: string, lang: string) => string,
  autocomplete?: AutocompleteProvider,
  keybindings?: EditorKeybindingsConfig
})
```

**Features:**
- Syntax highlighting via callback
- Word wrapping with paste marker awareness
- Kill ring and undo
- Autocomplete overlay
- Configurable keybindings (Emacs-style by default)

### SelectList

Scrollable selection list with optional filtering.

```typescript
new SelectList({
  items: SelectItem[],
  onSelect?: (item: SelectItem) => void,
  onCancel?: () => void,
  filter?: string,
  primaryColumnWidth?: number,
  theme?: SelectListTheme
})

interface SelectItem {
  value: string;
  label: string;
  description?: string;
}
```

### Markdown

Render Markdown to terminal output.

```typescript
new Markdown({
  text: string,
  padding?: number,
  theme?: MarkdownTheme
})
```

Supports code blocks with syntax highlighting, bold/italic, lists, blockquotes, and horizontal rules.

### Image

Display images inline using terminal graphics protocols.

```typescript
new Image({
  path?: string,        // File path
  data?: string,        // Base64 data
  mimeType?: string,
  maxWidth?: number,
  maxHeight?: number
})
```

Supports Kitty graphics protocol (Kitty, Ghostty, WezTerm) and iTerm2 inline images. Falls back to unicode block art.

### Loader / CancellableLoader

Spinner animation for long-running operations.

```typescript
new Loader({ label?: string })

new CancellableLoader({
  label?: string,
  onCancel: () => void
})
```

### TruncatedText

Text that truncates with ellipsis to fit width.

```typescript
new TruncatedText({ text: string, suffix?: string })
```

### Spacer

Adds vertical whitespace.

```typescript
new Spacer({ lines: number })
```

---

## Overlay System

Overlays render on top of the main UI, optionally capturing all input.

```typescript
type OverlayAnchor =
  | "center"
  | "top-left" | "top-right" | "bottom-left" | "bottom-right"
  | "top-center" | "bottom-center"
  | "left-center" | "right-center";

interface OverlayOptions {
  width?: number | `${number}%`;      // Absolute columns or percent of terminal
  minWidth?: number;
  maxHeight?: number | `${number}%`;
  anchor?: OverlayAnchor;             // Default: "center"
  offsetX?: number;
  offsetY?: number;
  row?: number | `${number}%`;        // Override row position
  col?: number | `${number}%`;        // Override col position
  margin?: number | OverlayMargin;
  visible?: (width: number, height: number) => boolean;
  nonCapturing?: boolean;             // Don't steal focus (informational overlays)
}

interface OverlayHandle {
  hide(): void;
  setHidden(hidden: boolean): void;
  isHidden(): boolean;
  focus(): void;
  unfocus(): void;
  isFocused(): boolean;
}
```

**Example — modal dialog:**

```typescript
const dialog = new Box({
  padding: 2,
  children: [
    new Text({ text: "Confirm action?" }),
    new SelectList({
      items: [{ value: "yes", label: "Yes" }, { value: "no", label: "No" }],
      onSelect: (item) => {
        handle.hide();
        if (item.value === "yes") doAction();
      }
    })
  ]
});

const handle = tui.showOverlay(dialog, {
  anchor: "center",
  width: "40%"
});
```

**Non-capturing overlay** (status bar, doesn't block input):

```typescript
tui.showOverlay(statusComponent, { nonCapturing: true, anchor: "bottom-center" });
```

---

## Key Handling

### Kitty Keyboard Protocol

Pi-tui supports the **Kitty keyboard protocol** for enhanced key event reporting:
- Distinguishes key press, repeat, and release
- Supports modifier combinations not available in legacy VT
- Enabled automatically when the terminal reports support

```typescript
isKittyProtocolActive(): boolean;
setKittyProtocolActive(active: boolean): void;
```

### Key Identifiers

```typescript
type KeyId =
  // Letters: "a" through "z"
  | "a" | "b" | ... | "z"
  // Digits: "0" through "9"
  | "0" | "1" | ... | "9"
  // Symbols: "`", "-", "=", "[", "]", "\\", ";", "'", ",", ".", "/"
  | SymbolKey
  // Special keys
  | "escape" | "enter" | "tab" | "backspace"
  | "home" | "end" | "pageup" | "pagedown"
  | "insert" | "delete"
  | "up" | "down" | "left" | "right"
  // Function keys
  | "f1" | ... | "f24"
  // Modified: e.g., "ctrl+a", "alt+b", "shift+enter", "ctrl+shift+d"
  | `${"ctrl+" | "alt+" | "shift+" | "ctrl+shift+" | ...}${KeyId}`;
```

**Key utilities:**

```typescript
matchesKey(data: string, keyId: KeyId): boolean;
parseKey(data: string): KeyId | null;
isKeyRelease(data: string): boolean;
isKeyRepeat(data: string): boolean;
decodeKittyPrintable(data: string): string;
```

**Example — global Escape handler:**

```typescript
tui.addInputListener((data) => {
  if (matchesKey(data, "escape")) {
    tui.stop();
    return true;  // Consume the event
  }
  return false;
});
```

### Keybindings Configuration

The editor uses configurable Emacs-style keybindings:

```typescript
type EditorAction =
  | "cursorUp" | "cursorDown" | "cursorLeft" | "cursorRight"
  | "cursorWordLeft" | "cursorWordRight"
  | "cursorLineStart" | "cursorLineEnd"
  | "deleteCharBackward" | "deleteCharForward"
  | "deleteWordBackward" | "deleteWordForward"
  | "deleteToLineStart" | "deleteToLineEnd"
  | "newLine" | "submit" | "tab"
  | "selectUp" | "selectDown" | "selectPageUp" | "selectPageDown"
  | "selectConfirm" | "selectCancel"
  | "copy" | "yank" | "yankPop" | "undo"
  | "expandTools" | "treeFoldOrUp" | "treeUnfoldOrDown"
  | "toggleSessionPath" | "toggleSessionSort"
  | "renameSession" | "deleteSession";

interface EditorKeybindingsConfig {
  [action: string]: KeyId | KeyId[];
}

// Access/modify global keybindings
const bindings = getEditorKeybindings();
setEditorKeybindings({ submit: ["enter", "ctrl+enter"] });
```

---

## Autocomplete System

```typescript
interface AutocompleteItem {
  value: string;
  label: string;
  description?: string;
}

interface AutocompleteProvider {
  provide(partial: string): Promise<AutocompleteItem[]>;
}

// Combine multiple providers
class CombinedAutocompleteProvider implements AutocompleteProvider {
  constructor(providers: AutocompleteProvider[]);
}
```

**Built-in provider: path completion**

Pi-tui ships a file path autocomplete provider that uses `fd` for fast file discovery:

```typescript
import { createPathAutocompleteProvider } from "@mariozechner/pi-tui";

const pathProvider = createPathAutocompleteProvider({
  cwd: process.cwd(),
  includeHidden: false
});

const input = new Input({ autocomplete: pathProvider });
```

---

## IME Support

Pi-tui positions the hardware cursor precisely for **Input Method Editor** candidate windows:

1. A focused component emits `CURSOR_MARKER` (`\x1b_pi:c\x07`) at the cursor position in its output
2. The TUI extracts the marker, determines its row/col
3. Moves the hardware cursor there using ANSI positioning
4. The OS IME picks up the cursor position for candidate window placement

Enable with:
```bash
PI_HARDWARE_CURSOR=1 pi
```

Or programmatically:
```typescript
tui.setShowHardwareCursor(true);
```

---

## Terminal Interface

```typescript
interface Terminal {
  columns: number;
  rows: number;
  kittyProtocolActive: boolean;

  start(onInput: (data: string) => void, onResize: () => void): void;
  stop(): void;
  drainInput(maxMs?: number, idleMs?: number): Promise<void>;

  write(data: string): void;
  moveBy(lines: number): void;
  hideCursor(): void;
  showCursor(): void;
  clearLine(): void;
  clearFromCursor(): void;
  clearScreen(): void;
  setTitle(title: string): void;
}

// Default implementation using process.stdin/stdout
class ProcessTerminal implements Terminal { ... }
```

For testing, implement the interface with a headless terminal (xterm/headless is used in pi-tui's own tests).

---

## Terminal Image Support

### Capability Detection

```typescript
interface TerminalCapabilities {
  images: "kitty" | "iterm2" | null;
  trueColor: boolean;
  hyperlinks: boolean;
}

// Detected from environment automatically:
// KITTY_WINDOW_ID, TERM_PROGRAM=kitty  → Kitty protocol
// GHOSTTY_RESOURCES_DIR, TERM=ghostty  → Kitty protocol
// WEZTERM_PANE                         → Kitty protocol
// ITERM_SESSION_ID                     → iTerm2 protocol
```

### Image Rendering API

```typescript
// High-level: auto-detect protocol, render file
async function renderImage(
  filePath: string,
  options?: ImageRenderOptions
): Promise<string>;   // Returns terminal escape sequence string

// Low-level: Kitty protocol
function encodeKitty(
  base64Data: string,
  options?: { columns?: number; rows?: number; imageId?: number }
): string;

function deleteKittyImage(imageId: number): string;
function deleteAllKittyImages(): string;
function allocateImageId(): number;

// Low-level: iTerm2 protocol
function encodeITerm2(
  base64Data: string,
  fileName: string,
  inline?: boolean
): string;
```

### Image Dimension Detection

```typescript
interface ImageDimensions { widthPx: number; heightPx: number; }

getImageDimensions(buffer: Buffer): Promise<ImageDimensions>;
getPngDimensions(buffer: Buffer): ImageDimensions;
getJpegDimensions(buffer: Buffer): ImageDimensions;
getWebpDimensions(buffer: Buffer): ImageDimensions;
getGifDimensions(buffer: Buffer): ImageDimensions;
```

---

## Text Utilities

```typescript
// Terminal display width (accounts for East Asian double-width characters)
visibleWidth(text: string): number;

// Truncate to terminal column width
truncateToWidth(text: string, width: number, suffix?: string): string;

// Word-wrap text respecting ANSI sequences
wrapTextWithAnsi(text: string, maxWidth: number): string[];

// Slice by display column (respects double-width chars and ANSI)
sliceByColumn(text: string, start: number, end: number, strict?: boolean): string;
```

---

## Supported Terminals

| Terminal | Images | Kitty Keys | True Color |
|----------|--------|------------|------------|
| Kitty | Kitty protocol | Yes | Yes |
| Ghostty | Kitty protocol | Yes | Yes |
| WezTerm | Kitty protocol | Yes | Yes |
| iTerm2 | iTerm2 protocol | Partial | Yes |
| Windows Terminal | None | Partial | Yes |
| xterm-256color | None | No | Yes |
| tmux | None | Partial (with config) | Yes |

**tmux note:** Extended key handling requires `set -g extended-keys on` in `~/.tmux.conf`.

**xfce4-terminal / Terminator note:** These terminals do not support Kitty keyboard protocol. Pi detects this and disables extended key reporting.
