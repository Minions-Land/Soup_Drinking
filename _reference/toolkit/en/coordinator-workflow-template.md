🌐 [中文版](../../toolkit/zh-CN/协调器工作流模板.md) | [← README](../../README.en.md)  
🗺️ **Navigation**: [← Toolkit Index](../../README.en.md#toolkit) | [Tutorial EN](../../tutorial/en/) | [Reference](../../en/)

---

# Coordinator Workflow Template

> 🔧 [English](#english) | [中文](#中文)  
> 🗺️ **Navigation**: [← Toolkit Index](../README.md#toolkit) | [Tutorial](../tutorial/) | [Reference EN](../en/) | [Reference ZH](../zh-CN/)

A practical template for building coordinator workflows -- the pattern where a lead agent orchestrates multiple sub-agents through a structured multi-phase process. This is the most common pattern for complex tasks that require research, synthesis, implementation, and verification.

## The 4-Phase Pattern

```
Phase 1: RESEARCH        Phase 2: SYNTHESIS       Phase 3: IMPLEMENTATION   Phase 4: VERIFICATION
+------------------+     +------------------+     +-------------------+     +-------------------+
| Fan-out to N     |     | Coordinator      |     | Coordinator or    |     | Fresh agent       |
| research agents  | --> | reads all results| --> | delegate agents   | --> | verifies the work |
| (parallel)       |     | builds a plan    |     | execute the plan  |     | (no prior context)|
+------------------+     +------------------+     +-------------------+     +-------------------+
```

### Phase 1: Research (Fan-Out)

The coordinator spawns multiple agents in parallel to investigate different aspects of the problem. Each research agent gets a specific, focused question.

```typescript
// Spawn research agents in parallel
const researchTasks = [
  spawnAgent({
    type: 'researcher',
    prompt: 'Find all files that import the auth module. List each file and the specific imports used.',
    mode: 'background',
    name: 'auth-imports',
  }),
  spawnAgent({
    type: 'researcher',
    prompt: 'Read the test files for auth. List every test case and whether it tests success or failure paths.',
    mode: 'background',
    name: 'auth-tests',
  }),
  spawnAgent({
    type: 'researcher',
    prompt: 'Search for all error handling patterns in src/. Categorize by try/catch vs Result type vs callbacks.',
    mode: 'background',
    name: 'error-patterns',
  }),
];
// All three run simultaneously. Coordinator waits for notifications.
```

**Key principles for research fan-out:**

- Each agent gets ONE specific question, not a vague directive.
- Agents return structured findings, not opinions or recommendations.
- Use background mode -- research agents are independent.
- Spawn all research agents in a single coordinator turn (one message, multiple tool calls).
- Never ask a research agent to "fix" anything. Research only.

### Phase 2: Synthesis

After all research notifications arrive, the coordinator reads the results and builds a concrete plan. This is where the coordinator's reasoning is most critical.

```
Coordinator receives notifications and reasons:

"Based on the research:
 - auth-imports found 12 files importing auth
 - auth-tests found 8 test cases, all testing success paths only
 - error-patterns found that 90% of the codebase uses Result types

 Plan:
 1. Refactor auth module to return Result<T, AuthError> instead of throwing
 2. Update all 12 importing files to handle the Result type
 3. Add 6 new test cases for failure paths
 4. Files to modify: [specific list with exact changes per file]"
```

**Key principles for synthesis:**

- The coordinator does the synthesis itself -- never delegate understanding to a sub-agent.
- Write a specific, file-by-file plan. Vague plans produce vague implementations.
- Reference specific findings from research agents by name.
- Identify dependencies between implementation steps.
- Write the plan to a shared scratchpad file for traceability.

### Phase 3: Implementation

The coordinator executes the plan, either directly or by delegating to specialized agents.

```typescript
// Option A: Coordinator implements directly (preferred for < 5 files)
await editFile('src/auth/index.ts', changes);
await editFile('src/auth/types.ts', changes);

// Option B: Delegate to implementation agents (for larger changes)
await spawnAgent({
  type: 'implementer',
  prompt: `Refactor these files to use Result types:
    - src/api/login.ts: change catch block to match on Result
    - src/api/register.ts: change catch block to match on Result
    Exact changes described in /tmp/scratchpad/plan.md`,
  mode: 'foreground',  // sequential to avoid file conflicts
});
```

**Key principles for implementation:**

- For small changes (under 5 files), the coordinator implements directly. Spawning agents for trivial edits wastes tokens and adds latency.
- For large changes, delegate but be SPECIFIC. Include exact file paths, exact changes, exact patterns to follow.
- Use foreground mode for implementation agents to avoid file write conflicts.
- Sequential implementation agents, not parallel, when they touch overlapping files.

### Phase 4: Verification

A fresh agent -- one that did NOT participate in research or implementation -- verifies the work.

```typescript
const verifier = await spawnAgent({
  type: 'verifier',
  prompt: `Verify the auth refactoring is complete and correct:
    1. Run: npm test
    2. Check that all 12 files listed in /tmp/scratchpad/plan.md were modified
    3. Grep for any remaining throw statements in src/auth/
    4. Report: PASS or FAIL with specific issues`,
  mode: 'background',  // fresh context = no confirmation bias
});
```

**Why a fresh agent?** An agent that did the implementation is biased toward believing its work is correct. It will gloss over issues because it "knows" what it intended. A fresh agent with no prior context will actually run tests, grep for problems, and report honestly. This is the single most important insight in the coordinator pattern.

## Scratchpad: Shared State Between Phases

Use a temporary scratchpad directory for inter-phase communication:

```
/tmp/scratchpad/{session-id}/
  research/
    auth-imports.md       # Research agent 1 output
    auth-tests.md         # Research agent 2 output
    error-patterns.md     # Research agent 3 output
  plan.md                 # Coordinator's synthesis plan
  progress.md             # Implementation progress tracking
  verification-report.md  # Verifier's final report
```

**Why files instead of message passing?**

- Files persist across agent lifecycles. Messages are lost when agents terminate.
- Files can be read by any agent at any time. Messages are scoped to parent-child relationships.
- Files provide an audit trail. You can review what each agent found and decided.
- Files avoid context window bloat. A 10-page research report in a message wastes tokens on every subsequent turn.

## Result Aggregation

When multiple research agents return, aggregate their findings systematically:

```typescript
function aggregateResearchResults(notifications: TaskNotification[]) {
  const completed = notifications.filter(n => n.status === 'completed');
  const failed = notifications.filter(n => n.status === 'failed');
  const findings: Record<string, { summary: string; result: string }> = {};

  for (const n of completed) {
    findings[n.taskId] = { summary: n.summary, result: n.result };
  }

  // Handle failures: log and proceed with available data
  if (failed.length > 0) {
    for (const f of failed) {
      console.warn(`Research agent ${f.taskId} failed: ${f.error}`);
    }
    // Decide: retry? proceed without? abort?
    // Usually: proceed with available data unless the failed agent was critical.
  }

  return { findings, failedCount: failed.length };
}
```

## Decision Framework: When to Use the Coordinator Pattern

```
Is the task decomposable into independent research questions?
  NO  --> Single agent is sufficient. Do not over-engineer.
  YES --> How many files/components are involved?
    < 3   --> Single agent with good prompting
    3-10  --> Coordinator with 2-3 research agents
    > 10  --> Full 4-phase coordinator with fan-out and delegated implementation
```

## Common Variations

### Research-Only Coordinator

Skip phases 3 and 4. Use when the goal is understanding, not modification:

```
Research --> Synthesis --> Report
```

### Implementation-Heavy Coordinator

Minimal research, heavy implementation. Use when the plan is already known:

```
(Minimal Research) --> Implementation --> Verification
```

### Iterative Coordinator

Loop through phases until verification passes:

```
Research --> Synthesis --> Implementation --> Verification
                ^                                |
                |         (if FAIL, loop)        |
                +--------------------------------+
```

Limit iterations (2-3 max) to prevent infinite loops and runaway token consumption.

## Anti-Patterns

| Anti-Pattern | Problem | Solution |
|-------------|---------|----------|
| Delegating synthesis | "Based on your findings, fix it" produces shallow work | Coordinator reads findings, writes specific plan |
| Parallel implementation on same files | Write conflicts, lost changes | Sequential foreground agents for overlapping files |
| Verifier with inherited context | Confirmation bias, rubber-stamping | Fresh background agent with no prior context |
| Vague research prompts | "Investigate the auth system" yields unfocused results | "List all files importing auth and their specific imports" |
| Skipping verification | Bugs ship to production undetected | Always verify with a fresh agent |
| Too many phases | Overhead exceeds benefit for simple tasks | Only use 4 phases for genuinely complex tasks |
---
📁 [← Toolkit Index](../../README.en.md#toolkit) | 🌐 [中文版](../../toolkit/zh-CN/协调器工作流模板.md)
