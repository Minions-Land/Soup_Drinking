# Utilities & Core Infrastructure

> 🗺️ **Navigation**: [← README](../README.md) | [zh-CN version](../zh-CN/08-工具与核心.md)  
> 📁 [Reference Index](../README.md#reference) | [Tutorial](../tutorial/) | [Toolkit](../toolkit/)

> Sources: `src/utils/`, `src/context/`, `src/bootstrap/`, `src/constants/`, `src/types/`, `src/memdir/`

## 1. Core Engine

### QueryEngine (`src/QueryEngine.ts`)
The main conversation loop:
- Sends conversation history to model
- Processes tool calls
- Manages tool concurrency (parallel vs serial)
- Applies tool result budget
- Handles compaction when context grows too large

### Core Types (`src/types/`)
- `message.ts` — UserMessage, AssistantMessage, SystemMessage, ProgressMessage
- `permissions.ts` — PermissionMode, PermissionResult, ToolPermissionRulesBySource
- `tools.ts` — Tool progress types (BashProgress, AgentToolProgress, etc.)
- `hooks.ts` — Hook types (PreToolUse, PostToolUse, etc.)
- `ids.ts` — AgentId, SessionId type definitions

## 2. Settings (`src/utils/settings/`)

| File | Purpose |
|------|---------|
| `settings.ts` | Core settings read/write |
| `validation.ts` | Schema validation |
| `settingsCache.ts` | Performance caching |
| `types.ts` | Settings type definitions |
| `applySettingsChange.ts` | Atomic settings updates |
| `changeDetector.ts` | Settings change monitoring |
| `mdm/` | Managed Device Management (enterprise) |
| `permissionValidation.ts` | Permission rule validation |
| `toolValidationConfig.ts` | Tool-specific setting rules |

## 3. Permissions (`src/utils/permissions/`)

Security gateway for tool execution:
- Feature gates for experimental tools
- User consent enforcement
- `denialTracking.ts` — Tracks repeated denials, escalates to ask
- `isScratchpadEnabled()` — Feature gate for shared memory

## 4. Memory System (`src/utils/memory/`)

| File | Purpose |
|------|---------|
| `versions.ts` | Memory format versioning |
| `types.ts` | Memory entry types |

Memory lives in `.claude/projects/` as markdown files with frontmatter.

## 5. MCP Integration (`src/utils/mcp/`)

Model Context Protocol utilities:
- Client connection management
- Server registration
- Tool schema conversion (MCP → Claude tool format)
- Resource access

## 6. Shell & Process (`src/utils/shell/`, `src/utils/bash/`)

Shell execution infrastructure:
- Process spawning and management
- Environment variable handling
- Platform-specific adaptations (macOS, Linux, Windows)

## 7. Hook System (`src/utils/hooks/`)

Event-driven extensibility:
- `PreToolUse` — Before tool execution
- `PostToolUse` — After tool execution
- `session_start` — Session initialization
- `pre_compact` / `post_compact` — Context compaction

## 8. Plugin System (`src/utils/plugins/`)

Plugin loading and lifecycle:
- Discovery from multiple sources
- Activation/deactivation
- Capability registration (skills, hooks, MCP servers)

## 9. Swarm Coordination (`src/utils/swarm/`)

Multi-agent swarm infrastructure:

| File | Purpose |
|------|---------|
| `spawnUtils.ts` | Launch teammates |
| `inProcessRunner.ts` | Stay-alive loop for in-process agents |
| `permissionSync.ts` | Leader → worker permission propagation |
| `teammateModel.ts` | Model selection for teammates |
| `reconnection.ts` | Handle disconnect recovery |
| `teamHelpers.ts` | Team management utilities |
| `teammatePromptAddendum.ts` | Teammate-specific prompt additions |
| `backends/TmuxBackend.ts` | Spawn in tmux panes |
| `backends/ITermBackend.ts` | Spawn in iTerm tabs |
| `backends/InProcessBackend.ts` | Spawn in same Node process |

## 10. Context Management (`src/context/`)

Manages conversation context:
- System prompt assembly
- Context window tracking
- Memory file injection
- CLAUDE.md loading

## 11. Bootstrap (`src/bootstrap/`)

Application initialization:
- Global state setup
- MCP client initialization
- Session context restoration
- Tool and skill registration

## 12. Memdir (`src/memdir/`)

Shared filesystem area for agents:
- Durable cross-worker knowledge layer
- Scratchpad directory management
- Path injected via coordinator

## 13. Other Utility Modules

| Module | Purpose |
|--------|---------|
| `processUserInput/` | Input processing pipeline |
| `background/` | Background task management |
| `teleport/` | Teleport functionality |
| `secureStorage/` | Secure credential storage |
| `claudeInChrome/` | Chrome extension integration |
| `todo/` | Todo list management |
| `sandbox/` | Sandboxing for tool execution |
| `dxt/` | Extension (DXT) format support |
| `nativeInstaller/` | Native app installer |
| `github/` | GitHub API integration |
| `computerUse/` | Computer use capabilities |
| `powershell/` | PowerShell support |
| `task/` | Task utilities |
| `cronTasks.ts` | Cron task scheduling |
| `fileRead.ts` | File reading utilities |
| `gitDiff.ts` | Git diff processing |
| `fileHistory.ts` | File modification tracking |
| `pdfUtils.ts` | PDF processing |
| `notebook.ts` | Jupyter notebook handling |
| `toolSearch.ts` | Tool discovery by keyword |
| `thinking.ts` | Extended thinking configuration |
| `sanitization.ts` | Input sanitization |
| `path.ts` | Path utilities |
| `uuid.ts` | UUID generation |
| `json.ts` / `jsonRead.ts` | JSON parsing utilities |
| `gracefulShutdown.ts` | Cleanup on exit |
| `releaseNotes.ts` | Release notes display |
| `editor.ts` | External editor integration |
| `imagePaste.ts` | Image paste handling |
| `imageValidation.ts` | Image validation |
