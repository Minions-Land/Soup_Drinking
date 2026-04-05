🌐 [中文版](../zh-CN/智能体生成模板.md) | [← README](../../README.en.md)  
🗺️ **Navigation**: [← Toolkit Index](../../README.en.md#toolkit) | [Tutorial EN](../../tutorial/en/) | [Reference](../../en/)

---

# Agent Spawning Template

> 🔧 [English](#english) | [中文](#中文)  
> 🗺️ **Navigation**: [← Toolkit Index](../../README.md#toolkit) | [Tutorial](../../tutorial/) | [Reference EN](../../en/) | [Reference ZH](../../zh-CN/)

A practical template for designing agent spawning systems. Spawning is how a parent agent creates child agents to divide work. The spawning mode determines isolation, communication, and lifecycle characteristics.

## Agent Definition Format

Every agent needs a definition that describes its identity, capabilities, and constraints.

```typescript
type AgentDefinition = {
  // Identity
  agentType: string                // Unique name, e.g. "code-reviewer"
  whenToUse: string                // Description for the parent model's tool prompt
  source: 'built-in' | 'user' | 'project' | 'plugin'

  // Capabilities
  tools?: string[]                 // Allowlist of tool names (whitelist)
  disallowedTools?: string[]       // Denylist of tool names (blacklist)
  skills?: string[]                // Skills or plugins to preload

  // Execution
  model?: string                   // Model override, or 'inherit' from parent
  maxTurns?: number                // Maximum agentic turns before forced stop
  effort?: 'low' | 'medium' | 'high'  // Reasoning effort level

  // Permissions
  permissionMode?: 'default' | 'plan' | 'bubble'
  background?: boolean             // Always run in background

  // Context
  getSystemPrompt(): string        // System prompt for the agent
  hooks?: HooksConfig              // Session-scoped hooks
  isolation?: 'worktree'           // Git worktree isolation for file safety

  // Memory
  memory?: 'user' | 'project'     // Persistent memory scope
  initialPrompt?: string           // Prepended to first user turn
}
```

### Markdown Agent File (Recommended for User-Defined Agents)

```markdown
---
name: code-reviewer
description: "Review code changes for bugs, style, and best practices"
tools: [Read, Glob, Grep, Bash]
model: inherit
permissionMode: plan
maxTurns: 50
---

You are a code reviewer. Read the changed files, identify bugs and
style issues, and report findings in a structured format.
Never modify files. Only read and analyze.
```

## Spawning Modes

### Mode 1: Foreground (Synchronous)

```
Parent ──call()──> Agent
  (blocked)        (running)
Parent <──result── Agent
```

- Parent blocks until agent completes.
- Shares parent's abort controller -- if parent is cancelled, agent is too.
- Permission prompts display in the parent's terminal.
- Agent result flows directly back as the tool return value.

**Use when:** You need the result before your next step. The task is a dependency.

```typescript
const result = await spawnAgent({
  type: 'code-reviewer',
  prompt: 'Review the changes in src/auth/',
  mode: 'foreground',
});
// result is available immediately
```

### Mode 2: Background (Asynchronous)

```
Parent ──spawn()──> Agent
  (continues)       (running independently)
  ...               ...
Parent <──notification── Agent (done)
```

- Parent continues immediately after spawning.
- Agent gets its own abort controller -- independent lifecycle.
- Permission prompts are auto-denied (no terminal access). Pre-approve permissions or use 'bubble' mode.
- Results delivered via notification messages injected into the parent's conversation.

**Use when:** You have genuinely independent parallel work. Tests, builds, linting.

```typescript
const taskId = await spawnAgent({
  type: 'test-runner',
  prompt: 'Run the full test suite',
  mode: 'background',
});
// parent continues; notification arrives when agent finishes
```

### Mode 3: Fork (Context Inheritance)

```
Parent ──fork()──> Fork
  (context)         (copy of parent's full context)
  ...               (running in background)
Parent <──notification── Fork (done)
```

- Fork inherits the parent's full conversation history.
- Shares prompt cache with parent -- highly cost-efficient for overlapping context.
- Always runs in background.
- Must use the same model as parent (for cache sharing).
- Permission prompts can bubble up to the parent terminal.

**Use when:** Research tasks that need everything the parent already knows, but whose intermediate work (tool calls, dead ends) should not pollute the parent's context.

```typescript
const taskId = await spawnAgent({
  prompt: 'Investigate null pointer issues in the auth module',
  mode: 'fork',  // no agentType needed -- inherits parent's identity
  name: 'auth-investigation',
});
```

## Context Inheritance Matrix

| Component | Foreground | Background | Fork |
|-----------|-----------|------------|------|
| Conversation messages | Fresh | Fresh | Full copy |
| System prompt | Agent's own | Agent's own | Parent's exact bytes |
| Abort controller | Shared | Independent | Independent |
| File read cache | Fresh | Fresh | Cloned from parent |
| Permission context | Inherited | Inherited + auto-deny | Inherited + bubble |
| Tools | Filtered by agent def | Filtered by agent def | Parent's full set |
| MCP servers | Merged (parent + agent) | Merged | Parent's set |

## Communication Patterns

### Notifications (Background/Fork to Parent)

```xml
<task-notification>
  <task-id>{agentId}</task-id>
  <status>completed|failed|killed</status>
  <summary>Human-readable description of what was accomplished</summary>
  <result>Agent's final text response</result>
  <usage>
    <total_tokens>15230</total_tokens>
    <tool_uses>12</tool_uses>
    <duration_ms>8400</duration_ms>
  </usage>
</task-notification>
```

### Progress Callbacks (During Execution)

```typescript
onProgress({
  toolUseID: context.toolUseId,
  data: {
    type: 'agent_progress',
    agentId: taskId,
    summary: 'Reading configuration files...',
    totalTokens: 1500,
    toolUses: 3,
  },
});
```

### SendMessage (Continue a Completed Agent)

```typescript
sendMessage({
  to: 'agent-a1b2c3',
  message: 'Now fix the bug you found in validate.ts:42',
});
```

### Mailbox (Peer-to-Peer in Swarm/Team)

File-based mailbox at `~/.agent/teams/{team}/inboxes/{agent}.json`. Each message has `from`, `text`, `timestamp`, `read` fields. Agents poll their inbox periodically.

## Decision Tree: Which Mode?

```
Do you need the result before your next step?
|
+-- YES --> FOREGROUND
|           Blocks parent. Result directly available.
|
+-- NO ---> Does the agent need the parent's full conversation context?
            |
            +-- YES --> FORK
            |           Inherits everything. Shares prompt cache.
            |           Must use same model.
            |
            +-- NO ---> BACKGROUND
                        Fresh context. Independent lifecycle.
                        Can use a different (cheaper) model.
```

**Additional factors:**

- Will the agent need permission prompts? Foreground handles them natively. Background auto-denies. Fork can bubble.
- Should multiple agents write to the same files? Use foreground (sequential) to avoid conflicts.
- Is this verification of another agent's work? Use background with a fresh context to avoid confirmation bias.

## Lifecycle Cleanup

Every resource allocated during agent startup must be released in a `finally` block:

```typescript
try {
  // agent execution
} finally {
  disconnectAgentMcpServers();
  unregisterSessionHooks(agentId);
  clearFileReadCache();
  releaseMessageArrayMemory();
  killOrphanedShellTasks(agentId);
  removeAgentStateEntries(agentId);
}
```

Forgetting cleanup leads to memory leaks, orphaned processes, and stale state entries.
---
📁 [← Toolkit Index](../../README.en.md#toolkit) | 🌐 [中文版](../zh-CN/智能体生成模板.md)
