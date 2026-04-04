🌐 [中文版](../../tutorial/zh-CN/05-权限与安全.md) | [← README](../../README.en.md)  
🗺️ **Navigation**: [← Ch.4](04-task-lifecycle.md) | **Chapter 5** | [Ch.6 →](06-terminal-ui.md)  
📁 [Tutorial EN](../../tutorial/en/) | [Toolkit EN](../../toolkit/en/) | [Reference](../../en/)

---

# Chapter 5: Permission & Security

> 📖 [English](#english) | [中文](#中文)  
> 🗺️ **Navigation**: [← Ch.4](04-task-lifecycle.md) | **Chapter 5** | [Ch.6 →](06-terminal-ui.md)  
> 📁 [Tutorial Index](../README.md#tutorial) | [Toolkit](../toolkit/) | [Reference EN](../en/) | [Reference ZH](../zh-CN/)

<a name="english"></a>

## Why Permissions Matter

Claude Code is an autonomous agent that can read files, write code, execute shell commands, and browse the web. Without a permission system, a single misunderstood instruction could delete your codebase, expose secrets, or run destructive commands. The permission system is what makes autonomous operation trustworthy.

## The Permission Model

Claude Code uses a **layered permission model**. Every tool call passes through multiple checkpoints before execution:

```
Tool Call Request
    |
    v
[1. Schema Validation]     -- Is the input well-formed?
    |
    v
[2. validateInput()]       -- Is the input semantically valid?
    |
    v
[3. checkPermissions()]    -- Is the user OK with this action?
    |
    v
[4. Hook: PreToolUse]      -- Do custom hooks allow it?
    |
    v
[5. Sandbox Enforcement]   -- Is the action within sandbox bounds?
    |
    v
[6. Tool Execution]        -- Finally, run the tool
    |
    v
[7. Hook: PostToolUse]     -- Post-execution inspection
```

Each layer can independently block the call. A tool call must pass all layers to execute. This defense-in-depth approach means that a bug in one layer does not compromise security.

### Permission Modes

Claude Code supports several permission modes defined in `src/types/permissions.ts`:

| Mode | Behavior |
|------|----------|
| `default` | Ask user for non-read-only operations |
| `plan` | Read-only tools only, no mutations allowed |
| `auto` | Model-guided approval with safety classifier |
| `bypassPermissions` | Skip all prompts (dangerous, explicit opt-in) |

## checkPermissions()

The `checkPermissions()` method is the primary permission gate. Each tool implements its own version. The method receives the tool input and context, and returns one of three outcomes:

- **allow.** The action is permitted. Proceed to execution.
- **deny.** The action is blocked. Return an error to the model.
- **ask.** The action requires explicit user approval. Show a prompt.

The decision logic considers several factors: Is this a read-only operation? Has the user already granted permission for this path? Is this path in the allowed list? Is it in the denied list? Does a permission rule cover this case?

## Permission Rules

Users can configure permission rules in `.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "Read:**",
      "Bash:npm test",
      "Bash:npm run build",
      "Write:src/**"
    ],
    "deny": [
      "Bash:rm -rf *",
      "Write:package-lock.json"
    ]
  }
}
```

Rules are matched in order: deny rules take priority over allow rules. A rule like `"Bash:npm test"` means "allow the Bash tool to run the command `npm test` without asking."

## BashTool Security: bashSecurity.ts

The Bash tool is the most dangerous tool in the system. It can execute arbitrary shell commands. The `src/tools/BashTool/` directory contains a dedicated security sub-system:

**bashSecurity.ts** -- Regex-based command filtering for dangerous patterns. The module parses the command string to understand what it does, identifying the command name, arguments, pipes, redirects, and subshells.

**commandSemantics.ts** -- Classifies commands by their nature: read, search, list, or write operations.

**destructiveCommandWarning.ts** -- Detects and warns about destructive commands like `rm -rf`, `git reset --hard`, and similar irreversible operations.

**pathValidation.ts** -- Validates that file paths in commands are within allowed directories.

**sedValidation.ts** -- Checks sed commands for safety.

**shouldUseSandbox.ts** -- Determines if sandbox execution is needed for a given command.

**Risk classification:**
- **Safe.** Read-only commands like `ls`, `cat`, `grep`, `git status`. Auto-allowed.
- **Moderate.** Commands that modify files in limited ways, like `mkdir` or `cp`. May require confirmation.
- **Dangerous.** Commands that could cause data loss, like `rm -rf`, `chmod 777`, or `curl | sh`. Always require confirmation.

## Sandbox

Claude Code can operate within a **sandbox** that restricts filesystem and network access:

- **Filesystem sandbox.** Limits which directories can be read from or written to. Paths outside the sandbox are blocked.
- **Network sandbox.** On macOS, the `sandbox-exec` mechanism can restrict network access for bash commands, preventing data exfiltration.

The sandbox is enforced at the OS level, not just in application code. This means even if a tool has a bug, the OS prevents it from accessing unauthorized resources.

## Permission Bubbling

When a sub-agent (Chapter 3) needs permission, the request **bubbles up** to the user:

1. Sub-agent's tool calls `checkPermissions()`.
2. If the result is `ask`, the request is forwarded to the parent agent's context.
3. The parent agent surfaces the permission prompt to the user.
4. The user's decision is propagated back to the sub-agent.

This bubbling ensures that permission decisions are always made by the user, regardless of how deeply nested the agent is. Granted permissions are inherited downward -- if the user grants write access to `src/`, all sub-agents receive that grant automatically.

## Denial Tracking

When the user denies a permission, the system tracks it:

1. **Avoid re-asking.** If the user denied `rm -rf node_modules`, the model should not ask again in the same session. The denial is cached.
2. **Inform the model.** The denial is included in the tool result as an error: "Permission denied by user." This tells the model to find an alternative approach.

Denial tracking prevents the frustrating pattern of an agent repeatedly asking for the same permission after being told "no."

## Hooks: PreToolUse and PostToolUse

Hooks are custom functions that run before or after tool execution. They are configured in `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "node .claude/hooks/validate-bash.js"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write",
        "command": "node .claude/hooks/lint-on-write.js"
      }
    ]
  }
}
```

**PreToolUse hooks** run after permission checks but before execution. They can:
- Block the tool call (exit code 2)
- Allow it (exit code 0)
- Modify the input (by writing to stdout)

**PostToolUse hooks** run after execution. They can:
- Log the action for auditing
- Run linters or formatters on written files
- Send notifications

Hooks receive the tool name, input, and (for PostToolUse) result as JSON on stdin. They are external processes, so they can be written in any language.

## Security Design Principles

The permission system embodies principles worth adopting in any agent:

1. **Default deny.** If no rule matches, ask the user. Never auto-allow an unknown action.
2. **Defense in depth.** Multiple independent layers each provide protection.
3. **Minimal privilege.** Read-only tools are auto-allowed; write tools require permission.
4. **User sovereignty.** The user always has the final say.
5. **Transparency.** Every permission decision is logged and visible.
6. **Fail closed.** A new tool defaults to more restrictions, not fewer.
---
📁 [← Back to Tutorial Index](../../README.en.md#tutorial) | 🌐 [中文版](../../tutorial/zh-CN/05-权限与安全.md) | [Ch.6 →](06-terminal-ui.md)
