# Multi-Agent System Architecture

> 🗺️ **Navigation**: [← README](../README.md) | [zh-CN version](../zh-CN/02-多智能体.md)  
> 📁 [Reference Index](../README.md#reference) | [Tutorial](../tutorial/) | [Toolkit](../toolkit/)

> Sources: `src/tools/AgentTool/`, `src/coordinator/`, `src/utils/swarm/`, `src/tasks/`

## 1. Architecture Overview

Claude Code implements a hierarchical multi-agent system with three execution modes:

```
┌─────────────────────────────────────────┐
│           Coordinator (Leader)           │
│  coordinatorMode.ts — orchestrates all   │
│  Phases: Research → Synthesis → Impl → Verify │
├──────────┬──────────┬───────────────────┤
│ Worker 1 │ Worker 2 │    Worker N       │
│ (Agent)  │ (Agent)  │    (Agent)        │
│          │          │                   │
│ AgentTool│ AgentTool│ AgentTool         │
│ runAgent │ runAgent │ runAgent          │
└──────────┴──────────┴───────────────────┘
```

### Execution Modes
| Mode | Description | Key File |
|------|-------------|----------|
| **Sub-agent** | Background agent, independent context | `runAgent.ts` |
| **Fork** | Child inherits parent's full context (prompt cache) | `forkSubagent.ts` |
| **In-Process Teammate** | Swarm member sharing process via AsyncLocalStorage | `inProcessRunner.ts` |

## 2. Agent Spawning

### 2.1 AgentTool Call Flow

```
Model calls AgentTool →
  1. loadAgentsDir() → Find matching agent definition
  2. createSubagentContext() → Build child context
  3. Choose mode:
     a. fork (isolation: "worktree") → forkSubagent()
     b. run in background → runAgent() as LocalAgentTask
     c. run foreground → runAgent() inline
  4. Register task in AppState
  5. Stream progress via onProgress callback
  6. Return final result to parent
```

### 2.2 Agent Definitions

Built-in agents (`src/tools/AgentTool/built-in/`):

| Agent | File | Purpose |
|-------|------|---------|
| `general-purpose` | `generalPurposeAgent.ts` | Research, code search, multi-step tasks |
| `Explore` | `exploreAgent.ts` | Fast codebase exploration |
| `Plan` | `planAgent.ts` | Architecture planning |
| `verification` | `verificationAgent.ts` | Check/verify implementation |
| `claude-code-guide` | `claudeCodeGuideAgent.ts` | Answer questions about Claude Code |
| `statusline-setup` | `statuslineSetup.ts` | Configure status line |

Custom agents can be loaded from `.claude/agents/` directory.

### 2.3 Agent Definition Format

```typescript
type AgentDefinition = {
  name: string
  description: string
  model?: string              // Model override
  tools?: string[]            // Allowed tools
  systemPrompt?: string       // Agent-specific instructions
  whenToUse?: string          // Trigger conditions
  alwaysAvailable?: boolean
}
```

## 3. Context Forking (Prompt Cache Optimization)

The key innovation: **forkSubagent** creates a child agent that inherits a **byte-exact copy** of the parent's system prompt and conversation prefix.

```
Parent Agent:
  [System Prompt] [User Msg 1] [Assistant 1] [User 2] [Assistant 2]
                   ↓ fork point
Child Agent (fork):
  [System Prompt] [User Msg 1] [Assistant 1] [New Task Prompt]
  ↑ Shared prompt cache — no re-tokenization!
```

**Why**: This saves massive token costs. The child agent starts with full context of the parent's research without re-processing.

Key implementation (`forkSubagent.ts`):
- Copies `renderedSystemPrompt` from parent context
- Clones `contentReplacementState` for tool result budget sharing
- Creates independent `AbortController` and `FileStateCache`

## 4. Inter-Agent Communication

### 4.1 SendMessageTool

Primary mechanism for agent-to-agent messaging:

```
Agent A → SendMessageTool.call({ to: "agent-id", message: "..." })
  → Finds target agent's task
  → Injects message into target's inbox
  → Target agent picks up on next poll
```

### 4.2 Task Notifications (XML blocks)

Asynchronous background results delivered as XML:

```xml
<task-notification task_id="abc123" status="completed">
  Agent task "Research API" completed.
</task-notification>
```

### 4.3 Permission Bubbling

When a sub-agent needs user permission:
1. Sub-agent's `checkPermissions` returns `{ behavior: 'ask' }`
2. Request bubbles up to leader's UI
3. User approves/denies
4. Response routes back to sub-agent

## 5. Coordinator Mode

The coordinator orchestrates a multi-phase workflow:

```
Phase 1: RESEARCH
  → Spawn parallel research agents
  → Each explores different code areas
  → Results collected

Phase 2: SYNTHESIS
  → Coordinator analyzes all research
  → Creates detailed implementation spec
  → Identifies files, dependencies, risks

Phase 3: IMPLEMENTATION
  → Spawn implementation agent(s)
  → Execute the synthesized spec

Phase 4: VERIFICATION
  → Spawn fresh verification agent
  → Independent code review (no rubber-stamping)
  → Run tests, check for regressions
```

Key: The verifier is a **fresh agent** that hasn't seen the implementation, preventing confirmation bias.

## 6. Swarm / Team Mode

For parallel execution with shared state:

```typescript
// src/utils/swarm/
├── spawnUtils.ts            // Launch teammates
├── inProcessRunner.ts       // Stay-alive loop for in-process agents
├── permissionSync.ts        // Sync permissions across agents
├── teammateModel.ts         // Model selection for teammates
├── reconnection.ts          // Handle disconnects
├── backends/
│   ├── TmuxBackend.ts       // Spawn in tmux panes
│   ├── ITermBackend.ts      // Spawn in iTerm tabs
│   └── InProcessBackend.ts  // Spawn in same Node process
```

### Team Management
- `TeamCreateTool` — Create a team of agents
- `TeamDeleteTool` — Dissolve a team
- `TaskListTool` — Shared work queue (agents "claim" tasks)
- `permissionSync.ts` — Leader's permission rules propagate to workers

## 7. Agent Memory

Agents maintain memory snapshots for context preservation:

```typescript
// agentMemory.ts
- Captures conversation state at key points
- Allows resumed agents to reconstruct context

// agentMemorySnapshot.ts
- Serializes agent state for persistence
- Used in crash recovery and session resume
```

## 8. Key Design Patterns for Reuse

### Pattern 1: Fan-Out Research
```
Leader → spawn N parallel agents
       → each searches different area
       → collect results → synthesize
```

### Pattern 2: Fork-Execute-Return
```
Parent → fork child (inherits context)
       → child executes specific task
       → returns result to parent
       → parent continues with result
```

### Pattern 3: Shared Work Queue
```
Leader → posts tasks to TaskList
       → Workers poll and claim tasks
       → Workers execute independently
       → Results aggregate in shared state
```

### Pattern 4: Two-Layer Verification
```
Agent A → implements changes
Agent B (fresh, no context) → verifies changes
→ Prevents confirmation bias
```

### Pattern 5: AsyncLocalStorage Isolation
```
Multiple agents share Node.js process
Each has isolated context via AsyncLocalStorage
No manual context passing through function signatures
```
