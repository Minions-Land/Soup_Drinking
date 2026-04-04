🌐 [中文版](../../tutorial/zh-CN/01-架构总览.md) | [← README](../../README.en.md)  
🗺️ **Navigation**: [← README](../../README.en.md) | **Chapter 1** | [Ch.2 →](02-tool-system-deep-dive.md)  
📁 [Tutorial EN](../../tutorial/en/) | [Toolkit EN](../../toolkit/en/) | [Reference](../../en/)

---

# Chapter 1: Understanding Claude Code's Architecture -- The Big Picture

> 📖 [English](#english) | [中文](#中文)  
> 🗺️ **Navigation**: [← README](../README.md) | **Chapter 1** | [Chapter 2 →](02-tool-system-deep-dive.md)  
> 📁 [Tutorial Index](../README.md#tutorial) | [Toolkit](../toolkit/) | [Reference EN](../en/) | [Reference ZH](../zh-CN/)

<a name="english"></a>

Welcome to this tutorial series on building agent systems, using Claude Code's source code as our reference architecture. Whether you are building your own AI-powered CLI, an autonomous coding assistant, or any agentic system, understanding how Claude Code is structured will give you a strong foundation.

## What Is Claude Code?

Claude Code is an AI-powered command-line tool for software engineering. You type natural language requests, and it reads files, writes code, runs commands, searches codebases, and orchestrates sub-agents -- all autonomously. Think of it as a senior developer who lives in your terminal.

But beneath the friendly interface lies a carefully designed agent architecture. Let's pull back the curtain.

## The Three Pillars

Claude Code's architecture rests on three core abstractions:

| Pillar | Role | Key File |
|--------|------|----------|
| **QueryEngine** | The brain -- manages conversation state, calls the model, loops until done | `src/QueryEngine.ts` |
| **Tools** | The hands -- discrete capabilities the model can invoke (read files, run bash, etc.) | `src/Tool.ts` + `src/tools/` |
| **Tasks** | The workers -- background processes and long-running operations | `src/Task.ts` + `src/tasks/` |

Think of it this way: the **QueryEngine** decides *what* to do, **Tools** decide *how* to do it, and **Tasks** handle work that runs *alongside* the main conversation.

## Data Flow: From Keypress to Result

Here is the journey of a user request through the system:

```
 User Input
     |
     v
 +-----------------------+
 | Command Parsing       |  commands.ts / processUserInput()
 | (slash commands, etc.) |
 +---------+-------------+
           |
           v
 +-----------------------+
 | QueryEngine           |  QueryEngine.ts
 | - builds system prompt |
 | - calls Claude API     |
 | - enters agentic loop  |
 +---------+-------------+
           |
           v
 +-----------------------+
 | query() loop          |  query.ts
 | - streams response     |
 | - detects tool_use     |
 | - executes tools       |
 | - feeds results back   |
 +---------+-------------+
           |
       tool_use?
      /        \
    yes         no
     |           |
     v           v
 +----------+  +----------+
 | Tool      |  | Render   |
 | execution |  | final    |
 | pipeline  |  | response |
 +----------+  +----------+
     |
     v
 Result fed back
 to query() loop
 (next API turn)
```

This is a **ReAct loop** (Reason + Act). The model reasons about what to do, acts by calling a tool, observes the result, and repeats until it has a final answer. The loop lives in `query.ts` and runs until the model emits `stop_reason: "end_turn"`.

Let's walk through each step:

**1. Command Parsing** -- The user's text is processed by `processUserInput()`. Slash commands like `/compact` or `/help` are handled locally and never reach the model. Everything else becomes a user message.

**2. QueryEngine** -- The `QueryEngine` class (in `src/QueryEngine.ts`) owns the conversation lifecycle. It assembles the system prompt, injects user context (working directory, project info, tool descriptions), and starts the query loop. One `QueryEngine` instance lives for the entire conversation session.

**3. The query() Loop** -- The `query()` function in `src/query.ts` is the agentic loop. It calls the Claude API, streams the response, and when the model emits a `tool_use` block, it executes the corresponding tool. The tool's result is appended as a `tool_result` message, and the loop continues. This repeats until the model emits a final text response with `stop_reason: "end_turn"`.

**4. Tool Execution** -- Each tool goes through a validation pipeline: schema validation (Zod), then `validateInput()`, then `checkPermissions()`, then `call()`. We will cover this in detail in Chapter 2.

**5. Result Rendering** -- The final assistant message is rendered to the terminal via Ink (React for CLIs). Each tool also has its own render methods for showing progress and results.

## Directory Structure Overview

```
src/
 |-- QueryEngine.ts      # The brain: conversation lifecycle
 |-- Tool.ts              # Tool interface + buildTool() factory
 |-- Task.ts              # Task types + state management
 |-- commands.ts          # Slash command registry
 |-- query.ts             # The agentic loop (model call + tool dispatch)
 |-- tools.ts             # Tool pool assembly
 |
 |-- tools/               # One directory per tool
 |   |-- BashTool/        # Shell command execution
 |   |-- FileReadTool/    # File reading with line ranges
 |   |-- FileEditTool/    # Surgical file editing
 |   |-- FileWriteTool/   # Full file writes
 |   |-- GrepTool/        # Regex search across files
 |   |-- GlobTool/        # File pattern matching
 |   |-- AgentTool/       # Sub-agent spawning (Chapter 3)
 |   |-- SendMessageTool/ # Inter-agent messaging
 |   +-- ...              # 30+ more tools
 |
 |-- tasks/               # Background task implementations
 |-- commands/            # Slash command implementations
 |-- ink/                 # Terminal UI components (React/Ink)
 |-- bridge/              # IDE integration bridge
 |-- services/            # API clients, MCP, analytics
 |-- utils/               # Shared utilities (permissions, config, etc.)
 |-- context/             # System prompt context assembly
 +-- state/               # Application state (AppState)
```

## ASCII Architecture Diagram

```
+------------------------------------------------------------------+
|                        Claude Code CLI                            |
+------------------------------------------------------------------+
|  Terminal UI (Ink/React)                                          |
|  +------------------------------------------------------------+  |
|  |  Input Bar  |  Message Stream  |  Status Bar  |  Tool View  |  |
|  +------------------------------------------------------------+  |
+------------------------------------------------------------------+
|  QueryEngine                                                     |
|  +-------------------+  +--------------------+  +--------------+ |
|  | System Prompt     |  | Conversation Mgr   |  | Tool Router  | |
|  | Builder           |  | (history, context) |  | (dispatch)   | |
|  +-------------------+  +--------------------+  +--------------+ |
+------------------------------------------------------------------+
|  Tool Layer                                                      |
|  +--------+ +--------+ +--------+ +--------+ +--------+         |
|  |  Bash  | | Read   | | Write  | | Grep   | | Agent  |  ...    |
|  +--------+ +--------+ +--------+ +--------+ +--------+         |
+------------------------------------------------------------------+
|  Task Layer                                                      |
|  +------------------+  +------------------+  +----------------+  |
|  | Background Tasks |  | Task Registry    |  | Zombie Guard   |  |
|  +------------------+  +------------------+  +----------------+  |
+------------------------------------------------------------------+
|  Infrastructure                                                  |
|  +-------------+ +-------------+ +------------+ +--------------+ |
|  | Permissions | | Hooks       | | Bridge     | | Persistence  | |
|  +-------------+ +-------------+ +------------+ +--------------+ |
+------------------------------------------------------------------+
```

## Key Files to Read First

If you want to understand Claude Code by reading source code, start with these five files:

1. **`src/Tool.ts`** -- Defines the `Tool` interface and the `buildTool()` factory. Every tool conforms to this shape.
2. **`src/tools/FileReadTool/FileReadTool.ts`** -- A well-structured tool. Reading it teaches you the full tool lifecycle.
3. **`src/QueryEngine.ts`** -- The core loop. Follow `submitMessage()` to see how the model is called and how tool calls are dispatched.
4. **`src/Task.ts`** -- The task abstraction. Understand how background work is scheduled and tracked.
5. **`src/tools/AgentTool/AgentTool.tsx`** -- The multi-agent entry point. See how a sub-agent is spawned with its own context.

## Design Philosophy

Claude Code embodies several principles worth studying:

- **Tools are data, not code.** Each tool is declared as a data structure (name, description, schema, handler). The system can introspect, filter, and compose tools without executing them.
- **The model is the orchestrator.** The code does not hard-code workflows. It gives the model a set of tools and lets the model decide the sequence.
- **Security is layered.** Permissions, sandboxing, hooks, and validation each provide an independent defense layer.
- **The UI is decoupled.** The terminal renderer is a React app (via Ink) that subscribes to state changes. The core logic knows nothing about rendering.

In the next chapter, we dive deep into the Tool interface and walk through a real tool implementation.
---
📁 [← Back to Tutorial Index](../../README.en.md#tutorial) | 🌐 [中文版](../../tutorial/zh-CN/01-架构总览.md) | [Ch.2 →](02-tool-system-deep-dive.md)
