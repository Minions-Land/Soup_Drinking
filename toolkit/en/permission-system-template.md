🌐 [中文版](../zh-CN/权限系统模板.md) | [← README](../../README.en.md)  
🗺️ **Navigation**: [← Toolkit Index](../../README.en.md#toolkit) | [Tutorial EN](../../tutorial/en/) | [Reference](../../en/)

---

# Permission System Template

> 🔧 [English](#english) | [中文](#中文)  
> 🗺️ **Navigation**: [← Toolkit Index](../../README.md#toolkit) | [Tutorial](../../tutorial/) | [Reference EN](../../en/) | [Reference ZH](../../zh-CN/)

A practical template for designing permission systems in agent architectures. Permissions are the safety boundary between what an agent can do and what it is allowed to do. A well-designed permission system prevents destructive actions without blocking productive work.

## Two-Stage Validation

Every tool invocation passes through two stages before execution:

```
Stage 1: Input Validation        Stage 2: Permission Check
+-------------------------+      +-------------------------+
| Is the input valid?     |      | Is this action allowed? |
| - Schema conformance    | ---> | - Permission rules      |
| - Business constraints  |      | - User approval         |
| - Resource existence    |      | - Sandbox restrictions  |
+-------------------------+      +-------------------------+
         |                                |
     REJECT if invalid             REJECT / ASK / ALLOW
```

### Stage 1: Input Validation (`validateInput`)

Runs before permissions. Catches malformed inputs early. Returns a clear error message without prompting the user.

```typescript
async validateInput(input, context): ValidationResult {
  if (!input.filePath) return { result: false, message: "filePath is required" };
  if (!await fileExists(input.filePath)) {
    return { result: false, message: `File not found: ${input.filePath}` };
  }
  return { result: true };
}
```

### Stage 2: Permission Check (`checkPermissions`)

Runs after validation succeeds. Returns one of three behaviors:

```typescript
type PermissionResult =
  | { behavior: 'allow'; updatedInput: Input }    // Proceed without asking
  | { behavior: 'ask'; message: string }           // Show prompt to user
  | { behavior: 'deny'; message: string }          // Block silently
```

**allow** -- The action is pre-approved. Execution proceeds immediately. Use for read-only operations, operations matching user-configured rules, or sandboxed environments.

**ask** -- The user must explicitly approve. Use for write operations, destructive actions, or operations outside known-safe patterns.

**deny** -- The action is blocked without prompting. Use for operations that violate hard security boundaries (e.g., writing outside the project directory).

## Permission Modes

Configure the overall permission stance for a session or agent:

| Mode | Behavior | Use Case |
|------|----------|----------|
| `default` | Ask for writes, allow reads | Interactive development |
| `plan` | Allow reads + writes, ask for shell commands | Semi-autonomous work |
| `acceptEdits` | Allow all file edits, ask for shell | Trusted edit sessions |
| `bypassPermissions` | Allow everything | CI/CD, fully trusted automation |
| `auto` | Allow everything (non-interactive) | SDK/headless mode |

### Mode Inheritance in Multi-Agent Systems

```
Parent Mode        Agent permissionMode    Effective Mode
-----------        --------------------    --------------
bypassPermissions  (any)                   bypassPermissions  (parent wins)
acceptEdits        (any)                   acceptEdits        (parent wins)
auto               (any)                   auto               (parent wins)
default            plan                    plan               (agent wins)
default            bubble                  bubble             (agent wins)
default            (none)                  default            (inherited)
```

Rule: Permissive parent modes always override. The agent's mode only applies when the parent is in `default` mode.

## Permission Rules

Rules are pattern-based entries that pre-answer permission questions:

```typescript
type PermissionRule = {
  tool: string              // Tool name or glob pattern: "Bash", "Edit*"
  allow?: boolean           // true = allow, false = deny
  prefix?: string           // Command prefix match: "npm test"
  path?: string             // File path glob: "src/**/*.ts"
  scope: 'session' | 'project' | 'global'
}
```

### Rule Evaluation Order

```
1. Check session rules (most specific, highest priority)
2. Check project rules (.claude/settings.json)
3. Check global rules (~/.claude/settings.json)
4. Check permission mode defaults
5. Fall through to 'ask'
```

### Example Rules

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(npm test*)",
      "Bash(git diff*)",
      "Edit(src/**/*.ts)"
    ],
    "deny": [
      "Bash(rm -rf*)",
      "Bash(git push --force*)",
      "Edit(.env*)"
    ]
  }
}
```

## Destructive Action Detection

Certain actions require elevated scrutiny. Detect and flag them automatically:

```typescript
function isDestructive(tool: string, input: any): boolean {
  if (tool === 'Edit' && input.operation === 'delete') return true;
  if (tool === 'Bash') {
    const cmd = input.command;
    if (/\brm\s+-rf?\b/.test(cmd)) return true;
    if (/\bgit\s+(push\s+--force|reset\s+--hard|clean\s+-f)/.test(cmd)) return true;
    if (/\bdrop\s+table\b/i.test(cmd)) return true;
    if (/\btruncate\b/i.test(cmd)) return true;
  }
  return false;
}
```

**Destructive action handling:**

1. Always require explicit user approval, regardless of permission mode.
2. Show a clear warning describing the irreversible nature of the action.
3. In background agents (no terminal), auto-deny destructive actions.
4. Log all destructive action attempts for audit trailing.

## Sandbox Integration

Sandboxes provide a hard boundary that permission rules cannot override:

```typescript
type SandboxConfig = {
  allowedPaths: string[]       // Paths the agent can read/write
  deniedPaths: string[]        // Paths always blocked
  networkAccess: boolean       // Can the agent make network calls
  maxFileSize: number          // Maximum file write size
  allowedCommands: string[]    // Shell command allowlist
  environment: 'macos' | 'docker' | 'vm'
}
```

### Sandbox Evaluation (Before Permission Rules)

```
Sandbox check (hard boundary)
  |
  +-- DENIED by sandbox --> REJECT (no override possible)
  |
  +-- ALLOWED by sandbox --> proceed to permission rules
      |
      +-- Permission rule match --> ALLOW or DENY
      |
      +-- No rule match --> ASK user
```

On macOS, the sandbox uses the native App Sandbox (`sandbox-exec`) to restrict file system access, network, and process spawning at the OS level. In Docker environments, container boundaries serve the same purpose. The sandbox is the layer of last resort -- nothing bypasses it.

## Permission Bubbling in Multi-Agent Systems

When a child agent needs permission but has no terminal access:

```
Child Agent (background)
  |
  needs permission for: Edit("src/auth.ts")
  |
  +-- permissionMode: 'bubble'
  |     |
  |     v
  |   Parent Agent's terminal
  |     "Child agent 'auth-fixer' wants to edit src/auth.ts"
  |     [Allow] [Deny] [Allow All for this agent]
  |
  +-- permissionMode: 'default' (no bubble)
        |
        v
      Auto-DENY (no terminal access)
```

### Implementing Permission Bubbling

```typescript
async function checkPermissionWithBubble(tool, input, agentContext) {
  const localResult = await tool.checkPermissions(input, agentContext);

  if (localResult.behavior === 'ask' && agentContext.permissionMode === 'bubble') {
    return await agentContext.parentPermissionHandler({
      agentId: agentContext.agentId,
      agentType: agentContext.agentType,
      tool: tool.name,
      input: input,
      message: localResult.message,
    });
  }

  if (localResult.behavior === 'ask' && agentContext.isBackground) {
    return { behavior: 'deny', message: 'Auto-denied: no terminal access' };
  }

  return localResult;
}
```

## Key Principles

- **Fail closed.** If the permission system encounters an unexpected state, deny the action. Never default to allow.
- **Sandbox is absolute.** No permission rule, no user approval, no mode override can bypass the sandbox.
- **Audit everything.** Every permission decision (allow, deny, ask+response) should be logged with timestamp, tool, input, and decision.
- **Session rules are ephemeral.** They die with the session. Project and global rules persist. This prevents accidental permanent grants.

## Implementation Checklist

1. Define your permission modes and their default behaviors.
2. Implement two-stage validation: validateInput then checkPermissions.
3. Build a rule engine with session/project/global scopes and evaluation order.
4. Add destructive action detection for common dangerous patterns.
5. Integrate sandbox boundaries as a hard layer before permission rules.
6. Implement permission bubbling for multi-agent scenarios.
7. Add audit logging for all permission decisions.
8. Test with adversarial prompts that attempt to bypass permissions.
---
📁 [← Toolkit Index](../../README.en.md#toolkit) | 🌐 [中文版](../zh-CN/权限系统模板.md)
