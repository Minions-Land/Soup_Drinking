🌐 [中文版](../../tutorial/zh-CN/07-命令与插件.md) | [← README](../../README.en.md)  
🗺️ **Navigation**: [← Ch.6](06-terminal-ui.md) | **Chapter 7** | [Ch.8 →](08-remote-bridge.md)  
📁 [Tutorial EN](../../tutorial/en/) | [Toolkit EN](../../toolkit/en/) | [Reference](../../en/)

---

# Chapter 7: Commands & Plugins -- Extending the Agent

> Previous: [Chapter 6: Terminal UI](06-terminal-ui.md) | Next: [Chapter 8: Remote Connection](08-remote-bridge.md)

Imagine you have a personal assistant who can answer questions and run tools. That is already powerful. But what if you could teach the assistant new tricks -- like a `/commit` command that knows your Git workflow, or a `/review-pr` command that follows your team's code review checklist? And what if anyone could package those tricks as a plugin for the whole community?

This chapter explains how Claude Code's **command and plugin system** makes the agent endlessly extensible without changing core code.

## What Is a Slash Command?

When you type `/commit` in Claude Code, you are invoking a **slash command**. A slash command is a shortcut that triggers a specific behavior. Some commands run local JavaScript logic. Others expand into a prompt that the AI model processes. Still others render an interactive React-based UI component right in your terminal.

The system that manages all of this is the **Command Registry**, defined in `src/commands.ts`.

## The Command Registry: One Place, Many Sources

The registry's most interesting design choice is that commands do not all come from the same place. Instead, it **aggregates** commands from four different sources at runtime:

| Source | Description | Example |
|--------|-------------|---------|
| Built-in | Hardcoded into the CLI source code | `/exit`, `/help`, `/clear` |
| Skills | Prompt files on disk that the AI model can invoke | `/commit`, `/review-pr` |
| Plugins | Dynamically loaded extensions | Third-party tools from `src/plugins/` |
| Workflows | Scriptable tool chains | Custom user-defined automation |

When you press Tab to see available commands, or when the agent considers which skill to invoke, all four sources are merged into one unified list. This is the **multi-source aggregation pattern** -- a single registry that pulls from many providers and presents a uniform interface.

Think of it like a restaurant menu. The kitchen (built-in) provides the standard dishes. A guest chef (plugin) adds specials. A seasonal recipe card (skill) adds limited-time items. The waiter (registry) hands the diner one combined menu.

## Three Command Types

Every command has a `type` field that determines how it executes:

```typescript
type: 'prompt' | 'local' | 'local-jsx'
```

**`prompt`** commands are the most interesting. When you type `/commit`, no JavaScript function directly creates a Git commit. Instead, the command constructs a carefully crafted prompt and injects it into the conversation. The AI model reads this prompt and then uses its tools (like `BashTool` to run `git commit`) to produce the result. The command defines the *intent*; the model figures out the *execution*.

**`local`** commands bypass the LLM entirely. `/clear` does not need AI intelligence -- it simply resets the conversation state in memory and returns confirmation text. `/exit` quits the program. These are deterministic operations that happen instantly.

**`local-jsx`** commands render interactive React/Ink UI components in the terminal. `/config` opens a settings editor where you navigate with arrow keys, toggle options, and save -- all without the AI model being involved. The command returns a React component rather than text.

This three-type design elegantly separates concerns: AI-driven behavior (`prompt`), programmatic logic (`local`), and interactive UI (`local-jsx`). When adding a new command, you pick the type that matches your needs.

## How Commands Are Discovered and Filtered

When the registry's `getCommands()` function is called, it performs multi-source aggregation:

```
getCommands() =
    Built-in (hardcoded array in src/commands/)
  + Skills (scanned from prompt files on disk)
  + Plugins (dynamically registered at startup)
  + Workflows (user-defined scriptable chains)
  --> Deduplicate by name
  --> Filter by auth level, mode, safety
  --> Return unified list
```

The filtering step is critical. Commands carry metadata about who can use them and when:

- **Auth level**: Some commands are only available to `claude-ai` users, others to `console` users. The same CLI binary serves both audiences, but the command list adapts.
- **Mode**: When Claude Code runs in a remote-controlled session (from claude.ai/code on your phone), only commands in the `REMOTE_SAFE_COMMANDS` set appear. You would not want a mobile user accidentally triggering `/clear` on a long conversation.
- **Bridge safety**: Commands used via mobile or web remote control are curated to prevent actions that could disrupt the local environment.

This layered filtering means the command list automatically adapts to the user's environment without any manual configuration.

## The Skill System: Where AI Meets Commands

Skills are the most novel part of the command system. A skill is a **prompt template** packaged as a command that both users and the AI model can invoke.

When the model decides it needs to create a Git commit, it calls the `SkillTool` (defined in `src/tools/SkillTool/`). The tool looks up the skill, loads its prompt file from disk, and injects the prompt into the conversation. From the model's perspective, it is as if the user typed a detailed request.

Here is the flow when someone types `/commit`:

1. The registry identifies `/commit` as a skill (type: `prompt`)
2. The skill's prompt template is loaded from disk
3. The prompt text is injected into the conversation
4. The model reads the instructions and acts -- calling `Bash` to run git commands, `Read` to check files, etc.
5. The model produces the commit

Skills are powerful because they are just text files. Anyone can write a new skill by creating a prompt file that describes what the model should do. No compiled code is required. The model's general intelligence handles the execution.

For example, the `/commit` skill instructs the model to: check `git status`, review diffs with `git diff`, examine recent commit messages for style conventions, draft a message, and create the commit. The `/compact` skill tells the model to summarize the conversation to reclaim context space. These are complex workflows described entirely in natural language.

## Plugin Architecture: Skills + Hooks + MCP Servers

Plugins are the broadest extension mechanism. A single plugin can provide any combination of three capabilities:

**Skills** -- Additional prompt-based commands, just like the built-in ones. A plugin might add `/deploy`, `/lint`, or `/migrate`.

**Hooks** -- Interceptors that run at specific lifecycle points. For example, a `PermissionRequest` hook can customize how permission prompts are handled, adding custom approval logic or logging.

**MCP Servers** -- External tool providers using the Model Context Protocol. An MCP server can expose entirely new tools to the agent. A plugin might provide a database query tool, a Jira integration, or a Figma reader. The model sees these as additional tools in its toolbox, indistinguishable from built-in ones.

### Built-in vs. Marketplace Plugins

Claude Code ships with **bundled plugins** in `plugins/bundled/`. These are pre-installed and can be toggled on or off via the `/plugin` command. They follow the same registration interface as any other plugin -- there is no privileged API for first-party code.

**Marketplace plugins** are loaded from external sources. Third-party developers publish them, and users install them. Because they use the same registration interface as built-in plugins, they work identically.

This two-tier model is a proven pattern in extensible software: ship useful defaults, but let the community extend infinitely. VS Code, Webpack, and Figma all follow the same approach.

## Four Commands in Practice

Let us trace four real commands to see the types in action:

**`/config`** (type: `local-jsx`): Opens an interactive settings editor rendered as a React/Ink component. The user navigates with arrow keys, toggles settings, and saves. No AI model is involved.

**`/commit`** (type: `prompt`/skill): Generates a detailed prompt that instructs the model to examine staged changes with `git diff`, read recent commit messages for style consistency, draft a commit message, and execute `git commit`. The model does all the thinking; the command just provides the recipe.

**`/compact`** (type: `prompt`): Tells the model to summarize the current conversation into a shorter form, preserving essential context while reducing token usage. Pure prompt engineering -- no custom code needed.

**`/branch`** (type: `prompt`/skill): Expands into a prompt that instructs the model to create or switch Git branches, handling edge cases like uncommitted changes.

The pattern is clear: most commands are just prompts. The AI's general capability does the heavy lifting. Only commands that need direct programmatic control use `local` or `local-jsx`.

## Designing Your Own Extensible Command System

If you are building an agent or chatbot, here is how to apply these patterns:

1. **Define a uniform command interface.** Keep it minimal -- name, type, and handler. Everything else should have safe defaults.
2. **Build a central registry** that aggregates commands from multiple sources. The registry should *discover* commands, not import them directly.
3. **Support multiple command types.** At minimum, support "prompt" (delegate to LLM) and "local" (execute directly).
4. **Add contextual filtering.** Not every command should be available in every context. Filter by user role, session type, or environment.
5. **Make it pluggable.** Define a plugin interface that lets external code register new commands, hooks, and tool providers. Use the same interface for first-party and third-party extensions.

## Key Takeaway: Multi-Source Aggregation Pattern

The central lesson from Claude Code's command system is the **multi-source aggregation pattern**:

> Instead of hardcoding all functionality, define a standard command interface and aggregate implementations from multiple independent sources (built-in code, skill files, plugins, workflows). Apply context-aware filtering at query time. This creates a system that is unified in its API and infinitely extensible in its capabilities.

This pattern appears everywhere in well-designed extensible systems. The power lies in the contract: as long as each provider implements the same interface, the registry does not care where the command came from.
---
📁 [← Tutorial Index](../../README.en.md#tutorial) | 🌐 [中文版](../../tutorial/zh-CN/07-命令与插件.md) | [Ch.8 →](08-remote-bridge.md)
