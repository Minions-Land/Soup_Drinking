---
name: find-pattern
description: "Find the right Claude Code design pattern for a specific agent engineering problem"
---

<!--
# /find-pattern — Pattern Finder
# /find-pattern — 设计模式匹配器

PURPOSE / 用途:
  Maps a specific agent engineering problem to the most relevant pattern
  from Claude Code's 12 battle-tested design patterns.

  将具体的智能体工程问题映射到 Claude Code 12 个经过生产验证的设计模式中最相关的一个。

USAGE / 用法:
  /find-pattern <problem description>
  /find-pattern <问题描述>

EXAMPLES / 示例:
  /find-pattern How do I let multiple agents share findings without passing huge text through prompts?
  /find-pattern 如何让多个智能体共享发现而不通过提示词传递大量文本？

  /find-pattern My child agent needs parent's full research context but re-tokenizing is expensive
  /find-pattern 子智能体需要父级的完整研究上下文，但重新分词成本很高

  /find-pattern I have 30 tools and the system prompt is getting too long
  /find-pattern 我有30个工具，系统提示词变得太长了

REFERENCE FILES READ / 读取的参考文件:
  - en/09-PATTERNS.md       (all 12 patterns)
  - relevant toolkit file   (implementation details)
-->

# /find-pattern — Pattern Finder

Usage: `/find-pattern <problem description>`

---

## Your Role

Quickly match the user's problem to the right Claude Code design pattern. Be fast and precise. They just need: WHAT pattern to use + WHERE to find the implementation.

---

## Step 1 — Read the patterns catalog

Read `/Users/mjm/Claude Code/_reference/en/09-PATTERNS.md`

---

## Step 2 — Output format

```markdown
## Pattern Match: Pattern #N — [Pattern Name]

**The problem it solves**
[one clear sentence]

**How Claude Code implements it**
[two sentences, concrete — name the actual source files]

**The non-obvious insight**
[the key thing that makes this pattern work, that isn't obvious at first]

**Read this next**
`/Users/mjm/Claude Code/_reference/[relevant toolkit or reference file]`

**First thing to implement**
[the single most important concrete step]
```

If multiple patterns apply, lead with the PRIMARY one, then add a brief "Also consider: Pattern #N" section.

---

## Pattern Reference Map

| Pattern | Core idea | Toolkit | Reference |
|---------|-----------|---------|-----------|
| #1 Tool Interface + Defaults | `buildTool()` fills safe defaults | `toolkit/tool-template.md` | `en/01-TOOL-SYSTEM.md` |
| #2 Context Forking | byte-exact prompt cache inheritance | `toolkit/agent-spawning-template.md` | `en/02-MULTI-AGENT.md` |
| #3 Fan-Out → Synthesize → Impl → Verify | 4-phase coordinator, fresh verifier | `toolkit/coordinator-workflow-template.md` | `en/02-MULTI-AGENT.md` |
| #4 Two-Stage Validation | `validateInput` (logic) + `checkPermissions` (auth) | `toolkit/permission-system-template.md` | `tutorial/05-permission-security.md` |
| #5 AsyncLocalStorage Isolation | per-agent state without manual threading | `toolkit/task-system-template.md` | `en/03-TASK-SYSTEM.md` |
| #6 Progressive Result Delivery | typed `ToolCallProgress<P>` callbacks | `toolkit/tool-template.md` | `en/01-TOOL-SYSTEM.md` |
| #7 Deferred Tool Loading | `shouldDefer` + ToolSearch discovery | `toolkit/tool-template.md` | `en/01-TOOL-SYSTEM.md` |
| #8 XML Async Notifications | `<task-notification>` XML in message stream | `toolkit/agent-spawning-template.md` | `en/02-MULTI-AGENT.md` |
| #9 Shared Work Queue | agents self-claim from TaskList | `toolkit/swarm-team-template.md` | `en/02-MULTI-AGENT.md` |
| #10 Multi-Source Command Registry | aggregated built-in + skills + plugins | `en/05-COMMAND-SYSTEM.md` | `tutorial/07-command-plugin-system.md` |
| #11 GC-Free Packed Buffers | `Int32Array` cells, interning pools | `en/06-INK-UI.md` | `tutorial/06-terminal-ui.md` |
| #12 Coordinator + Scratchpad | shared filesystem for cross-agent findings | `toolkit/coordinator-workflow-template.md` | `en/02-MULTI-AGENT.md` |

All files are under: `/Users/mjm/Claude Code/_reference/`

---

## Common Problem → Pattern Shortcuts

| Problem keywords | Most likely pattern |
|-----------------|---------------------|
| "share context", "inherit", "child agent knows parent's work" | #2 Context Forking |
| "parallel research", "multiple agents searching" | #3 Fan-Out + Coordinator |
| "share findings", "cross-agent knowledge", "scratchpad" | #12 Coordinator + Scratchpad |
| "same process", "multiple agents", "isolation" | #5 AsyncLocalStorage |
| "background", "async", "notification when done" | #8 XML Notifications |
| "work queue", "distribute tasks", "parallel workers" | #9 Shared Work Queue |
| "too many tools", "prompt too long", "token budget" | #7 Deferred Loading |
| "long running", "streaming output", "progress" | #6 Progressive Delivery |
| "dangerous command", "safe default", "permission" | #4 Two-Stage Validation |
| "new tool", "buildTool", "tool interface" | #1 Tool Interface |
| "extensible commands", "slash commands", "plugins" | #10 Command Registry |
| "terminal UI", "rendering performance", "GC" | #11 GC-Free Buffers |
