🌐 [中文版](../zh-CN/06-终端UI框架.md) | [← README](../../README.en.md)  
🗺️ **Navigation**: [← Ch.5](05-permission-security.md) | **Chapter 6** | [Ch.7 →](07-command-plugin-system.md)  
📁 [Tutorial EN](../en/) | [Toolkit EN](../../toolkit/en/) | [Reference](../en/)

---

# Chapter 6: Terminal UI Framework -- React for the Command Line

> 📖 [English](#english) | [中文](#中文)  
> 🗺️ **Navigation**: [← Ch.5](05-permission-security.md) | **Chapter 6** | [Ch.7 →](07-command-plugin-system.md)  
> 📁 [Tutorial Index](../../README.md#tutorial) | [Toolkit](../../toolkit/) | [Reference EN](../en/) | [Reference ZH](../zh-CN/)

<a name="english"></a>

Claude Code is a command-line tool, but its interface is far from a bare text stream. It renders permission dialogs, animated spinners, scrollable code views, clickable buttons, and multi-pane layouts -- all inside a terminal. This is possible because Claude Code uses a heavily customized fork of **Ink**, a React renderer for terminal applications. This chapter explains how the terminal becomes a canvas for declarative UI.

## What Is Ink?

Ink is an open-source library that lets you build terminal UIs using React. Instead of rendering to a browser DOM, your React components render to terminal character cells using ANSI escape sequences. You write `<Box>` and `<Text>` instead of `<div>` and `<span>`, and Ink handles layout, styling, and output.

Claude Code forked Ink and rebuilt major parts of it for extreme performance. The original Ink re-renders the entire screen on every change. Claude Code's version only re-renders what changed, uses zero-allocation buffers, and produces minimal terminal output. The result is a UI that feels instant even when rendering thousands of lines of tool output.

The entire framework lives in `src/ink/`.

## The Rendering Pipeline

Every frame in Claude Code follows a five-stage pipeline:

```
React State Change
     |
     v
Reconciler (reconciler.ts)
  - React Fiber diffs virtual tree
  - Updates Ink DOM nodes
  - Sets dirty flags on changed nodes
     |
     v
Layout (Yoga)
  - Flexbox computation on the DOM tree
  - Converts Ink styles to Yoga properties
  - Produces pixel-precise (character-cell-precise) positions
     |
     v
Render (render-node-to-output.ts)
  - Walks the DOM tree
  - SKIPS clean subtrees (blitting from previous frame)
  - Writes text, borders, backgrounds to the Screen buffer
     |
     v
Screen Buffer (screen.ts)
  - Int32Array packed cells (2 words per cell)
  - Diffs against previous frame's buffer
  - Produces minimal list of ANSI escape patches
     |
     v
Terminal Output (terminal.ts)
  - Wraps patches in DEC 2026 synchronized output
  - Single atomic write to stdout
```

The key insight is that most frames change very little -- a spinner ticks, a line of text is appended, a progress bar advances. By tracking dirty flags and diffing at the cell level, the system avoids recomputing or rewriting anything that has not changed.

## Key Optimizations

Claude Code's Ink fork introduces several optimizations that make a dramatic difference at scale:

### O(dirty) Rendering

The standard approach is to re-render the entire tree every frame. Claude Code's reconciler sets a `dirty` flag on each DOM node that changes. The render pass in `render-node-to-output.ts` checks this flag and skips entire subtrees that are clean. For a screen with 100 components where only 1 changed, this means processing 1 node instead of 100.

### GC-Free Buffers

JavaScript's garbage collector can cause visible pauses in a terminal UI -- a momentary freeze or flicker. Claude Code eliminates this by using `Int32Array` for its screen buffer. Instead of allocating JavaScript objects for each cell, it packs all cell data into a flat typed array. No objects means no GC pressure.

### Blitting

When a subtree has not changed, its output from the previous frame can be copied directly into the current frame's buffer. This is called "blitting" (block transfer). The render pass detects clean nodes via `nodeCache` and copies their rectangular regions instead of re-rendering them. This is especially effective for large static content like file previews.

### Synchronized Output

Terminal emulators render output as it arrives, which can cause visible tearing when a large update is written. Claude Code wraps each frame in DEC mode 2026 begin/end markers (`BSU`/`ESU`). Terminal emulators that support this protocol buffer the entire update and display it atomically, eliminating flicker.

### Double Buffering

The `Ink` class maintains two `Frame` objects: `frontFrame` (what was last displayed) and `backFrame` (what is being prepared). After rendering, the system diffs the two frames at the cell level and produces only the ANSI sequences needed to transform the front into the back. Then the frames swap.

## The Packed Cell Format

Each terminal cell in the screen buffer is represented as 2 x Int32 words in `src/ink/screen.ts`:

- **Word 0**: An interned character ID. Characters are stored in a `CharPool` and referenced by integer index. ASCII characters get a fast-path direct lookup; non-ASCII characters use a `Map`. The pool is shared across all screens, so blitting can copy IDs directly without re-interning.

- **Word 1**: Packed bitfields containing foreground color, background color, text attributes (bold, italic, underline, etc.), and display width. This encoding allows cell comparison with a single integer comparison rather than multiple property checks.

There is also a `HyperlinkPool` for OSC 8 hyperlink URIs and a `StylePool` for ANSI SGR sequences. All use the same interning pattern: store strings once, reference them by integer ID, compare by ID rather than string equality.

## The Event System

Terminal input is fundamentally different from browser input. Instead of named events on DOM elements, the terminal sends raw byte sequences that must be decoded. Claude Code's event pipeline works as follows:

**1. Keypress Parsing** (`src/ink/parse-keypress.ts`) -- Raw input bytes are decoded into structured key events. The parser handles multiple protocols: standard ANSI escape sequences, the Kitty keyboard protocol (for modifier key tracking), SGR mouse tracking (for clicks and scrolls), and bracketed paste mode.

**2. Event Dispatch** (`src/ink/events/dispatcher.ts`) -- Once a keypress is parsed, it enters a capture/bubble dispatch cycle modeled on the browser DOM:

```
Root (capture phase, top-down)
  -> Parent (capture)
    -> Target (at_target, both capture and bubble handlers)
  -> Parent (bubble)
Root (bubble phase, bottom-up)
```

The dispatcher walks from the target node to the root, collecting capture handlers (prepended) and bubble handlers (appended). This gives parent nodes the opportunity to intercept events before children see them, just like `addEventListener` with `capture: true` in the browser.

**3. Focus Management** (`src/ink/focus.ts`) -- The `FocusManager` tracks which node has focus. Tab navigation, auto-focus, and programmatic focus changes all route through this system. Only the focused node's handlers fire for keyboard events.

## React Hooks for the Terminal

Claude Code provides custom hooks that bridge React's declarative model with terminal-specific concerns:

| Hook | File | Purpose |
|------|------|---------|
| `useInput` | `hooks/use-input.ts` | Subscribe to keyboard events on the focused element |
| `useTerminalViewport` | `hooks/use-terminal-viewport.ts` | Know which rows are visible; enables viewport culling |
| `useAnimationFrame` | `hooks/use-animation-frame.ts` | Run logic synchronized with the render frame clock |
| `useTerminalTitle` | `hooks/use-terminal-title.ts` | Set the terminal window title via escape sequences |
| `useDeclaredCursor` | `hooks/use-declared-cursor.ts` | Position the hardware cursor for IME input |
| `useSelection` | `hooks/use-selection.ts` | Track text selection state in alt-screen mode |
| `useTabStatus` | `hooks/use-tab-status.ts` | Set the terminal tab status bar |

These hooks let component authors work with terminal features using the same declarative patterns they would use in a React web app.

## Components

The component library in `src/ink/components/` provides building blocks for terminal UIs:

- **`Box`** -- A flexbox container, equivalent to `<div>`. Supports padding, margin, borders, and all Yoga flex properties.
- **`Text`** -- Styled text with colors, bold, italic, underline, strikethrough, and dimming.
- **`ScrollBox`** -- A scrollable container with virtual scrolling. Handles mousewheel events, keyboard scrolling, and sticky-scroll (auto-follow new content).
- **`Button`** -- A clickable element with focus and hover states.
- **`Link`** -- An OSC 8 hyperlink that opens URLs in the user's browser.
- **`RawAnsi`** -- Passes pre-formatted ANSI text through without processing, for displaying raw tool output.
- **`AlternateScreen`** -- Switches to the terminal's alternate screen buffer (full-screen mode).
- **`Spacer`** -- A flexible spacer for layout.

## How Tool Results Are Rendered

When a tool completes, its result is rendered through a tool-specific UI component. Each tool defines `renderResultUI()` and `renderToolUseProgressUI()` methods that return React elements. These elements are composed into the conversation view, laid out by Yoga, rendered to the screen buffer, and diffed against the previous frame.

For example, when `FileReadTool` returns file content, it renders syntax-highlighted text inside a `ScrollBox`. When `BashTool` returns command output, it renders the output with appropriate error styling if the command failed. The tool does not need to know about ANSI codes or terminal dimensions -- it just returns React components, and the framework handles the rest.

## Key Takeaway

Claude Code proves that terminal UIs do not have to be primitive. By applying React's declarative component model to terminal rendering -- and investing heavily in performance through dirty tracking, typed-array buffers, blitting, and synchronized output -- you can build interfaces that rival GUI applications in responsiveness and complexity. The framework handles the hard parts (ANSI escapes, layout computation, input protocol parsing, GC avoidance) so that component authors can focus on what the UI should look like, not how to render it character by character.
---
📁 [← Back to Tutorial Index](../../README.en.md#tutorial) | 🌐 [中文版](../zh-CN/06-终端UI框架.md) | [Ch.7 →](07-command-plugin-system.md)
