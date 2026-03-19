# pi-tui: Terminal User Interface Framework

Package: `@mariozechner/pi-tui` (v0.60.0)
License: MIT
Source: `packages/tui/`

pi-tui is a minimal terminal UI framework with differential rendering and synchronized output for flicker-free interactive CLI applications. It provides the visual layer for the pi coding agent's terminal interface.

## Dependencies

- `chalk` -- ANSI color styling
- `marked` -- Markdown parsing for the Markdown component
- `get-east-asian-width` -- Correct width calculation for CJK/fullwidth characters
- `mime-types` -- MIME type detection for image rendering
- `koffi` (optional) -- Windows VT input support via Win32 console API

Dev dependencies include `@xterm/headless` for `VirtualTerminal` testing.

## Architecture Overview

pi-tui is built around three core abstractions:

1. **Terminal** -- An interface abstracting raw terminal I/O (stdin/stdout, cursor movement, screen clearing)
2. **Component** -- A render unit that produces lines of text for a given width
3. **TUI** -- The main container that orchestrates components, manages focus, runs differential rendering, and composites overlays

The rendering pipeline:

```
Components render(width) -> lines[] -> overlay compositing -> differential compare -> ANSI output
```

All output is wrapped in CSI 2026 synchronized output sequences for atomic, flicker-free screen updates.

## Terminal Abstraction

The `Terminal` interface (`src/terminal.ts`) defines the contract for terminal I/O:

```typescript
interface Terminal {
  start(onInput: (data: string) => void, onResize: () => void): void;
  stop(): void;
  drainInput(maxMs?: number, idleMs?: number): Promise<void>;
  write(data: string): void;
  get columns(): number;
  get rows(): number;
  get kittyProtocolActive(): boolean;
  moveBy(lines: number): void;
  hideCursor(): void;
  showCursor(): void;
  clearLine(): void;
  clearFromCursor(): void;
  clearScreen(): void;
  setTitle(title: string): void;
}
```

### ProcessTerminal

The production implementation wraps `process.stdin`/`process.stdout`. On startup it:

1. Enables raw mode on stdin
2. Enables bracketed paste mode (`CSI ?2004h`) so paste content is wrapped in markers
3. On Windows, calls `SetConsoleMode` via koffi/kernel32.dll to enable `ENABLE_VIRTUAL_TERMINAL_INPUT` (0x0200), which makes the console emit VT escape sequences for modified keys
4. Queries the terminal for Kitty keyboard protocol support (`CSI ?u`). If the terminal responds, Kitty protocol flags 1+2+4 are enabled (disambiguate, report event types, report alternate keys). Otherwise, xterm `modifyOtherKeys` mode 2 is enabled as a fallback (needed for tmux)
5. Sets up `StdinBuffer` to split batched stdin into individual sequences, ensuring each component receives single key events

### StdinBuffer

`StdinBuffer` (`src/stdin-buffer.ts`) solves a critical problem: over SSH or in high-latency terminals, multiple key events may arrive in a single stdin data event. StdinBuffer parses and splits these into individual escape sequences with a configurable timeout (default 10ms) for incomplete sequences. It also detects and re-wraps bracketed paste content.

### drainInput

The `drainInput()` method addresses a specific issue with the Kitty keyboard protocol: when the agent exits, key release events may still be in transit (especially over slow SSH connections). Without draining, these leak to the parent shell. The method disables the keyboard protocol, then reads and discards stdin for up to 1 second (or 50ms of idle time).

### VirtualTerminal

A testing implementation using `@xterm/headless` for headless terminal emulation.

## Component System

### Component Interface

Every visual element implements:

```typescript
interface Component {
  render(width: number): string[];
  handleInput?(data: string): void;
  wantsKeyRelease?: boolean;
  invalidate(): void;
}
```

- `render(width)` returns an array of strings, one per line. Each line must not exceed `width` visible characters or the TUI throws an error (with a crash log written to `~/.pi/agent/pi-crash.log`)
- `handleInput(data)` is called when the component has focus and receives keyboard input
- `wantsKeyRelease` opts in to Kitty protocol key release events (default: filtered out)
- `invalidate()` clears cached render state, forcing a fresh render on the next cycle

### Focusable Interface

Components that display a text cursor and need IME support implement:

```typescript
interface Focusable {
  focused: boolean;
}
```

When focused, a component emits `CURSOR_MARKER` (an APC escape sequence `ESC _ pi:c BEL`) at the cursor position. TUI finds this zero-width marker, strips it, and positions the hardware terminal cursor there. This enables IME candidate windows (for CJK input) to appear at the correct location.

Container components with embedded inputs must propagate the `focused` property to their child input component, or IME candidate windows will appear in the wrong position.

### Container

The base grouping component. `TUI` itself extends `Container`. Children are rendered sequentially, their lines concatenated.

```typescript
class Container implements Component {
  children: Component[] = [];
  addChild(component: Component): void;
  removeChild(component: Component): void;
  clear(): void;
}
```

## Built-in Components

### Text and TruncatedText

- `Text` -- Multi-line text with word wrapping and configurable padding/background. Uses `wrapTextWithAnsi()` to preserve ANSI codes across line breaks.
- `TruncatedText` -- Single-line text that truncates to fit the viewport width. Useful for status lines and headers.

### Input

Single-line text input with horizontal scrolling. Supports full Unicode input including CJK characters. Key bindings follow readline conventions:

- `Ctrl+A`/`Ctrl+E` -- line start/end
- `Ctrl+W` / `Alt+Backspace` -- delete word backwards
- `Ctrl+K` / `Ctrl+U` -- kill to end/start of line
- `Ctrl+Y` / `Alt+Y` -- yank / yank-pop (kill ring)
- `Ctrl+Left`/`Ctrl+Right`, `Alt+Left`/`Alt+Right` -- word navigation
- Arrow keys, Backspace, Delete

The Input component handles wide characters correctly by using visual column width for scroll offset calculations and strict slice boundaries to prevent splitting emoji or CJK characters.

### Editor

The multi-line text editor (`src/components/editor.ts`) is the primary input component for the coding agent. It requires a `TUI` reference for height-aware vertical scrolling.

Key features:

- **Word wrapping** -- Content wraps at word boundaries with correct handling of wide characters at wrap boundaries
- **Vertical scrolling** -- When content exceeds 30% of terminal height (minimum 5 lines), the editor scrolls vertically with indicators showing lines above/below the viewport
- **Paste handling** -- Bracketed paste mode detects pastes. Pastes over 10 lines create a `[paste #N +M lines]` marker that can be expanded later via `getExpandedText()`
- **Autocomplete** -- Slash command completion (type `/`) with fuzzy matching, and file path completion (press `Tab`) with directory traversal
- **Undo stack** -- Ctrl+- undo, coalescing consecutive word characters into one unit (fish-style)
- **Kill ring** -- Emacs-style kill ring with Ctrl+K (kill), Ctrl+Y (yank), Alt+Y (yank-pop)
- **Character jump** -- Ctrl+] jumps forward to next occurrence of a character, Ctrl+Alt+] jumps backward
- **Sticky column tracking** -- Vertical cursor navigation restores the preferred column when moving across short lines
- **History** -- Up/Down navigation through previously submitted texts

The editor renders horizontal border lines above and below its content. It uses a fake cursor (reverse video block) rather than the real terminal cursor, with the real cursor positioned via `CURSOR_MARKER` for IME support.

Submit behavior: Enter submits, while Shift+Enter, Ctrl+Enter, or Alt+Enter insert newlines (Alt+Enter is the most reliable across terminals).

### EditorComponent Interface

`EditorComponent` (`src/editor-component.ts`) defines an interface for custom editor implementations (vim mode, emacs mode, etc.) that can replace the built-in editor while maintaining compatibility with the core application. It specifies required methods (`getText`, `setText`, `handleInput`, `onSubmit`, `onChange`) and optional ones (`addToHistory`, `insertTextAtCursor`, `getExpandedText`, `setAutocompleteProvider`, `borderColor`, `setPaddingX`).

### Markdown

Renders markdown with syntax highlighting and theming. Built on the `marked` parser. Supports headings, bold, italic, strikethrough, code blocks (with optional syntax highlighting via a `highlightCode` callback), lists (ordered and unordered), links (rendered with OSC 8 hyperlinks), blockquotes, horizontal rules, and tables. HTML tags are rendered as plain text. Uses render caching for performance.

### SelectList

Interactive selection list with keyboard navigation, scroll indicators, and fuzzy filtering. Configurable layout options including custom primary-label truncation hooks. Handles multi-line descriptions by replacing newlines with spaces.

### SettingsList

Settings panel supporting value cycling (Enter/Space cycles through preset values) and submenus (Enter opens a child component). Optional fuzzy search filtering.

### Loader and CancellableLoader

`Loader` is an animated spinner component. `CancellableLoader` extends it with an Escape key handler and an `AbortSignal` that aborts when the user presses Escape.

### Image

Renders images inline using either the Kitty graphics protocol or the iTerm2 inline images protocol. Falls back to a text placeholder on unsupported terminals. Supported formats: PNG, JPEG, GIF, WebP. Image dimensions are parsed directly from binary headers. Cell dimensions are queried from the terminal via `CSI 16 t` for accurate sizing.

### Box and Spacer

- `Box` -- Container with configurable padding and background color function
- `Spacer` -- Empty lines for vertical spacing

## Keyboard Protocol and Keybindings

### Key Detection (`src/keys.ts`)

The keyboard handling system supports three input modes:

1. **Kitty keyboard protocol** -- The preferred mode. Provides unambiguous key identification including modifiers, key release events, and non-Latin keyboard layout support. Enabled when the terminal responds to the `CSI ?u` query. Flags used: disambiguate (1), report event types (2), report alternate keys (4).

2. **xterm modifyOtherKeys** -- Fallback for terminals like tmux. Uses the `CSI 27;modifier;keycode ~` format.

3. **Legacy terminal sequences** -- Traditional escape sequences for arrow keys, function keys, and ctrl combinations.

The `matchesKey(data, keyId)` function is the primary API. It accepts a type-safe `KeyId` string (e.g., `"ctrl+c"`, `"shift+enter"`, `"alt+left"`) and matches against all three modes simultaneously. The `Key` helper object provides autocomplete-friendly constants.

The `parseKey(data)` function does the reverse: given raw terminal input, it returns the key identifier string.

Key release detection: `isKeyRelease(data)` checks for Kitty protocol release events (`:3` suffix in sequences). The TUI filters these out by default unless a component sets `wantsKeyRelease = true`.

Non-Latin keyboard layout support: When the Kitty protocol reports a `baseLayoutKey` (the key in standard PC-101 QWERTY layout), `matchesKey` falls back to it for non-Latin codepoints. This allows Ctrl+C to work even when the active layout is Cyrillic, for example. However, when the primary codepoint is already a recognized Latin letter or symbol, the base layout key is ignored -- this prevents remapped layouts (Dvorak, Colemak, xremap) from causing false matches.

Kitty CSI-u printable decoding: `decodeKittyPrintable(data)` extracts printable characters from Kitty CSI-u sequences (needed because Kitty protocol flag 1 wraps all keys in CSI-u, including plain printable characters).

### Keybindings System (`src/keybindings.ts`)

The `EditorKeybindingsManager` maps editor actions to key identifiers. Each action can be bound to one or more keys. Defaults follow Emacs/readline conventions:

- **Cursor movement**: arrows, Ctrl+B/F (left/right), Ctrl+A/E (line start/end), Alt+B/F (word left/right), Ctrl+] (jump forward to character)
- **Deletion**: Backspace, Delete, Ctrl+D (delete forward), Ctrl+W/Alt+Backspace (word backward), Alt+D/Alt+Delete (word forward), Ctrl+U (to line start), Ctrl+K (to line end)
- **Clipboard**: Ctrl+C (copy), Ctrl+Y (yank), Alt+Y (yank-pop)
- **Undo**: Ctrl+- (undo)
- **Submit**: Enter (submit), Shift+Enter (new line)
- **Autocomplete**: Tab, arrows for selection, Enter to confirm, Escape to cancel
- **Tree navigation**: Ctrl+Left/Right, Alt+Left/Right (fold/unfold)
- **Tool output**: Ctrl+O (expand tools)

Custom keybindings can be set via `EditorKeybindingsConfig`:

```typescript
const manager = new EditorKeybindingsManager({
  cursorLeft: ["left", "ctrl+b"],
  submit: "enter",
});
setEditorKeybindings(manager);
```

## Autocomplete and Fuzzy Matching

### CombinedAutocompleteProvider (`src/autocomplete.ts`)

Handles two autocomplete modes:

1. **Slash commands** -- Triggered when the cursor is on a line starting with `/`. Uses fuzzy matching against registered command names. Supports argument completion for commands that provide `getArgumentCompletions()`.

2. **File paths** -- Triggered by Tab key or when the text contains path-like patterns (`~/`, `./`, `../`, `/`). Reads the filesystem directly via `readdirSync`. Supports `@` prefix for fuzzy file search using `fd` (fast, respects .gitignore). Handles quoted paths for filenames with spaces.

The file autocomplete:
- Sorts directories before files
- Supports home directory expansion (`~/`)
- Does not add trailing spaces after directory completions (so the user can continue autocompleting into subdirectories)
- Handles symlinks (resolves to check if they point to directories)

### Fuzzy Matching (`src/fuzzy.ts`)

The `fuzzyMatch(query, text)` function matches if all query characters appear in order in the text (not necessarily consecutive). Scoring rewards:
- Consecutive character matches
- Word boundary matches (after spaces, dashes, underscores, dots, slashes, colons)

Scoring penalizes:
- Gaps between matched characters
- Later match positions

An additional heuristic swaps letter/digit order in queries (e.g., `abc123` also tries `123abc`) with a small score penalty, catching common typos.

`fuzzyFilter(items, query, getText)` supports space-separated tokens: all tokens must independently match the text.

## TUI Class and Differential Rendering

### Rendering Strategies

The `TUI.doRender()` method implements three rendering strategies:

1. **First render** -- Output all lines without clearing scrollback. Assumes a clean screen.

2. **Width/height changed** -- Full clear screen (`CSI 2J`, `CSI H`, `CSI 3J`) and re-render everything. This handles terminal resizes.

3. **Normal differential update** -- Find the first and last changed lines between the previous and current render. Move the cursor to the first changed line, clear and re-render only the changed range. If new lines were appended (content grew), output is appended without clearing. If content shrunk, extra lines are cleared.

A special case: if the first changed line is above the previous viewport (content scrolled), a full re-render is triggered.

The `clearOnShrink` option (default: off, controlled by `PI_CLEAR_ON_SHRINK=1`) triggers full re-renders when content shrinks, which clears empty rows but causes more flicker.

### Synchronized Output

All rendering is wrapped in CSI 2026 synchronized output markers:
- `\x1b[?2026h` -- begin synchronized output
- `\x1b[?2026l` -- end synchronized output

This tells the terminal to buffer all output and display it atomically, eliminating flicker even with complex updates.

### Line Width Enforcement

Every rendered line is checked against the terminal width. If any line exceeds the width, TUI writes a crash log to `~/.pi/agent/pi-crash.log` with all rendered lines and their widths, then throws an error. Components must use `truncateToWidth()` or manual wrapping to ensure compliance.

After rendering, each line gets an SGR reset (`ESC[0m`) and OSC 8 reset (`ESC]8;;BEL`) appended. Styles do not carry across lines.

## Overlay System

Overlays render components on top of existing content without replacing it. The overlay system supports:

### Positioning

Overlays can be positioned using three methods (in resolution order):
1. **Absolute** -- `row` and `col` as numbers
2. **Percentage** -- `row` and `col` as strings like `"25%"` (0%=top/left, 100%=bottom/right)
3. **Anchor** -- 9 anchor points: `center`, `top-left`, `top-right`, `bottom-left`, `bottom-right`, `top-center`, `bottom-center`, `left-center`, `right-center`

Additional options:
- `width` / `maxHeight` -- sizing (absolute or percentage)
- `minWidth` -- minimum width floor
- `offsetX` / `offsetY` -- offset from anchor position
- `margin` -- margin from terminal edges (number for all sides, or `{top, right, bottom, left}`)
- `visible(termWidth, termHeight)` -- responsive callback, controls whether the overlay renders each frame

### Focus Management

- Overlays capture keyboard focus by default when shown
- `nonCapturing: true` prevents auto-focus
- `OverlayHandle.focus()` / `unfocus()` provide programmatic focus control
- Overlay compositing order follows focus order (higher = rendered on top)
- When an overlay is hidden, focus restores to the next visible overlay or the pre-focus component
- `setHidden(true/false)` temporarily hides/shows without destroying the overlay

### Compositing

The `compositeOverlays()` method merges overlay content into base content lines. For each overlay line, `compositeLineAt()` splices the overlay content into the base line at the specified column, preserving ANSI styling on both sides. A final width check truncates the result to prevent terminal overflow.

## Utilities

### visibleWidth

Calculates the visible width of a string, ignoring ANSI escape codes (SGR, OSC 8 hyperlinks, APC sequences, generic OSC sequences). Uses grapheme-based width calculation with East Asian width support. Handles ZWJ emoji sequences, regional indicators, and other complex Unicode correctly.

### truncateToWidth

Truncates a string to a maximum visible width while preserving ANSI codes. Adds an ellipsis by default (configurable). Properly closes any open ANSI sequences at the truncation point.

### wrapTextWithAnsi

Wraps text to a given width, preserving ANSI codes across line breaks. Reapplies active styles at the start of each wrapped line.

## Debug and Logging

- `PI_TUI_WRITE_LOG=<path>` -- Captures the raw ANSI stream written to stdout
- `PI_TUI_DEBUG=1` -- Writes detailed render debug info to `/tmp/tui/`
- `PI_DEBUG_REDRAW=1` -- Logs full redraw triggers to `~/.pi/agent/pi-debug.log`
- `PI_HARDWARE_CURSOR=1` -- Shows the real terminal cursor (default: hidden)
- `PI_CLEAR_ON_SHRINK=1` -- Clear empty rows when content shrinks (default: off)
