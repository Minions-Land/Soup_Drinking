🌐 [中文版](../zh-CN/工具设计模板.md) | [← README](../../README.en.md)  
🗺️ **Navigation**: [← Toolkit Index](../../README.en.md#toolkit) | [Tutorial EN](../../tutorial/en/) | [Reference](../../en/)

---

# Tool Design Template

> 🔧 [English](#english) | [中文](#中文)  
> 🗺️ **Navigation**: [← Toolkit Index](../../README.md#toolkit) | [Tutorial](../../tutorial/) | [Reference EN](../../en/) | [Reference ZH](../../zh-CN/)

A practical template for designing tools that agents can invoke. Every tool is a contract between the agent (caller) and the capability (callee). Getting the contract right is the single most important design decision in an agent system.

## Minimal Tool Structure

Every tool must implement five core elements:

```typescript
Tool {
  name: string              // Unique identifier, verb-noun pattern
  inputSchema: JSONSchema   // Strict schema for parameters
  call(input): Result       // Execute the action, return structured output
  prompt(): string          // System-prompt snippet: when/how to use
  render(result): string    // Human-readable formatting of the result
}
```

**name** -- Short, verb-noun pattern. `QueryDatabase`, `ReadFile`, `SendEmail`. The name appears in the model's tool list and must be self-explanatory.

**inputSchema** -- JSON Schema with `required` fields explicitly listed. Include `description` on every property. Constrain types tightly: use `enum` where values are finite, `pattern` for strings, `minimum`/`maximum` for numbers.

**call(input)** -- The execution body. Must be idempotent where possible. Returns a structured result object, never raw strings. Throws typed errors with actionable messages.

**prompt()** -- Injected into the system prompt. Tells the model *when* this tool is appropriate, *what* it does, and *what it does not do*. Keep under 200 words.

**render(result)** -- Converts the result into a string the model can reason about. Strip unnecessary detail. Truncate large outputs with a summary line.

## Step-by-Step Checklist

1. **Define the capability boundary.** What does this tool do? What does it explicitly NOT do? Write one sentence for each.
2. **Draft the input schema.** List every parameter. For each, decide: required or optional? What type? What constraints?
3. **Implement call().** Handle errors gracefully. Return structured data. Log for observability.
4. **Write prompt().** Describe the tool in terms the model understands. Include 1-2 usage examples inline.
5. **Implement render().** Format output for model consumption. Prefer tables or key-value pairs over prose.
6. **Add validation.** Validate inputs before execution. Return clear error messages on schema violations.
7. **Test with the model.** Run 10 diverse prompts. Check: Does the model select this tool appropriately? Does it fill parameters correctly? Does it misuse the tool?
8. **Iterate on prompt().** This is where most tool quality issues live. Refine until the model's selection accuracy exceeds 95%.

## Decision Matrix for Optional Methods

| Method | When to Implement | Skip When |
|---|---|---|
| `userConfirm(input)` | Tool has side effects (write, delete, send) | Read-only tools |
| `estimateCost(input)` | Tool calls paid APIs or long-running processes | Fast, free operations |
| `validatePermission(input)` | Tool accesses sensitive resources | Public/sandboxed resources |
| `cacheKey(input)` | Identical inputs always produce identical results | Time-sensitive or stateful results |
| `timeout()` | Tool may hang (network calls, shell commands) | In-memory operations |
| `rollback(result)` | Tool modifies state that may need reverting | Idempotent or read-only tools |
| `isReadOnly()` | Tool never writes to disk or network | Any tool with side effects |
| `isConcurrencySafe()` | Tool can safely run in parallel with itself | Tools that modify shared state |
| `isDestructive()` | Tool performs irreversible operations | Tools that can be undone |
| `progressCallback()` | Tool runs longer than 2 seconds | Sub-second tools |

## Example: DatabaseQueryTool

```typescript
const DatabaseQueryTool = {
  name: "DatabaseQuery",
  inputSchema: {
    type: "object",
    required: ["query"],
    properties: {
      query: {
        type: "string",
        description: "SQL SELECT query. Must be read-only."
      },
      database: {
        type: "string",
        enum: ["production_replica", "analytics", "staging"],
        default: "production_replica",
        description: "Target database. Defaults to production replica."
      },
      limit: {
        type: "integer",
        minimum: 1,
        maximum: 500,
        default: 100,
        description: "Maximum rows to return."
      }
    }
  },

  prompt() {
    return `Use DatabaseQuery to run read-only SQL against the database.
Always use production_replica unless the user specifies otherwise.
Never use this for INSERT, UPDATE, or DELETE operations.
Prefer specific column selection over SELECT *.`;
  },

  async call(input) {
    if (/\b(INSERT|UPDATE|DELETE|DROP|ALTER|TRUNCATE)\b/i.test(input.query)) {
      throw new ToolError("Only SELECT queries are permitted.");
    }
    const rows = await db.query(input.query, {
      database: input.database,
      limit: input.limit
    });
    return { rows, rowCount: rows.length, truncated: rows.length === input.limit };
  },

  render(result) {
    if (result.rowCount === 0) return "No rows returned.";
    const header = Object.keys(result.rows[0]).join(" | ");
    const lines = result.rows.map(r => Object.values(r).join(" | "));
    let output = `${header}\n${lines.join("\n")}`;
    if (result.truncated) output += `\n... (truncated at ${result.rowCount} rows)`;
    return output;
  }
};
```

## Key Principles

- **Fail loudly.** A tool that silently returns empty results is worse than one that throws an error. The model cannot debug silence.
- **Constrain inputs.** The tighter the schema, the fewer hallucinated parameters. Use enums, ranges, and patterns aggressively.
- **Prompt is king.** The model decides whether to use your tool based on prompt(), not on your implementation. Invest 80% of your iteration time here.
- **Render for reasoning.** The model will reason about render() output. Structure it for that purpose -- tables beat paragraphs, counts beat lists.
- **One tool, one job.** If your tool does two things, split it. `ReadFile` and `WriteFile`, not `FileOperation`.
- **Idempotency by default.** If the model retries a tool call (which happens), the second call should not corrupt state.
---
📁 [← Toolkit Index](../../README.en.md#toolkit) | 🌐 [中文版](../zh-CN/工具设计模板.md)
