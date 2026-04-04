# Tools Catalog

> 🗺️ **Navigation**: [← README](../README.md) | [zh-CN version](../zh-CN/04-工具目录.md)  
> 📁 [Reference Index](../README.md#reference) | [Tutorial](../tutorial/) | [Toolkit](../toolkit/)

> Complete catalog of all 30+ tools in Claude Code

## File System Tools

### FileReadTool
- **Purpose**: Read file contents with line numbers
- **Key Features**: Binary/PDF detection, token limits, image support (multimodal)
- **Parameters**: `file_path`, `offset?`, `limit?`, `pages?` (PDF)
- **Pattern**: Auto-truncates large files, returns `cat -n` format

### FileEditTool
- **Purpose**: Search-and-replace text editing
- **Key Features**: Exact string matching, `replace_all` flag, LSP diagnostic tracking
- **Parameters**: `file_path`, `old_string`, `new_string`, `replace_all?`
- **Pattern**: Clears LSP errors after successful edit

### FileWriteTool
- **Purpose**: Create or overwrite files
- **Parameters**: `file_path`, `content`
- **Constraint**: Must read file first before overwriting

### GlobTool
- **Purpose**: Fast file pattern matching (any codebase size)
- **Parameters**: `pattern`, `path?`
- **Pattern**: Returns sorted by modification time, excludes VCS dirs

### GrepTool
- **Purpose**: Content search via ripgrep
- **Parameters**: `pattern`, `path?`, `glob?`, `type?`, `output_mode`, `head_limit`, `offset`, `multiline?`, context flags
- **Pattern**: Supports regex, pagination (head_limit + offset), file type filtering

### NotebookEditTool
- **Purpose**: Edit Jupyter notebook cells
- **Parameters**: `notebook_path`, `new_source`, `cell_id?`, `cell_type?`, `edit_mode`

## Execution Tools

### BashTool
- **Purpose**: Execute shell commands
- **Key Features**: Security filtering (`bashSecurity.ts`), path validation, destructive command detection, sandbox support
- **Parameters**: `command`, `description?`, `timeout?`, `run_in_background?`
- **Sub-modules**: `commandSemantics.ts` (read/search/list classification), `sedValidation.ts`, `pathValidation.ts`
- **Pattern**: Complex regex-based command filtering, auto-sandbox

### PowerShellTool
- **Purpose**: Execute PowerShell commands (Windows)
- **Pattern**: Similar architecture to BashTool

## Web Tools

### WebFetchTool
- **Purpose**: Fetch URL content, convert HTML to Markdown
- **Key Features**: Auto HTTP→HTTPS upgrade, redirect handling, 15-min cache
- **Parameters**: `url`, `prompt`
- **Pattern**: Uses small/fast model to process fetched content

### WebSearchTool
- **Purpose**: Web search with result formatting
- **Parameters**: `query`, `allowed_domains?`, `blocked_domains?`
- **Pattern**: Returns search results as markdown with source links

## Agent & Communication Tools

### AgentTool
- **Purpose**: Spawn and manage sub-agents
- **Parameters**: `description`, `prompt`, `subagent_type?`, `model?`, `isolation?`, `run_in_background?`
- **Key Files**: `runAgent.ts`, `forkSubagent.ts`, `builtInAgents.ts`, `loadAgentsDir.ts`
- **Pattern**: Supports fork (context inheritance), background, worktree isolation

### SendMessageTool
- **Purpose**: Send messages to existing agents
- **Parameters**: `to` (agent ID or name), `message`
- **Pattern**: Resumes agent with full context preserved

### TeamCreateTool / TeamDeleteTool
- **Purpose**: Create/dissolve teams of agents
- **Pattern**: Swarm management for parallel execution

## Task Management Tools

### TaskCreateTool
- **Purpose**: Create background tasks
- **Pattern**: Registers task in AppState, runs asynchronously

### TaskGetTool / TaskOutputTool
- **Purpose**: Retrieve task output (blocking or non-blocking)
- **Parameters**: `task_id`, `block`, `timeout`

### TaskStopTool
- **Purpose**: Stop a running task
- **Parameters**: `task_id`

### TaskListTool
- **Purpose**: List all tasks (shared work queue in swarm mode)

### TaskUpdateTool
- **Purpose**: Update task metadata

## Planning & Organization Tools

### EnterPlanModeTool / ExitPlanModeTool
- **Purpose**: Switch to plan mode for complex tasks
- **Pattern**: Restricts to read-only tools, then presents plan for user approval

### TodoWriteTool
- **Purpose**: Structured task list management
- **Parameters**: `todos[]` with `content`, `status`, `activeForm`
- **Pattern**: Updates dedicated UI panel (not transcript)

## MCP (Model Context Protocol) Tools

### MCPTool
- **Purpose**: Execute MCP server tools
- **Key Feature**: Dynamically generated from connected MCP server schemas
- **Pattern**: Proxies tool calls to external MCP servers

### McpAuthTool
- **Purpose**: Handle OAuth for MCP services
- **Pattern**: Browser-based OAuth flow for Google Drive, GitHub, etc.

### ReadMcpResourceTool / ListMcpResourcesTool
- **Purpose**: Access MCP server resources

## Scheduling & Automation

### CronCreateTool
- **Purpose**: Schedule recurring or one-shot prompts
- **Parameters**: `cron` (5-field expression), `prompt`, `recurring?`, `durable?`
- **Pattern**: Off-minute scheduling to avoid fleet hotspots

### CronDeleteTool / CronListTool
- **Purpose**: Manage scheduled tasks

## Workspace Tools

### EnterWorktreeTool / ExitWorktreeTool
- **Purpose**: Create/manage git worktrees for isolated work
- **Pattern**: Creates branch in `.claude/worktrees/`

### ConfigTool
- **Purpose**: Read/update settings
- **Pattern**: Validates against supported settings schema

## Utility Tools

### AskUserQuestionTool
- **Purpose**: Interactive questions with options
- **Parameters**: `questions[]` with `question`, `options[]`, `multiSelect`, `header`
- **Pattern**: Supports preview rendering for comparison

### BriefTool
- **Purpose**: Share files/attachments with context
- **Pattern**: Upload and attach files to conversation

### SkillTool
- **Purpose**: Invoke slash commands (skills)
- **Parameters**: `skill`, `args?`
- **Pattern**: Dispatches to skill scripts in skills directory

### SleepTool
- **Purpose**: Wait/delay execution

### ToolSearchTool
- **Purpose**: Discover deferred tools by keyword
- **Pattern**: Matches `searchHint` and tool names

### LSPTool
- **Purpose**: Code intelligence (Go to Definition, Find References)
- **Pattern**: Communicates with language servers, maps editor coordinates

### RemoteTriggerTool
- **Purpose**: Trigger remote operations

### SyntheticOutputTool
- **Purpose**: Generate synthetic tool outputs for testing/replay

## Shared Infrastructure

### `src/tools/shared/spawnMultiAgent.ts`
- Shared logic for initializing teammates across backends (local, tmux, iTerm)

### `src/tools/shared/gitOperationTracking.ts`
- Tracks git changes for commit attribution

### `src/tools/utils.ts`
- Path expansion, error handling (ENOENT), filesystem abstraction (`getFsImplementation`)
