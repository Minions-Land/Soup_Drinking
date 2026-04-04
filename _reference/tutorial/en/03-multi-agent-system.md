🌐 [中文版](../../tutorial/zh-CN/03-多智能体系统.md) | [← README](../../README.en.md)  
🗺️ **Navigation**: [← Ch.2](02-tool-system-deep-dive.md) | **Chapter 3** | [Ch.4 →](04-task-lifecycle.md)  
📁 [Tutorial EN](../../tutorial/en/) | [Toolkit EN](../../toolkit/en/) | [Reference](../../en/)

---

# Chapter 3: The Multi-Agent System

> 📖 [English](#english) | [中文](#中文)  
> 🗺️ **Navigation**: [← Ch.2](02-tool-system-deep-dive.md) | **Chapter 3** | [Ch.4 →](04-task-lifecycle.md)  
> 📁 [Tutorial Index](../README.md#tutorial) | [Toolkit](../toolkit/) | [Reference EN](../en/) | [Reference ZH](../zh-CN/)

<a name="english"></a>

## Why Multiple Agents?

A single model turn can accomplish a lot, but some tasks are too complex for one linear sequence of tool calls. You might need to search across a large codebase, analyze findings, implement changes in multiple files, and verify correctness -- all within one user request. Claude Code solves this with a multi-agent architecture: the main agent can spawn sub-agents, each with its own conversation context, tools, and focus area.

The key insight: **agents are just tools**. The `AgentTool` is a regular tool that happens to create new AI conversations. The model decides when to delegate, just as it decides when to read a file.

## AgentTool: The Entry Point

The multi-agent system begins with `AgentTool`, defined in `src/tools/AgentTool/`. When the model decides a subtask is better handled by a separate agent, it calls `AgentTool` with a prompt describing the work:

```typescript
z.object({
  prompt: z.string().describe("Task description for the sub-agent"),
  subagent_type: z.string().optional(),
  run_in_background: z.boolean().optional(),
})
```

The system then spawns a new agent, lets it run to completion, and returns the agent's final answer as the tool result.

**Key files:**
- `AgentTool.tsx` -- Tool definition and UI rendering
- `runAgent.ts` -- The core agent execution loop
- `forkSubagent.ts` -- Context forking for prompt cache sharing
- `builtInAgents.ts` -- Registry of built-in agent types
- `built-in/` -- Individual agent definitions (explore, plan, verify, etc.)
- `loadAgentsDir.ts` -- Loading custom agents from `.claude/agents/`

## How Agents Spawn: runAgent()

The `runAgent()` function (in `src/agents/runAgent.ts`) is responsible for creating and running a sub-agent. Here is what happens:

```
Model calls AgentTool({ prompt, subagent_type, run_in_background })
  -> loadAgentsDir() finds matching agent definition
  -> createSubagentContext() builds isolated child context
  -> Choose execution mode:
     a. Foreground: runAgent() inline, blocks parent
     b. Background: runAgent() as LocalAgentTask, parent continues
     c. Fork: forkSubagent() inherits parent's prompt cache
  -> Register task in AppState
  -> Return result (or stream progress) to parent
```

1. **Context forking.** A new conversation context is created from the parent's context.
2. **Tool pool selection.** The sub-agent receives a curated set of tools. Sub-agents typically cannot spawn further sub-agents to prevent infinite recursion.
3. **System prompt assembly.** The sub-agent gets its own system prompt, tailored to its role.
4. **Query loop execution.** The sub-agent enters its own `query()` loop -- the same ReAct loop from Chapter 1 -- and runs until it produces a final answer.
5. **Result collection.** The sub-agent's final response is returned to the parent as a tool result.

## Context Forking: forkSubagent()

The `forkSubagent()` function is where the most important optimization lives. It creates a child agent that inherits a **byte-exact copy** of the parent's system prompt and conversation prefix:

```
Parent: [System Prompt][Msg1][Msg2][Msg3]
                        | fork point
Child:  [System Prompt][Msg1][Msg2][New Task Prompt]
        ^ Shared prompt cache -- no re-tokenization!
```

This means a child agent starts with full knowledge of the parent's research, at zero additional token cost for the shared prefix.

**What is inherited:**
- System prompt (byte-exact copy for prompt cache hit)
- File read timestamps (so the sub-agent knows which files were recently read)
- Permission grants (the sub-agent does not re-ask for permissions the user already granted)
- Working directory and project configuration
- Abort signal (if the parent is cancelled, the sub-agent is too)

**What is NOT inherited:**
- Full conversation history beyond the fork point
- Tool results (the sub-agent does not see the parent's tool outputs)
- In-progress state (the sub-agent has its own independent state)

This isolation is critical. A sub-agent working on "find all uses of function X" should not be distracted by the parent's ongoing conversation about refactoring.

## SendMessageTool: Inter-Agent Communication

Sub-agents spawned by `AgentTool` typically run to completion and return their full result. But Claude Code also supports a different model of agent interaction through `SendMessageTool`.

`SendMessageTool` allows agents to send structured messages to each other's "inboxes." This is used in more advanced scenarios where agents need to coordinate rather than simply delegate. The message includes a recipient identifier and a content payload.

Background agents deliver results via XML blocks injected into the message stream:

```xml
<task-notification task_id="abc" status="completed">
  Agent task "Research API" completed.
</task-notification>
```

## Coordinator Mode: The Four Phases

For complex, multi-file tasks, Claude Code can operate in **Coordinator mode**. The workflow follows four phases:

**Phase 1: Fan-Out (Research).** The coordinator spawns multiple sub-agents in parallel to gather information. One agent might search for all references to a function, another reads the test files, a third checks documentation.

**Phase 2: Synthesis (Planning).** The coordinator collects results from all research agents and synthesizes a plan. It writes this plan to a scratchpad -- a shared document that sub-agents can reference.

**Phase 3: Implement.** The coordinator spawns implementation agents, each responsible for a specific part of the plan. These agents read from the scratchpad to understand their role.

**Phase 4: Verify.** A **fresh** agent (with no implementation context) is spawned to review the changes. This freshness prevents confirmation bias -- the verifier has not seen the implementation process.

```
         Coordinator
        /    |    \
       /     |     \
  [Fan-Out Research Agents]     Phase 1: Gather info
       \     |     /
        \    |    /
     [Synthesis/Plan]           Phase 2: Create plan
        /    |    \
       /     |     \
  [Implementation Agents]       Phase 3: Make changes
       \     |     /
        \    |    /
    [Fresh Verification Agent]  Phase 4: Unbiased review
```

## Built-In Agents

Claude Code includes several built-in agent types:

| Agent | Purpose | Tools Available |
|-------|---------|----------------|
| `general-purpose` | Research, code search, multi-step tasks | All tools |
| `Explore` | Fast codebase exploration | Read-only tools |
| `Plan` | Architecture and implementation planning | Read-only tools |
| `verification` | Independent code review | Read-only tools |

The distinction between these agents is primarily about tool selection and system prompt tuning. The underlying mechanism is the same `runAgent()` function. Custom agents can be defined in `.claude/agents/` and loaded dynamically.

## Background Agents

Claude Code supports **background agents** -- agents that run asynchronously while the user continues interacting with the main conversation. Background agents are spawned as Tasks (Chapter 4) and run in a separate execution context.

Key properties:
- They do not block the main conversation
- They report progress through the task system
- Their results are surfaced via `<task-notification>` XML when they complete
- They can be cancelled by the user
- The model can check progress with `TaskGetTool` and read output with `TaskOutputTool`

## Design Takeaways

1. **Delegation is a tool.** Making agent spawning a regular tool call keeps the architecture uniform.
2. **Context isolation prevents confusion.** Sub-agents work best with a clean context focused on their specific task.
3. **Prompt cache sharing saves tokens.** Context forking with byte-exact system prompt copies eliminates re-tokenization costs.
4. **Parallel research, sequential implementation.** Fan out for research (safe, read-only) but serialize implementation (mutations need coordination).
5. **Fresh verification prevents bias.** The verifier must not have seen the implementation process.
6. **Depth limits prevent runaway recursion.** Sub-agents typically cannot spawn further sub-agents.
---
📁 [← Back to Tutorial Index](../../README.en.md#tutorial) | 🌐 [中文版](../../tutorial/zh-CN/03-多智能体系统.md) | [Ch.4 →](04-task-lifecycle.md)
