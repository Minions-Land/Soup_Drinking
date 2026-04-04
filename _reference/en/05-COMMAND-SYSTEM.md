# Command System & CLI Architecture

> 🗺️ **Navigation**: [← README](../README.md) | [zh-CN version](../zh-CN/05-命令系统.md)  
> 📁 [Reference Index](../README.md#reference) | [Tutorial](../tutorial/) | [Toolkit](../toolkit/)

> Sources: `src/commands.ts`, `src/commands/`, `src/cli/`, `src/vim/`, `src/plugins/`, `src/keybindings/`

## 1. Command Registry

Central registry (`src/commands.ts`) aggregates commands from multiple sources:

| Source | Type | Example |
|--------|------|---------|
| Built-in | Local/UI commands | `/exit`, `/help`, `/clear` |
| Skills | Prompt-based AI commands | `/commit`, `/review-pr` |
| Plugins | Dynamically loaded | From `src/plugins/` |
| Workflows | Scriptable tool chains | Custom workflows |

### Command Types

```typescript
type Command = {
  name: string
  type: 'prompt' | 'local' | 'local-jsx'
  // 'prompt' → expands to LLM prompt
  // 'local' → executes JS logic, returns text
  // 'local-jsx' → renders React/Ink UI component
}
```

### Available Slash Commands (partial list)

| Command | Purpose |
|---------|---------|
| `/help` | Show help |
| `/clear` | Clear conversation |
| `/compact` | Compact conversation history |
| `/config` | Edit settings (interactive) |
| `/commit` | Create git commit |
| `/branch` | Switch/create branches |
| `/context` | Manage conversation context |
| `/copy` | Copy content to clipboard |
| `/color` | Change theme |
| `/add-dir` | Add working directory |
| `/agents` | List/manage agents |
| `/bridge` | Remote control management |
| `/chrome` | Chrome integration |
| `/advisor` | Get advice |

### Filtering & Dispatch

Commands filtered by:
- **Auth level** — `claude-ai` vs `console` users
- **Mode** — `REMOTE_SAFE_COMMANDS` for remote sessions
- **Bridge safety** — safe for mobile/web remote control

## 2. CLI Infrastructure

### Structured IO (`src/cli/structuredIO.ts`)

Base layer for NDJSON message exchange:
- Handles `control_request` (permission prompts)
- Handles `control_response` (user answers)
- Protocol for remote operation

### Remote IO (`src/cli/remoteIO.ts`)

Extends StructuredIO for bidirectional streaming:
- Integrates with CCR (Claude Control REPL)
- Heartbeat management
- Epoch tracking
- Session state reporting

### Transports (`src/cli/transports/`)

| Transport | Protocol | Use Case |
|-----------|----------|----------|
| `SSETransport` | Server-Sent Events | CCR v2 |
| `WebSocketTransport` | WebSocket | Full-duplex |
| `HybridTransport` | WS+POST | Read=WS, Write=POST |
| `ccrClient.ts` | CCR protocol | Claude Control REPL |

## 3. Vim Mode (`src/vim/`)

Full Vim emulation in the terminal input:

### State Machine

```
idle → count → operator → operatorFind/operatorTextObj → idle
```

States: `idle`, `count`, `operator`, `operatorFind`, `operatorTextObj`

### Motions (`motions.ts`)
Pure functions calculating target cursor positions:
- `h`, `j`, `k`, `l` — basic movement
- `w`, `b`, `e` — word movement
- `G`, `gg` — line jumping
- `f`, `F`, `t`, `T` — find character
- `0`, `$`, `^` — line positions

### Operators (`operators.ts`)
- `d` — delete
- `c` — change
- `y` — yank
- `r` — replace
- `>`, `<` — indent/dedent

### Text Objects (`textObjects.ts`)
- `iw` / `aw` — inner/around word
- `i"` / `a"` — inner/around quotes
- `i(` / `a(` — inner/around parens

## 4. Plugin System (`src/plugins/`)

### Plugin Architecture

Plugins provide:
- **Skills** — prompt-based commands
- **Hooks** — e.g., `PermissionRequest` interceptors
- **MCP servers** — external tool providers

### Built-in Plugins (`plugins/bundled/`)
- Ship with CLI
- Can be toggled via `/plugin` UI
- Standard toggle/enable/disable patterns

### Marketplace Plugins
- Loaded from external sources
- Third-party extensions
- Standard registration interface

## 5. Keybindings (`src/keybindings/`)

### Key Features
- **Multi-keystroke chords** — e.g., `ctrl+k ctrl+s`
- **Context-aware** — scoped to `Global`, `Chat`, `Vim`
- **Customizable** — user overrides loaded from settings
- **Resolver** — handles chord sequences and disambiguation

## 6. Migrations (`src/migrations/`)

Auto-migration scripts for settings between CLI versions:

| Migration | Purpose |
|-----------|---------|
| `migrateSonnet45ToSonnet46` | Model upgrade |
| `migrateOpusToOpus1m` | Context window upgrade |
| `migrateAutoUpdatesToSettings` | Settings restructure |
| `resetProToOpusDefault` | Default model change |
| `migrateBypassPermissionsAcceptedToSettings` | Permission settings move |

## 7. Buddy Feature (`src/buddy/`)

Gamified companion system:
- **Deterministic generation** via seeded PRNG (`mulberry32`) from user ID
- **Attributes**: species, rarity (common→legendary), randomized stats
- **ASCII sprites** rendered via Ink components
