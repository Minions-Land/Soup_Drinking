# Tool System Architecture

> 🗺️ **Navigation**: [← README](../README.md) | [zh-CN version](../zh-CN/01-工具系统.md)  
> 📁 [Reference Index](../README.md#reference) | [Tutorial](../tutorial/) | [Toolkit](../toolkit/)

> Source: `src/Tool.ts` — The foundation of all tool implementations in Claude Code.

## 1. The Tool Interface

Every tool in Claude Code implements this generic interface:

```typescript
type Tool<Input, Output, Progress> = {
  name: string                    // Unique identifier
  aliases?: string[]              // Backward-compatible names
  searchHint?: string             // Keyword for ToolSearch discovery
  shouldDefer?: boolean           // Lazy-load via ToolSearch
  maxResultSizeChars: number      // Auto-persist threshold

  // Schema
  inputSchema: ZodSchema          // Zod v4 validation
  outputSchema?: ZodSchema        // Optional output validation
  inputJSONSchema?: object        // For MCP tools (raw JSON Schema)

  // Core lifecycle
  call(args, context, canUseTool, parentMessage, onProgress?)
  validateInput?(input, context) -> ValidationResult
  checkPermissions(input, context) -> PermissionResult
  description(input, options) -> string

  // Capabilities
  isEnabled() -> boolean
  isReadOnly(input) -> boolean
  isDestructive?(input) -> boolean
  isConcurrencySafe(input) -> boolean
  interruptBehavior?() -> 'cancel' | 'block'

  // UI Rendering (React/Ink)
  renderToolUseMessage(input, options) -> ReactNode
  renderToolResultMessage?(content, progress, options) -> ReactNode
  renderToolUseProgressMessage?(progress, options) -> ReactNode
  renderGroupedToolUse?(toolUses, options) -> ReactNode | null

  // Utilities
  userFacingName(input) -> string
  getPath?(input) -> string
  toAutoClassifierInput(input) -> unknown
  mapToolResultToToolResultBlockParam(content, id) -> ToolResultBlockParam
  preparePermissionMatcher?(input) -> (pattern) => boolean
  prompt(options) -> string       // System prompt for this tool
}
```

## 2. buildTool Helper

All tools use `buildTool()` to fill in safe defaults:

```typescript
const MyTool = buildTool({
  name: 'MyTool',
  inputSchema: z.object({ ... }),
  call: async (args, context) => { ... },
  // Defaults filled by buildTool:
  // isEnabled: () => true
  // isConcurrencySafe: () => false (assume not safe)
  // isReadOnly: () => false (assume writes)
  // isDestructive: () => false
  // checkPermissions: () => { behavior: 'allow' }
  // toAutoClassifierInput: () => '' (skip classifier)
  // userFacingName: () => name
})
```

## 3. ToolUseContext

The rich context passed to every tool call:

```typescript
type ToolUseContext = {
  options: {
    commands: Command[]
    tools: Tools
    mcpClients: MCPServerConnection[]
    mainLoopModel: string
    thinkingConfig: ThinkingConfig
    agentDefinitions: AgentDefinitionsResult
    maxBudgetUsd?: number
    customSystemPrompt?: string
    appendSystemPrompt?: string
  }
  abortController: AbortController
  readFileState: FileStateCache     // Read file dedup/caching
  messages: Message[]               // Conversation history
  getAppState(): AppState           // Global app state
  setAppState(f): void              // State mutations
  toolDecisions?: Map<string, {}>   // Tool-level approval cache
  agentId?: AgentId                 // Set for sub-agents
  agentType?: string                // Agent type name
  contentReplacementState?: ...     // Tool result budget mgmt
  renderedSystemPrompt?: SystemPrompt // For cache-sharing forks
}
```

## 4. Permission System

Two-stage validation before any tool executes:

```
1. validateInput(input, context)
   → Checks logical validity (file exists, path valid, etc.)
   → Returns { result: true } or { result: false, message, errorCode }

2. checkPermissions(input, context)
   → Checks user authorization
   → Returns { behavior: 'allow' | 'deny' | 'ask', updatedInput }
   → Integrates with settings rules (alwaysAllow, alwaysDeny, alwaysAsk)
```

Permission modes: `default`, `plan`, `auto`, `bypassPermissions`

## 5. Tool Result Management

```typescript
type ToolResult<T> = {
  data: T                           // The tool's output
  newMessages?: Message[]           // Inject system messages
  contextModifier?: (ctx) => ctx    // Modify context for next call
  mcpMeta?: { ... }                 // MCP protocol metadata
}
```

**Auto-persistence**: If `data` exceeds `maxResultSizeChars`, the result is saved to a temp file and Claude receives a truncated preview with the file path.

## 6. Common Tool Patterns

| Pattern | Example | Purpose |
|---------|---------|---------|
| `prompt.ts` | Separate file with template string | System prompt for the tool |
| `constants.ts` | `TOOL_NAME`, `MAX_RESULT_SIZE` | Shared constants |
| `UI.tsx` | React/Ink component | Tool-specific UI rendering |
| `utils.ts` | Helper functions | Tool-specific utilities |
| Zod schema | `z.object({ file_path: z.string() })` | Input validation |

## 7. Key Design Principles

1. **Fail-closed defaults** — Tools default to non-concurrent, non-read-only, non-destructive
2. **Two-stage validation** — Logic check (validateInput) then auth check (checkPermissions)
3. **Context-aware rendering** — Tools render differently in verbose/condensed/transcript modes
4. **Progressive result delivery** — Progress callbacks for long-running operations
5. **Deferred loading** — Less-used tools loaded on demand via ToolSearch
6. **Auto-classifier input** — Security-relevant tools provide structured input for auto-mode safety
