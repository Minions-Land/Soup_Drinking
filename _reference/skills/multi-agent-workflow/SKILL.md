---
name: multi-agent-workflow
description: "Design a multi-agent workflow using Claude Code's coordinator/swarm patterns — outputs a complete orchestration spec"
---

<!--
# /multi-agent-workflow — Multi-Agent Workflow Designer
# /multi-agent-workflow — 多智能体工作流设计器

PURPOSE / 用途:
  Given a complex task description, designs a complete multi-agent workflow
  using Claude Code's proven coordination models (Coordinator, Swarm, Fork).

  给定复杂任务描述，使用 Claude Code 经过验证的协调模型（协调者、Swarm、Fork）
  设计完整的多智能体工作流。

USAGE / 用法:
  /multi-agent-workflow <complex task description>
  /multi-agent-workflow <复杂任务描述>

EXAMPLES / 示例:
  /multi-agent-workflow Analyze a large codebase, find all security vulnerabilities, generate a fix for each
  /multi-agent-workflow 分析大型代码库，找出所有安全漏洞，为每个漏洞生成修复方案

  /multi-agent-workflow Research 10 competing libraries, benchmark them, and write a comparison report
  /multi-agent-workflow 研究10个竞争库，对它们进行基准测试，并撰写比较报告

  /multi-agent-workflow Migrate a monolithic codebase to microservices, file by file, with testing
  /multi-agent-workflow 将单体代码库逐文件迁移到微服务架构，并进行测试

COORDINATION MODELS / 协调模型:
  Model A: Coordinator + Workers — sequential phases, Research→Synthesis→Implement→Verify
  模型 A: 协调者 + 工作者 — 顺序阶段，研究→综合→实现→验证

  Model B: Swarm — parallel autonomous workers, shared work queue
  模型 B: Swarm — 并行自主工作者，共享工作队列

  Model C: Fork — child inherits parent context (prompt cache optimization)
  模型 C: Fork — 子级继承父级上下文（Prompt 缓存优化）

REFERENCE FILES READ / 读取的参考文件:
  - toolkit/coordinator-workflow-template.md
  - toolkit/swarm-team-template.md
  - toolkit/agent-spawning-template.md
  - en/02-MULTI-AGENT.md
-->

# /multi-agent-workflow — Multi-Agent Workflow Designer

Usage: `/multi-agent-workflow <complex task description>`

---

## Your Role

Design a complete multi-agent workflow using Claude Code's proven patterns. Select the right coordination model, define each agent's role precisely, and specify how they communicate and recover from failures.

---

## Step 1 — Read the reference files

Read in this order:
1. `/Users/mjm/Claude Code/_reference/toolkit/coordinator-workflow-template.md`
2. `/Users/mjm/Claude Code/_reference/toolkit/swarm-team-template.md`
3. `/Users/mjm/Claude Code/_reference/toolkit/agent-spawning-template.md`

---

## Step 2 — Select coordination model

**Model A: Coordinator + Workers**
- Task has distinct phases (research, then synthesize, then implement, then verify)
- Each phase depends on the previous one
- Need anti-confirmation-bias: fresh verifier sees no implementation context
- Template: `coordinator-workflow-template.md`

**Model B: Swarm**
- Task is embarrassingly parallel (N independent items to process)
- Workers don't depend on each other's output
- Prefer autonomy: workers self-claim work from a shared queue
- Template: `swarm-team-template.md`

**Model C: Fork (Context Inheritance)**
- Child needs parent's full research context immediately
- Want to share prompt cache (zero re-tokenization cost)
- Sequential subtasks that build on parent's knowledge
- Template: `agent-spawning-template.md`

**Hybrid**: Most production systems combine models. E.g., Coordinator orchestrates Phase 1 with fan-out research (Model A), then Phase 2 as a Swarm for parallel implementation (Model B).

---

## Step 3 — Produce the workflow design document

````markdown
## Workflow Design: [Name]

### Coordination Model
**Primary model**: [A / B / C / Hybrid]
**Reason**: [why this model fits the task]

---

### Agent Roster

| Agent | Role | Tools Allowed | Spawn Mode | Output |
|-------|------|--------------|------------|--------|
| Coordinator | Orchestrate all phases | All (read-heavy) | Main thread | Synthesis doc |
| Research-A | [specific area] | Glob, Grep, Read | Background | `scratchpad/research-a.md` |
| Research-B | [specific area] | Glob, Grep, Read | Background | `scratchpad/research-b.md` |
| Implementer | Execute synthesis spec | All | Fork from Coordinator | File changes |
| Verifier | Independent review | Read-only | Fresh (no prior context) | Verdict report |

---

### Communication Design

**Coordinator → Workers**: [how tasks are assigned]
```
Option 1: Direct spawn — AgentTool({ prompt: task_description, run_in_background: true })
Option 2: Shared queue — TaskList.add(task), workers poll and claim
Option 3: SendMessage — to: "agent-id", message: "new task"
```
**Choice**: [which option and why]

**Workers → Coordinator**: [how results return]
```
Option 1: <task-notification> XML injected into coordinator's stream
Option 2: Write to scratchpad/[agent-name].md, coordinator reads all
Option 3: TaskOutputTool(task_id, block=true)
```
**Choice**: [which option and why]

**Shared State**: [cross-agent knowledge]
```
Scratchpad directory: .claude/scratchpad/[workflow-name]/
  ├── research-findings.md
  ├── synthesis.md
  └── implementation-log.md
```

---

### Phase Breakdown

**Phase 1: Research** *(N agents, parallel, ~X minutes)*
```
Coordinator spawns simultaneously:
  Agent "research-backend" → [specific search area, specific tools]
  Agent "research-frontend" → [specific search area, specific tools]
  Agent "research-tests" → [specific search area, specific tools]

Collection: Wait for all <task-notification> completions
Result: scratchpad/research-*.md files
```

**Phase 2: Synthesis** *(Coordinator only)*
```
Read all scratchpad/research-*.md
Produce: scratchpad/implementation-spec.md
  Contains: [list of specific changes with file paths and exact modifications]
```

**Phase 3: Implementation** *(1-N agents)*
```
[If one agent]: Fork from coordinator → inherits context → executes spec
[If multiple]: Swarm — coordinator splits spec into chunks, workers claim chunks
```

**Phase 4: Verification** *(1 FRESH agent — CRITICAL)*
```
⚠️  MUST be a fresh agent with NO implementation context
Only receives: original requirements + file diff
Checks:
  - [ ] Requirements are met
  - [ ] No regressions introduced
  - [ ] Tests pass
  - [ ] Code quality acceptable
```

---

### Prompt Cache Strategy

| Agent | Fork from? | Shared prefix | Savings |
|-------|-----------|---------------|---------|
| Research agents | No (independent context) | 0 | — |
| Implementer | Yes, from Coordinator | ~[N]k tokens | ~$[X] |
| Verifier | No (fresh — intentional) | 0 | — |

---

### Permission Model

```
Coordinator permissions: [list]
Workers inherit via permission sync: [list]
Bubble up to user for: [sensitive operations]
```

---

### Failure Recovery

| Failure scenario | Detection | Recovery |
|-----------------|-----------|----------|
| Research agent crashes | No task-notification after timeout | Re-spawn with same task |
| Coordinator crashes | Session .claude/pointer file | Resume from pointer |
| Verification fails | Verifier returns rejection | [escalate to user / re-implement / partial accept] |
| Scratchpad write conflict | File lock error | Retry with exponential backoff |

---

### Anti-Patterns Checklist

- [ ] ✅ Verifier has NO access to implementation plan (prevents confirmation bias)
- [ ] ✅ Workers do NOT share mutable state except via scratchpad (prevents races)
- [ ] ✅ Synthesis phase exists between research and implementation (prevents "telephone game")
- [ ] ✅ Context window per agent bounded by task scope (prevents overflow)
- [ ] ✅ Permission level appropriate per agent (researcher = read-only, implementer = full)
````

---

## Built-in Agent Types

Ready to use — no custom definition needed:

| Type | Key constraint | Best for |
|------|---------------|----------|
| `general-purpose` | All tools | Research, implementation, multi-step |
| `Explore` | Read-only tools only | Fast codebase exploration |
| `Plan` | Read-only tools only | Architecture planning |
| `verification` | Read-only tools only | Independent review |

Custom agents: create `.claude/agents/[name].md` with frontmatter:
```yaml
---
name: my-agent
description: what it does and when to use it
model: claude-opus-4-6
tools: [Bash, Read, Grep, Glob, FileWriteTool]
---
You are a specialized agent that...
```
