🌐 [中文版](../../tutorial/zh-CN/09-设计模式总结.md) | [← README](../../README.en.md)  
🗺️ **Navigation**: [← Ch.8](08-remote-bridge.md) | **Chapter 9** | [← README](../../README.en.md)  
📁 [Tutorial EN](../../tutorial/en/) | [Toolkit EN](../../toolkit/en/) | [Reference](../../en/)

---

# Chapter 9: 12 Reusable Design Patterns -- Your Agent Design Cookbook

> Previous: [Chapter 8: Remote Connection](08-remote-bridge.md)

Throughout this tutorial series, we have dissected Claude Code's architecture -- its tool system, multi-agent coordination, permission model, terminal UI, command registry, and remote bridge. Along the way, we encountered recurring design patterns that solve fundamental problems in agent engineering.

This chapter distills those patterns into a reference cookbook. Each pattern is presented with the problem it solves, the solution, when to use it, and a key insight. Think of this as your cheat sheet for building production-grade agent systems.

---

## Pattern 1: Tool Interface with Safe Defaults

**Problem**: When your system has 30+ tools, each tool needs permission checking, input validation, result rendering, concurrency flags, and error handling. Writing all of this for every tool creates massive boilerplate and inconsistency.

**Solution**: Create a `buildTool()` factory function that fills in safe defaults for everything. Tools only override what they need. The factory provides: `isEnabled: true`, `isConcurrencySafe: false`, `isReadOnly: false`, `checkPermissions: ask-user`.

**When to use**: Any system with more than a handful of tools or plugins that need consistent behavior.

**Key insight**: The defaults should be fail-closed. A tool that forgets to declare `isReadOnly` defaults to `false` (assumed dangerous). A tool that forgets `checkPermissions` defaults to asking the user. Safety by default, flexibility by override.

---

## Pattern 2: Context Forking for Prompt Cache Sharing

**Problem**: In a multi-agent system, child agents often need the parent's research context. But re-tokenizing the parent's conversation for each child is expensive in both time and API costs.

**Solution**: Fork the child agent with a byte-exact copy of the parent's system prompt and conversation prefix. Because the bytes are identical, the LLM provider's prompt cache is reused -- zero re-tokenization cost.

```
Parent: [System Prompt][Msg1][Msg2][Msg3]
                        | fork
Child:  [System Prompt][Msg1][Msg2][New Child Task]
        ^-- Shared cache, zero re-tokenization
```

**When to use**: Multi-agent systems where child agents need access to research context accumulated by the parent.

**Key insight**: Copy `renderedSystemPrompt` from the parent. Clone `contentReplacementState`. But create an independent `AbortController` -- the child must be cancellable without killing the parent.

---

## Pattern 3: Fan-Out Research, Synthesis, Implementation, Verification

**Problem**: Complex tasks require understanding from multiple perspectives before implementation. A single agent doing everything sequentially is slow and produces tunnel vision.

**Solution**: A four-phase coordinator workflow. Phase 1: parallel research agents gather information (read-only, safe to parallelize). Phase 2: coordinator synthesizes findings into a spec. Phase 3: targeted agent implements the spec. Phase 4: a fresh verification agent reviews.

**When to use**: Any task that requires understanding a system before modifying it, especially cross-cutting changes.

**Key insight**: The verification agent must be **fresh** -- no context from the implementation phase. If the verifier has seen the implementation reasoning, it suffers from confirmation bias and is less likely to catch errors.

---

## Pattern 4: Two-Stage Permission Validation

**Problem**: Before executing a tool, you need to check "Is this input valid?" and "Is the user allowing this action?" These are separate concerns that many systems conflate, producing confusing error messages.

**Solution**: Split validation into two explicit stages. Stage 1: `validateInput()` checks logic -- "Does this file exist? Is this a valid regex?" Stage 2: `checkPermissions()` checks authorization -- "Is the user allowing file writes? Does this match an always-allow rule?" The first returns error messages; the second returns `allow`, `deny`, or `ask`.

**When to use**: Any system where tools need both validity checking and authorization.

**Key insight**: Validity is objective (the file exists or it does not). Authorization is policy (the user allows it or not). Separating them produces better error messages: "file not found" vs. "permission denied" rather than a generic "operation failed."

---

## Pattern 5: AsyncLocalStorage for Agent Isolation

**Problem**: Multiple agents run concurrently in the same Node.js process. Each needs its own state (conversation history, tool configuration, abort signals). Passing state through every function parameter is tedious and error-prone.

**Solution**: Use Node.js `AsyncLocalStorage` to create per-agent context that flows automatically through async call chains. Any code, no matter how deeply nested, can retrieve the current agent's state without it being passed as a parameter.

**When to use**: In-process multi-agent systems, or any Node.js application that needs request-scoped state.

**Key insight**: `AsyncLocalStorage` eliminates the need to pass context through every function signature. This is the same mechanism Express uses for request-scoped data, but applied to agent isolation. Zero manual context threading.

---

## Pattern 6: Progressive Result Delivery

**Problem**: Some tools take seconds or minutes to complete (running tests, searching large codebases). The UI shows nothing, leaving the user wondering if the system is frozen.

**Solution**: A typed progress callback system (`ToolCallProgress<P>`) that tools use to report incremental updates. The UI renders these progressively; the final result replaces them.

**When to use**: Any tool that takes more than a few seconds to complete.

**Key insight**: Progress updates serve double duty. For the UI, they show work is happening. For the system, they can carry partial results that the model can reason about before the tool finishes -- enabling early course correction.

---

## Pattern 7: Deferred Tool Loading

**Problem**: Your agent has 30+ tools, but only 10-15 are commonly used. Including all tool schemas in the system prompt wastes thousands of tokens on tools the model rarely needs.

**Solution**: Mark infrequently-used tools with `shouldDefer: true`. They are omitted from the initial system prompt. A lightweight `ToolSearchTool` is included instead. When the model needs a specialized tool, it searches by keyword, and the tool schema is loaded for the next turn.

**When to use**: Systems with 20+ tools where prompt space is at a premium.

**Key insight**: Each deferred tool provides a `searchHint` -- keywords for matching. The model pays a one-turn latency cost to discover a tool, but saves tokens on every turn where the tool is not needed. This is essentially a search index over tool capabilities.

---

## Pattern 8: XML-Based Async Notifications

**Problem**: Background agents complete work at unpredictable times. The main conversation needs to know when results arrive, but you cannot interrupt the model mid-generation.

**Solution**: Inject XML-tagged notifications into the message stream between turns. The model sees these in its conversation context and acts on them naturally.

```xml
<task-notification task_id="abc123" status="completed">
  Agent task "Research API endpoints" completed.
</task-notification>
```

**When to use**: Any multi-agent system where background work produces results asynchronously.

**Key insight**: XML is ideal because it is simultaneously parseable by code (for routing and status tracking) and readable by the model (for understanding what happened). No format conversion needed.

---

## Pattern 9: Shared Work Queue (Swarm)

**Problem**: Multiple agents need to work on a large set of tasks. A single coordinator dispatching tasks one-by-one creates a bottleneck.

**Solution**: A shared task list where agents autonomously claim and complete work. The leader posts tasks; workers poll, claim, execute, and mark complete. No micromanagement needed.

**When to use**: Embarrassingly parallel workloads where tasks are independent (testing across files, searching multiple repositories, processing many items).

**Key insight**: Workers are self-directed. The leader only defines the task list and waits for completion. This eliminates the coordinator as a bottleneck and scales linearly with workers.

---

## Pattern 10: Multi-Source Command Registry

**Problem**: Commands come from different sources (hardcoded built-ins, user-installed skills, third-party plugins, custom workflows). You need a unified way to discover, filter, and invoke them.

**Solution**: A central registry that aggregates commands from all sources, deduplicates by name, and applies context-aware filters (auth level, mode, safety). Returns a unified command list.

**When to use**: Any extensible CLI, chatbot, or agent system with multiple extension points.

**Key insight**: The registry does not care where a command comes from. It only cares that each command implements the standard interface. This decouples the core from extensions and lets anyone add commands without modifying core code.

---

## Pattern 11: GC-Free Packed Buffer Rendering

**Problem**: Terminal UI re-renders many times per second. Each render creates thousands of JavaScript objects. Garbage collector pauses become noticeable, causing jank.

**Solution**: Replace object-per-cell rendering with typed arrays. Each cell is two `Int32` values in an `Int32Array`: one for the interned character ID, one for packed bitfields (foreground, background, attributes, width). Characters and styles are interned in pools. Zero object allocation per frame.

**When to use**: High-frequency terminal or canvas rendering where GC pauses are visible.

**Key insight**: This is a classic game-engine technique applied to terminal rendering. The tradeoff is code complexity (bit manipulation vs. property access), but for hot render paths the performance gain is dramatic.

---

## Pattern 12: Coordinator-Worker with Scratchpad

**Problem**: In a multi-agent system, workers produce large intermediate artifacts (research summaries, code analysis, test results). Passing these through conversation context wastes tokens and hits context limits.

**Solution**: Create a shared filesystem directory (the "scratchpad") accessible to all agents. Workers write findings to files; the coordinator reads them for synthesis. The scratchpad path is injected into each agent's context.

**When to use**: Multi-agent systems where intermediate artifacts are too large for conversation context, or where workers need to share findings with each other.

**Key insight**: Files persist across agent lifetimes. A worker can crash, be restarted, and pick up where it left off by reading the scratchpad. This is more durable than passing data through conversation turns and sidesteps context window limits entirely.

---

## Putting It All Together

These 12 patterns are not isolated techniques. In a production system like Claude Code, they work together:

- **Tools** use Pattern 1 (safe defaults) and Pattern 4 (two-stage validation)
- **Multi-agent orchestration** combines Pattern 2 (context forking), Pattern 3 (fan-out/synthesis), Pattern 5 (AsyncLocalStorage), and Pattern 12 (scratchpad)
- **The command system** implements Pattern 10 (multi-source registry) with Pattern 7 (deferred loading)
- **The terminal UI** uses Pattern 6 (progressive delivery) and Pattern 11 (GC-free rendering)
- **Background tasks** rely on Pattern 8 (XML notifications) and Pattern 9 (work queues)

Each pattern solves a specific, recurring problem. When you encounter that problem in your own system, you do not need to reinvent the solution -- reach into this cookbook.

> This concludes the tutorial series on Agent Design Patterns from Claude Code.
---
📁 [← Tutorial Index](../../README.en.md#tutorial) | 🌐 [中文版](../../tutorial/zh-CN/09-设计模式总结.md) | [← README](../../README.en.md)
