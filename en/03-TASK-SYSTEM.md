# Task System

> 🗺️ **Navigation**: [← README](../README.md) | [zh-CN version](../zh-CN/03-任务系统.md)  
> 📁 [Reference Index](../README.md#reference) | [Tutorial](../tutorial/) | [Toolkit](../toolkit/)

> Sources: `src/Task.ts`, `src/tasks/`, `src/tools/Task*Tool/`

## 1. Task Abstraction

Every long-running or backgroundable operation is a **Task**:

```
Task Lifecycle: pending → running → (completed | failed | killed)
```

All tasks registered in global `AppState` — enables UI tracking, zombie protection, and session persistence.

### Task Types

| Type | File | Description |
|------|------|-------------|
| `LocalShellTask` | `tasks/LocalShellTask/` | Background bash processes |
| `LocalAgentTask` | `tasks/LocalAgentTask/` | Local sub-agent execution |
| `RemoteAgentTask` | `tasks/RemoteAgentTask/` | Remote agent execution |
| `InProcessTeammateTask` | `tasks/InProcessTeammateTask/` | Swarm member in same process |
| `DreamTask` | `tasks/DreamTask/` | Background "dream" processing |
| `LocalMainSessionTask` | `tasks/LocalMainSessionTask.ts` | Main session tracking |

## 2. Task Management Tools

```
TaskCreateTool  → Create background task
TaskGetTool     → Retrieve task output
TaskListTool    → List all tasks (shared work queue)
TaskOutputTool  → Stream task output
TaskStopTool    → Stop/kill a task
TaskUpdateTool  → Update task metadata
```

## 3. Background Task Flow

```
1. Model calls TaskCreateTool (or AgentTool with background flag)
2. Task created, registered in AppState
3. Task runs asynchronously
4. Model can:
   a. Check progress via TaskGetTool
   b. Read output via TaskOutputTool (block=true/false)
   c. Stop via TaskStopTool
5. Completion notified via <task-notification> XML
6. Zombie protection: killShellTasksForAgent() on agent death
```

## 4. Persistence & Recovery

- Task states persisted to `.claude/projects/...`
- Enables session resumption after crash/restart
- `DreamTask` supports long-running background processing that survives session boundaries

## 5. Context Isolation

Uses `AsyncLocalStorage` for:
- Agent Context — tracks which agent a function call belongs to
- Teammate Context — isolates in-process teammates
- No manual context threading through call stacks
