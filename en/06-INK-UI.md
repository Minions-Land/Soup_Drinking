# Terminal UI Framework (Ink)

> 🗺️ **Navigation**: [← README](../README.md) | [zh-CN version](../zh-CN/06-终端UI.md)  
> 📁 [Reference Index](../README.md#reference) | [Tutorial](../tutorial/) | [Toolkit](../toolkit/)

> Source: `src/ink/` — A high-performance React-for-CLI framework, custom fork of Ink.

## 1. System Overview

Key optimizations over standard Ink:
- **O(dirty) rendering** — only modified subtrees re-processed
- **GC-free cell buffers** — Typed arrays (Int32Array) eliminate GC pauses
- **Blitting** — clean subtree output copied from previous frame
- **Synchronized output** — DEC mode 2026 prevents flicker
- **Advanced terminal probing** — capability detection without blocking

## 2. Rendering Pipeline

```
Input → React State Change → Reconciler → DOM Update → Dirty Flag
  → Layout (Yoga) → Render Tree Walk → Blit/Render → Screen Buffer
  → Diff Old/New → Minimal ANSI Output → Synchronized Write to stdout
```

### Key Components

| Component | File | Role |
|-----------|------|------|
| Renderer | `renderer.ts` | Frame lifecycle, double-buffering |
| RenderNodeToOutput | `render-node-to-output.ts` | Tree walker, blitting optimization |
| Output | `output.ts` | Accumulator for write/blit/clear/clip |
| Screen | `screen.ts` | Low-level buffer (`Int32Array` packed cells) |
| Optimizer | `optimizer.ts` | Merge/simplify terminal update patches |
| Terminal | `terminal.ts` | Final stdout writer, synchronized output |

### Packed Cell Format

Each cell = 2x Int32 words:
- **Word 0**: Character ID (interned)
- **Word 1**: Packed bitfields — FG color, BG color, attributes (bold/italic/etc.), display width

Interning pools for characters, styles, and hyperlink URIs.

## 3. Layout Engine

Grid-aligned flexbox via Yoga WASM:

| Component | File | Role |
|-----------|------|------|
| Yoga Bridge | `layout/yoga.ts` | Ink styles → Yoga properties |
| Layout Node | `layout/node.ts` | Coordinate rounding for char-cell alignment |
| Geometry | `layout/geometry.ts` | Geometry calculations |
| Engine | `layout/engine.ts` | Layout computation driver |

## 4. DOM Abstraction

| Component | File | Role |
|-----------|------|------|
| DOM Elements | `dom.ts` | `DOMElement`, `TextNode` with dirty flags |
| Reconciler | `reconciler.ts` | React Fiber ↔ Ink DOM mapping |
| Root | `root.ts` | Root container management |
| Focus Manager | `focus.ts` | Tab navigation, auto-focus |

## 5. Event System

| Component | File | Role |
|-----------|------|------|
| Input Parser | `parse-keypress.ts` | ANSI, Kitty protocol, SGR mouse, bracketed paste |
| Dispatcher | `events/dispatcher.ts` | Capture/bubble event dispatch |
| Emitter | `events/emitter.ts` | Event emitter base |
| Event Types | `events/*.ts` | Keyboard, Click, Focus, Terminal events |

Terminal Querier (`terminal-querier.ts`): Probes capabilities via DA1 sentinels.

## 6. Text Processing

| Component | File | Purpose |
|-----------|------|---------|
| String Width | `stringWidth.ts` | CJK, emoji, zero-width joiner handling |
| Bidi | `bidi.ts` | RTL language support |
| Colorize | `colorize.ts` | 8/16/256/Truecolor mode mapping |
| Wrap Text | `wrap-text.ts` | Word wrapping |
| Measure Text | `measure-text.ts` | Text dimension calculation |
| Search Highlight | `searchHighlight.ts` | Search result highlighting |
| Slice ANSI | `wrapAnsi.ts` | ANSI-aware text slicing |

## 7. Terminal I/O (`termio/`)

ANSI escape sequence serialization/deserialization:

| Module | Purpose |
|--------|---------|
| `parser.ts` | Streaming ANSI sequence parser |
| `tokenize.ts` | Tokenizer for escape sequences |
| `csi.ts` | Control Sequence Introducer |
| `sgr.ts` | Select Graphic Rendition (colors, styles) |
| `osc.ts` | Operating System Command (titles, links) |
| `esc.ts` | Escape sequences |
| `dec.ts` | DEC private modes |
| `ansi.ts` | ANSI constants |

## 8. React Hooks

| Hook | Purpose |
|------|---------|
| `useTerminalViewport` | Visibility-based culling (offscreen detection) |
| `useAnimationFrame` | Synchronized animation clock (pauses when hidden) |
| `useInput` | High-level keyboard input handling |
| `useStdin` | Raw stdin access |
| `useTerminalTitle` | Window title control |
| `useTabStatus` | Tab status indicator (OSC 21337) |
| `useTerminalFocus` | Terminal focus/blur detection |
| `useSelection` | Text selection management |
| `useDeclaredCursor` | Cursor position declaration |
| `useInterval` | Repeating timer |
| `useSearchHighlight` | Search highlight state |

## 9. Components

| Component | Purpose |
|-----------|---------|
| `Box` | Flexbox container |
| `Text` | Styled text |
| `ScrollBox` | Scrollable container |
| `Button` | Clickable button |
| `Link` | Clickable hyperlink |
| `Spacer` | Flexible space |
| `Newline` | Line break |
| `RawAnsi` | Raw ANSI passthrough |
| `NoSelect` | Non-selectable region |
| `AlternateScreen` | Alternate screen buffer |
| `ErrorOverview` | Error display |

## 10. Interaction Flow Summary

```
1. User presses key
2. parse-keypress decodes ANSI/Kitty/SGR
3. Dispatcher sends to focused DOMElement (capture → bubble)
4. React state changes
5. Reconciler modifies DOMElement → marks "dirty"
6. Renderer triggers Yoga layout for dirty subtree
7. RenderNodeToOutput walks tree:
   - Clean nodes → blit from old buffer
   - Dirty nodes → re-render to Output
8. Output writes to Screen typed arrays
9. Terminal diffs old/new screen → generates minimal ANSI
10. Wraps in Synchronized Update (DEC 2026) → writes stdout
```
