# Reusable Design Patterns

> 🗺️ **Navigation**: [← README](../README.md) | [zh-CN version](../zh-CN/09-设计模式.md)  
> 📁 [Reference Index](../README.md#reference) | [Tutorial](../tutorial/) | [Toolkit](../toolkit/)

> Key architectural patterns from Claude Code that you can directly copy/adapt for your own agent systems.

## Pattern 1: Tool Interface with Safe Defaults

**Problem**: Many tools share boilerplate (permissions, validation, rendering).
**Solution**: `buildTool()` provides safe defaults; tools only override what they need.

```typescript
// Define only what's unique
const MyTool = buildTool({
  name: 'MyTool',
  inputSchema: z.object({ ... }),
  maxResultSizeChars: 100_000,
  call: async (args, ctx) => ({ data: result }),
  prompt: async () => "Instructions for the model...",
  renderToolUseMessage: (input) => <Text>...</Text>,
  // Everything else gets safe defaults:
  // isEnabled: true, isConcurrencySafe: false,
  // isReadOnly: false, checkPermissions: allow
})
```

**When to use**: Any tool/plugin system where you want consistency without boilerplate.

---

## Pattern 2: Context Forking for Prompt Cache Sharing

**Problem**: Child agents need parent context but re-tokenizing is expensive.
**Solution**: Fork child with byte-exact copy of parent's system prompt and conversation prefix.

```
Parent's context:
  [System Prompt][Msg1][Msg2][Msg3]
                  ↓ fork at Msg2
Child's context:
  [System Prompt][Msg1][Msg2][New Task]
  ↑ Shared prompt cache — zero re-tokenization cost
```

**Key insight**: Copy `renderedSystemPrompt` from parent. Clone `contentReplacementState`. Create independent `AbortController`.

**When to use**: Multi-agent systems where child agents need parent research context.

---

## Pattern 3: Fan-Out Research → Synthesis → Implementation

**Problem**: Complex tasks need multiple perspectives before implementation.
**Solution**: Four-phase coordinator workflow.

```
Phase 1: RESEARCH (parallel)
  Agent A → searches backend code
  Agent B → searches frontend code
  Agent C → reads documentation

Phase 2: SYNTHESIS (coordinator)
  Coordinator → analyzes all results
  → creates detailed implementation spec

Phase 3: IMPLEMENTATION (targeted)
  Agent D → executes the synthesized spec

Phase 4: VERIFICATION (fresh agent)
  Agent E (no prior context) → reviews changes
  → runs tests → checks for regressions
```

**Critical**: The verification agent must be **fresh** (no implementation context) to avoid confirmation bias.

---

## Pattern 4: Two-Stage Permission Validation

**Problem**: Need both logical validation and authorization checks.
**Solution**: Separate `validateInput` (logic) from `checkPermissions` (auth).

```
Stage 1: validateInput(input, context)
  → "Does this file exist?"
  → "Is this a valid regex?"
  → Returns { result: true } or { result: false, message }

Stage 2: checkPermissions(input, context)
  → "Is the user allowing file writes?"
  → "Does this match an always-allow rule?"
  → Returns { behavior: 'allow' | 'deny' | 'ask' }
```

**When to use**: Any system where tools need both validity checks and authorization.

---

## Pattern 5: AsyncLocalStorage for Agent Isolation

**Problem**: Multiple agents running in the same Node.js process need isolated context.
**Solution**: Use `AsyncLocalStorage` to maintain per-agent state without manual threading.

```typescript
const agentContext = new AsyncLocalStorage<AgentState>()

// Each agent runs in its own context
agentContext.run(agentState, async () => {
  // All nested calls see this agent's state
  const state = agentContext.getStore()
  await doWork()  // No need to pass state explicitly
})
```

**When to use**: In-process multi-agent systems, request-scoped state in Node.js.

---

## Pattern 6: Progressive Result Delivery

**Problem**: Long-running tools block the UI.
**Solution**: Progress callback system with typed progress events.

```typescript
type ToolCallProgress<P> = (progress: ToolProgress<P>) => void

// In tool implementation:
async call(args, ctx, canUse, msg, onProgress) {
  onProgress?.({ toolUseID: id, data: { type: 'bash', status: 'running' } })
  // ... do work ...
  onProgress?.({ toolUseID: id, data: { type: 'bash', output: '50% done' } })
  return { data: result }
}
```

**When to use**: Any tool that takes more than a few seconds.

---

## Pattern 7: Deferred Tool Loading

**Problem**: Too many tools in the system prompt wastes tokens.
**Solution**: Mark rarely-used tools as `shouldDefer: true`. Model discovers them via `ToolSearchTool`.

```
Turn 1: Model has 15 core tools + ToolSearchTool
Turn 2: Model calls ToolSearchTool("jupyter notebook")
  → Finds NotebookEditTool (deferred)
  → Tool schema loaded for next turn
Turn 3: Model can use NotebookEditTool
```

Tools provide `searchHint` for keyword matching.

**When to use**: Systems with 20+ tools where prompt space is constrained.

---

## Pattern 8: XML-Based Async Notifications

**Problem**: Background agents need to notify the main conversation.
**Solution**: Inject XML-tagged notifications into the message stream.

```xml
<task-notification task_id="abc123" status="completed">
  Agent task "Research API endpoints" completed.
</task-notification>
```

**When to use**: Asynchronous multi-agent communication where results arrive at unpredictable times.

---

## Pattern 9: Shared Work Queue (Swarm)

**Problem**: Multiple agents need to coordinate without a central bottleneck.
**Solution**: Shared task list where agents autonomously claim work.

```
Leader: Post tasks to TaskList
Worker 1: Poll → Claim "Task A" → Execute → Complete
Worker 2: Poll → Claim "Task B" → Execute → Complete
Worker 3: Poll → Claim "Task C" → Execute → Complete
```

No leader micro-management needed. Workers are self-directed.

**When to use**: Embarrassingly parallel workloads (testing across files, searching multiple repos).

---

## Pattern 10: Command Registry with Multi-Source Aggregation

**Problem**: Commands come from different sources (built-in, plugins, skills).
**Solution**: Central registry that discovers and filters commands.

```
Registry.getCommands() →
  BuiltIn (hardcoded) +
  Skills (prompt files) +
  Plugins (dynamic) +
  Workflows (scriptable)
  → Filter by auth level, mode, safety
  → Return unified command list
```

**When to use**: Extensible CLI/chat systems with multiple extension points.

---

## Pattern 11: GC-Free Rendering with Packed Buffers

**Problem**: Terminal UI rendering causes GC pauses.
**Solution**: Use typed arrays (Int32Array) instead of objects for cell buffers.

```
Each cell = 2 × Int32:
  Word 0: Interned character ID
  Word 1: Packed bitfields (FG, BG, attrs, width)
```

Intern pools for characters, styles, URIs. Zero object allocation per frame.

**When to use**: High-frequency terminal rendering where GC pauses are noticeable.

---

## Pattern 12: Coordinator-Worker with Scratchpad

**Problem**: Workers need to share findings without passing through conversation.
**Solution**: Shared filesystem directory (scratchpad) accessible to all agents.

```
Coordinator → Creates scratchpad dir
Worker A → Writes findings to scratchpad/research-a.md
Worker B → Reads scratchpad/research-a.md, writes scratchpad/research-b.md
Coordinator → Reads all scratchpad files → Synthesizes
```

Path injected via coordinator context. Durable across agent lifetimes.

**When to use**: Multi-agent systems with large intermediate artifacts.
