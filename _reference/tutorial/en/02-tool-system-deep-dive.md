🌐 [中文版](../../tutorial/zh-CN/02-工具系统深入.md) | [← README](../../README.en.md)  
🗺️ **Navigation**: [← Ch.1](01-architecture-overview.md) | **Chapter 2** | [Ch.3 →](03-multi-agent-system.md)  
📁 [Tutorial EN](../../tutorial/en/) | [Toolkit EN](../../toolkit/en/) | [Reference](../../en/)

---

# Chapter 2: The Tool System -- How Tools Work

> 📖 [English](#english) | [中文](#中文)  
> 🗺️ **Navigation**: [← Ch.1](01-architecture-overview.md) | **Chapter 2** | [Ch.3 →](03-multi-agent-system.md)  
> 📁 [Tutorial Index](../README.md#tutorial) | [Toolkit](../toolkit/) | [Reference EN](../en/) | [Reference ZH](../zh-CN/)

<a name="english"></a>

## Introduction

Tools are the primary way Claude Code interacts with the outside world. The model cannot read files, run commands, or edit code on its own. It must request a tool call, and the system executes it. This chapter explains the tool interface, walks through a real tool, and shows you how to design your own.

## The Tool Interface

Every tool in Claude Code implements the `Tool` interface defined in `src/Tool.ts`. Here is its essential shape:

```typescript
interface Tool {
  name: string;                    // Unique identifier (e.g., "Read")
  description: string;             // Shown to the model in the system prompt
  inputSchema: ZodSchema;          // Zod schema for input validation
  
  // Lifecycle methods
  validateInput?(input): Promise<ValidationResult>;
  checkPermissions?(input, context): Promise<PermissionResult>;
  call(input, context): Promise<ToolResult>;
  
  // UI methods
  renderToolUseMessage?(input): React.ReactNode;
  renderToolResultMessage?(result): React.ReactNode;
  
  // Metadata
  isReadOnly?: boolean;            // Does this tool modify state?
  isEnabled?(context): boolean;    // Should this tool be available?
  prompt?: string;                 // Extra instructions for the model
  userFacingName?: string;         // Display name in the UI
}
```

This interface is the contract every tool must honor. Notice the separation: validation is separate from permissions, which is separate from execution, which is separate from rendering. Each concern has its own method, making the system modular and testable.

## The buildTool() Pattern

Rather than implementing all methods directly, tools use the `buildTool()` factory function. This function takes a configuration object and returns a fully-formed `Tool`:

```typescript
export const MyTool = buildTool({
  name: "MyTool",
  description: "Does something useful",
  inputSchema: z.object({
    path: z.string().describe("The file path to process"),
  }),
  isReadOnly: true,
  async call(input, context) {
    // Do the work
    return { result: "Done" };
  },
});
```

`buildTool()` provides sensible defaults for optional methods. Crucially, the defaults are **conservative**: a new tool is assumed to be write-capable and not concurrency-safe until you explicitly say otherwise. This is a "fail-closed" design -- forgetting to set a flag results in more restrictions, not fewer.

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,    // assume NOT safe
  isReadOnly: () => false,           // assume writes
  checkPermissions: (input) =>       // defer to general permission system
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
};
```

## Walkthrough: FileReadTool

Let us trace through `FileReadTool` as a concrete example. This tool reads a file from disk and returns its contents.

**File structure:**

```
src/tools/FileReadTool/
  |-- FileReadTool.ts   # Main implementation (call, validation, permissions)
  |-- UI.tsx            # Rendering functions
  |-- prompt.ts         # Tool description and prompt constants
  |-- limits.ts         # Token and size limits
  |-- imageProcessor.ts # Image handling logic
```

This is the standard tool directory layout: implementation, UI, prompt, and specialized modules.

**Input schema:**

```typescript
const inputSchema = z.object({
  file_path: z.string().describe("Absolute path to the file"),
  offset: z.number().optional().describe("Line number to start from"),
  limit: z.number().optional().describe("Number of lines to read"),
})
```

Zod schemas serve double duty: they define what the model sees in the tool description (parameter names, types, constraints) and they validate the model's output at runtime.

**validateInput():** Checks that the file path is absolute (not relative), that it does not point to a blocked device like `/dev/zero`, and that `offset` and `limit` are non-negative. If validation fails, it returns a human-readable error message that the model sees, so it can correct its request.

**checkPermissions():** Calls `checkReadPermissionForTool()` which verifies that the user has granted read access to this file path. If the path is inside a protected directory (like `~/.ssh`), the permission check triggers a user approval prompt.

**call():** The actual work: resolve the file path, detect file type (text, image, PDF, notebook, binary), read with line ranges and line numbers, check against token limits, and return the result with appropriate metadata.

**renderToolUseMessage():** Displays the file path and range being read in the terminal UI.

**renderToolResultMessage():** Shows a summary (file name, line count) rather than the full file contents.

## The Two-Stage Validation Pattern

Claude Code uses a two-stage validation approach that is worth studying:

**Stage 1: Schema validation (automatic).** The `inputSchema` (a Zod schema) is validated automatically before any method is called. If the model passes `{ file_path: 123 }` instead of a string, the error is caught here and the model receives a structured error message. This is cheap and catches structural errors.

**Stage 2: Semantic validation (validateInput).** The `validateInput()` method checks business logic: Does the file exist? Is the path absolute? Is the range valid? This stage produces human-readable errors that help the model self-correct.

The key insight: validation errors get a clear error message back to the model ("path must be absolute"), while permission denials trigger the user-facing permission prompt ("Allow Read on /etc/passwd?"). Separating these concerns keeps each layer simple.

## ToolUseContext

When a tool's `call()` method is invoked, it receives a `ToolUseContext` object. This context provides everything the tool needs without importing globals:

```typescript
interface ToolUseContext {
  appState: AppState;              // Global application state
  abortSignal: AbortSignal;        // For cancellation
  options: ToolOptions;            // Tool-specific config
  readFileTimestamps: Map;         // Staleness detection
  setToolResult(result): void;     // For streaming results
  messages: Message[];             // Full conversation history
  tools: Tool[];                   // The full tool pool
}
```

Key points about the context:

- **AppState** gives access to the permission system, conversation history, task registry, and configuration. Tools should read from it, not write to it directly.
- **AbortSignal** lets tools be cancelled mid-execution. Long-running tools (like Bash) check this signal regularly.
- **readFileTimestamps** is used by write tools to detect if a file changed since it was last read, preventing accidental overwrites.
- **tools** -- the full tool pool, because tools can invoke other tools.

This context-injection approach keeps tools decoupled from global state. They do not import singletons; they receive everything they need.

## Tool Result Management

A tool's `call()` method returns a `ToolResult`, which can be a string, a content block array (text, images, structured data), or an error (with `is_error: true`).

When a result exceeds `maxResultSizeChars` (typically 40,000 characters), the system persists the full output to disk and sends the model a truncated preview with a path to the full file. This prevents context window exhaustion while keeping the model informed.

Tools can also produce **progressive results** using `setToolResult()`. This lets a tool stream partial output (like lines from a running command) before the final result is ready. The UI renders these results as they arrive.

## How to Design Your Own Tools

If you want to add a tool to an agent system inspired by Claude Code, follow these guidelines:

1. **Start with the schema.** Define the input shape with Zod. Be explicit with `.describe()` -- the model reads these descriptions to understand how to use the tool.
2. **Keep tools focused.** Each tool should do one thing. A tool that reads files should not also write them.
3. **Validate before executing.** Use two-stage validation. Catch structural errors with the schema, business logic errors with `validateInput()`.
4. **Return structured errors.** When something goes wrong, return a clear error message, not an exception. The model uses these errors to self-correct.
5. **Mark read-only tools.** If your tool does not modify state, set `isReadOnly: true`. This helps the permission system and enables parallel execution.
6. **Inject context, do not import globals.** Pass everything a tool needs through a context object. This makes tools testable and composable.
7. **Manage result sizes.** Large results can exhaust the context window. Set a threshold and persist oversized results to disk.
8. **Respect cancellation.** Check the `AbortSignal` in long-running operations.

The tool system is Claude Code's most important abstraction. Every capability -- from reading files to spawning sub-agents -- is a tool. Understanding this system unlocks everything else.
---
📁 [← Back to Tutorial Index](../../README.en.md#tutorial) | 🌐 [中文版](../../tutorial/zh-CN/02-工具系统深入.md) | [Ch.3 →](03-multi-agent-system.md)
