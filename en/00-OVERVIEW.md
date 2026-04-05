# Claude Code Source Code - Complete Architecture Reference

> 🗺️ **Navigation**: [← README](../README.md) | [zh-CN version](../zh-CN/00-总览.md)  
> 📁 [Reference Index](../README.md#reference) | [Tutorial](../tutorial/) | [Toolkit](../toolkit/)

> This reference is auto-generated from 1332 TypeScript source files across the Claude Code CLI codebase.
> Purpose: Serve as a reusable pattern library for designing agent workflows and tools.

## Repository Structure

```
src/
├── Tool.ts                  # Base Tool interface & buildTool helper
├── Task.ts                  # Base Task abstraction
├── QueryEngine.ts           # Main conversation/query engine
├── commands.ts              # Slash command registry
├── history.ts               # Conversation history management
│
├── tools/                   # 30+ tool implementations
│   ├── AgentTool/           # Multi-agent system (core)
│   ├── BashTool/            # Shell execution with security
│   ├── FileEditTool/        # File editing (search & replace)
│   ├── FileReadTool/        # File reading with limits
│   ├── FileWriteTool/       # File creation/overwrite
│   ├── GlobTool/            # File pattern matching
│   ├── GrepTool/            # Content search (ripgrep)
│   ├── WebFetchTool/        # URL fetching & HTML->MD
│   ├── WebSearchTool/       # Web search
│   ├── MCPTool/             # Model Context Protocol
│   ├── SendMessageTool/     # Inter-agent messaging
│   ├── TaskCreateTool/      # Background task creation
│   ├── TodoWriteTool/       # Progress tracking
│   ├── SkillTool/           # Slash command dispatcher
│   ├── EnterPlanModeTool/   # Plan mode entry
│   ├── ScheduleCronTool/    # Cron scheduling
│   ├── TeamCreateTool/      # Swarm team management
│   └── shared/              # Shared utilities
│
├── tasks/                   # Task lifecycle management
│   ├── DreamTask/           # Background "dream" tasks
│   ├── LocalShellTask/      # Shell process tasks
│   ├── LocalAgentTask/      # Local sub-agent tasks
│   ├── RemoteAgentTask/     # Remote agent tasks
│   └── InProcessTeammateTask/ # In-process swarm members
│
├── coordinator/             # Orchestrator/leader mode
├── bridge/                  # Remote connection (claude.ai/code)
├── server/                  # Direct connect / SDK server
├── ink/                     # Terminal UI framework (React-for-CLI)
├── vim/                     # Vim mode emulation
├── cli/                     # CLI infrastructure & transports
├── commands/                # Slash command implementations
├── plugins/                 # Plugin architecture
├── utils/                   # Utility modules
│   ├── settings/            # Settings management
│   ├── permissions/         # Permission system
│   ├── memory/              # Memory system
│   ├── mcp/                 # MCP utilities
│   ├── shell/               # Shell execution
│   ├── hooks/               # Hook system
│   ├── swarm/               # Multi-agent coordination
│   ├── sandbox/             # Sandboxing
│   └── ...                  # 20+ more utility modules
├── memdir/                  # Shared filesystem for agents
├── bootstrap/               # App initialization
├── constants/               # Constants
├── types/                   # Type definitions
├── keybindings/             # Key binding system
├── buddy/                   # Companion/buddy feature
└── migrations/              # Settings migration scripts
```

## Module Index

| # | Reference File | Module | Reusable Patterns |
|---|----------------|--------|-------------------|
| 1 | [01-TOOL-SYSTEM.md](01-TOOL-SYSTEM.md) | Tool Architecture | Tool interface, buildTool, permission system, validation |
| 2 | [02-MULTI-AGENT.md](02-MULTI-AGENT.md) | Multi-Agent System | Agent spawning, forking, communication, swarm |
| 3 | [03-TASK-SYSTEM.md](03-TASK-SYSTEM.md) | Task Management | Task lifecycle, background tasks, async coordination |
| 4 | [04-TOOLS-CATALOG.md](04-TOOLS-CATALOG.md) | Tools Catalog | All 30+ tools with parameters and patterns |
| 5 | [05-COMMAND-SYSTEM.md](05-COMMAND-SYSTEM.md) | Commands & CLI | Slash commands, CLI architecture, vim mode |
| 6 | [06-INK-UI.md](06-INK-UI.md) | Terminal UI Framework | Rendering pipeline, layout, events, hooks |
| 7 | [07-BRIDGE-SERVER.md](07-BRIDGE-SERVER.md) | Remote Connection | Bridge, sessions, transports, auth |
| 8 | [08-UTILS-CORE.md](08-UTILS-CORE.md) | Utilities & Core | Settings, permissions, MCP, plugins, sandbox |
| 9 | [09-PATTERNS.md](09-PATTERNS.md) | Reusable Design Patterns | Key patterns to copy for your own agents |
