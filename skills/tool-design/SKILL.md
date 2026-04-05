---
name: tool-design
description: "Design a new tool using Claude Code's buildTool pattern — produces a complete, implementation-ready tool spec"
---

<!--
# /tool-design — Tool Designer
# /tool-design — 工具设计器

PURPOSE / 用途:
  Given a description of what a tool should do, produces a complete tool
  specification using Claude Code's buildTool() pattern — ready to implement.

  根据工具的功能描述，使用 Claude Code 的 buildTool() 模式生成完整的工具规范——可立即实现。

USAGE / 用法:
  /tool-design <what the tool should do>
  /tool-design <工具的功能>

EXAMPLES / 示例:
  /tool-design A tool that queries PostgreSQL and returns formatted table results
  /tool-design 一个查询 PostgreSQL 并返回格式化表格结果的工具

  /tool-design A tool that sends Slack messages to a specified channel
  /tool-design 一个向指定频道发送 Slack 消息的工具

  /tool-design A tool that runs pytest and returns test results with failure details
  /tool-design 一个运行 pytest 并返回带失败详情的测试结果的工具

REFERENCE FILES READ / 读取的参考文件:
  - toolkit/tool-template.md     (buildTool pattern template)
  - en/01-TOOL-SYSTEM.md         (Tool interface deep dive)
  - tutorial/02-tool-system-deep-dive.md  (step-by-step explanation)
-->

# /tool-design — Tool Designer

Usage: `/tool-design <what the tool should do>`

---

## Your Role

Design a complete tool using Claude Code's `buildTool()` pattern. Produce a filled-in spec the user can implement immediately — no hand-waving, every field decided.

---

## Step 1 — Read the reference

Read `/Users/mjm/Claude Code/_reference/toolkit/tool-template.md`

Also skim `/Users/mjm/Claude Code/_reference/en/01-TOOL-SYSTEM.md` sections 1-4.

---

## Step 2 — Complete tool specification

Produce every section of the following spec. Do not leave anything as "TBD":

````markdown
## Tool Design: [ToolName]

### Metadata
| Field | Value | Reason |
|-------|-------|--------|
| `name` | `[ToolName]` | |
| `isReadOnly` | yes / no | [reads-only vs. modifies state] |
| `isDestructive` | yes / no | [irreversible? delete/overwrite/send?] |
| `isConcurrencySafe` | yes / no | [safe to run in parallel with itself?] |
| `maxResultSizeChars` | [N] | [typical output size × 2, or Infinity if self-bounded] |
| `shouldDefer` | yes / no | [used rarely = defer, used often = load always] |
| `searchHint` | `"[3-8 keywords]"` | [terms not in the tool name, for ToolSearch] |

---

### Input Schema (Zod)

```typescript
z.object({
  // Required parameters
  param_name: z.string().describe("Clear description for the model"),

  // Optional parameters
  optional_param: z.number().optional().describe("Description (default: N)"),
})
```

---

### Two-Stage Validation

**Stage 1 — validateInput** (logic, no user prompt):
```
[ ] Check: [e.g., "connection string matches postgresql:// format"]
[ ] Check: [e.g., "query does not contain semicolons (injection prevention)"]
[ ] Check: [e.g., "timeout is between 1 and 300 seconds"]
```

**Stage 2 — checkPermissions** (authorization):
```
Behavior: 'allow' / 'ask' / always-ask
Permission rule pattern: [ToolName]([key_param] pattern)
Example rule: DatabaseQuery(SELECT *)  → allow
              DatabaseQuery(DROP *)    → ask
Reason: [why this level of permission is appropriate]
```

---

### call() — Implementation Steps

```
1. [e.g., "Parse and validate connection string"]
2. [e.g., "Create connection pool with timeout config"]
3. [e.g., "Execute query with parameter binding (no string interpolation)"]
4. [e.g., "Format results as markdown table if < 50 rows, else CSV"]
5. [e.g., "Close connection, return { data: formattedResult }"]
```

**Error handling**:
- [Error type 1]: [how to surface it]
- [Error type 2]: [how to surface it]

---

### Progress Reporting (if long-running)

```typescript
// Stage 1
onProgress({ toolUseID, data: { type: 'tool-name', stage: 'connecting' } })

// Stage 2
onProgress({ toolUseID, data: { type: 'tool-name', stage: 'executing', rowsScanned: N } })
```

If fast operation (< 1s): "N/A — no progress needed"

---

### Rendering

**renderToolUseMessage**: `[ToolName] — [key_param_value]`
Example: `DatabaseQuery — SELECT * FROM users LIMIT 10`

**renderToolResultMessage**:
[How to display: table / count / summary / raw]
Example: "Show row count + first 5 rows as markdown table"

**isResultTruncated**: [yes if output can exceed display / no if always small]

---

### Security Checklist

- [ ] No string interpolation into queries/commands (use parameterized)
- [ ] Input sanitization: [specific sanitization needed]
- [ ] Path validation: [if file paths involved]
- [ ] Credential handling: [from env vars, not hardcoded]
- [ ] Sandbox needed: [yes/no + reason]
- [ ] Rate limiting consideration: [if calling external API]

---

### prompt() — Model Instructions

Key things to tell the model about this tool:
- When to use it: [specific conditions]
- When NOT to use it: [anti-patterns]
- Example invocation: [show correct parameters]
````

---

## Key Design Decisions to Justify

**`isReadOnly`**: Matters for plan mode — read-only tools are allowed in plan mode, write tools are blocked.

**`isConcurrencySafe`**: If false, this tool serializes with other non-concurrent tools. Set true only if the tool has no shared state side effects.

**`maxResultSizeChars`**: If the result exceeds this, it auto-persists to disk and Claude gets a preview. Set to `Infinity` for tools that self-bound (like `FileReadTool`). Rule of thumb: `typical_output_size × 3`.

**`shouldDefer`**: Set true if this tool is used in fewer than ~20% of conversations. Saves prompt tokens. The model uses `ToolSearchTool` to discover deferred tools when needed.

**`searchHint`**: Choose terms the model would naturally use when looking for this capability, that aren't in the tool name. For a `DatabaseQueryTool`, good hints: `"sql postgres mysql database query rows table"`.
