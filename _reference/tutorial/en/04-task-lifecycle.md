🌐 [中文版](../../tutorial/zh-CN/04-任务生命周期.md) | [← README](../../README.en.md)  
🗺️ **Navigation**: [← Ch.3](03-multi-agent-system.md) | **Chapter 4** | [Ch.5 →](05-permission-security.md)  
📁 [Tutorial EN](../../tutorial/en/) | [Toolkit EN](../../toolkit/en/) | [Reference](../../en/)

---

# Chapter 4: Task Lifecycle

> 📖 [English](#english) | [中文](#中文)  
> 🗺️ **Navigation**: [← Ch.3](03-multi-agent-system.md) | **Chapter 4** | [Ch.5 →](05-permission-security.md)  
> 📁 [Tutorial Index](../README.md#tutorial) | [Toolkit](../toolkit/) | [Reference EN](../en/) | [Reference ZH](../zh-CN/)

<a name="english"></a>

## What Is a Task?

In Claude Code, a **Task** is a unit of work that can run independently of the main conversation loop. While the query loop (Chapter 1) handles synchronous tool calls -- the model requests a tool, waits for the result, and continues -- Tasks handle work that needs to run in the background, persist across turns, or be managed as a first-class entity.

Think of the distinction this way: a tool call is like calling a function. A task is like spawning a thread.

## The Task Abstraction

The Task interface is defined in `src/Task.ts`. Every task has these core properties:

```typescript
interface Task {
  id: string;                      // Unique identifier
  type: TaskType;                  // Category of task
  status: TaskStatus;              // Current lifecycle state
  description: string;             // Human-readable description
  createdAt: number;               // Timestamp
  result?: TaskResult;             // Final output (when completed)
  error?: string;                  // Error message (when failed)
  
  // Lifecycle methods
  start(): Promise<void>;
  cancel(): Promise<void>;
  getProgress(): TaskProgress;
}
```

Tasks are registered in `AppState` and can be queried, monitored, and cancelled through task management tools.

## Task Types

Claude Code supports several task types, each implemented in its own directory under `src/tasks/`:

| Type | File | Description |
|------|------|-------------|
| `LocalShellTask` | `src/tasks/LocalShellTask/` | Background bash processes |
| `LocalAgentTask` | `src/tasks/LocalAgentTask/` | Local sub-agent execution |
| `RemoteAgentTask` | `src/tasks/RemoteAgentTask/` | Remote agent execution |
| `InProcessTeammateTask` | `src/tasks/InProcessTeammateTask/` | Swarm member in same process |
| `DreamTask` | `src/tasks/DreamTask/` | Background "dream" processing |

Each type is a class that extends the base `Task` interface, adding type-specific behavior like output streaming or progress reporting.

## Lifecycle States

A task moves through a well-defined lifecycle:

```
  +---------+     start()     +-----------+
  | PENDING | --------------> | RUNNING   |
  +---------+                 +-----------+
                                |       |
                       complete |       | error/cancel
                                v       v
                          +-----------+  +--------+
                          | COMPLETED |  | FAILED |
                          +-----------+  +--------+
                                         |
                                         v
                                     +----------+
                                     | KILLED   |
                                     +----------+
```

**PENDING:** The task has been created but not yet started. It is registered in AppState but no work is happening.

**RUNNING:** The task is actively executing. It reports progress and can be cancelled.

**COMPLETED:** The task finished successfully. Its result is available.

**FAILED:** The task encountered an error. The error message is stored for inspection.

**KILLED:** The user or system explicitly terminated the task.

State transitions are enforced -- you cannot move from COMPLETED back to RUNNING. This prevents race conditions and makes task state predictable.

## AppState Registration

When a task is created, it is immediately registered in `AppState.tasks`, a centralized registry. This registration serves several purposes:

1. **Discovery.** The model can ask "what tasks are running?" and get a list via `TaskGetTool`.
2. **Status reporting.** The UI can show task progress to the user.
3. **Lifecycle management.** The system can cancel all running tasks on shutdown.
4. **Persistence.** Task metadata can be saved and restored across sessions.

## Background Execution

Background tasks run in a separate async context. This means they have their own:

- **AsyncLocalStorage context** (covered below)
- **Error boundary** -- a task failure does not crash the main conversation
- **Cancellation scope** -- cancelling a task does not affect the main loop
- **Output buffer** -- task output is collected and delivered when requested

The main conversation continues while tasks run. The model can check on tasks with `TaskGetTool`, read output with `TaskOutputTool`, or stop them with `TaskStopTool`. Completion is notified via `<task-notification>` XML injected into the message stream.

## Task Management Tools

The model interacts with tasks through dedicated tools:

- **TaskGetTool.** Lists all tasks with their current status, type, and progress.
- **TaskStopTool.** Cancels a running task by ID.
- **TaskOutputTool.** Retrieves the output of a running or completed task.
- **TaskCreateTool.** Creates a new background task (alternative to `AgentTool` with `run_in_background`).

These tools let the model manage tasks naturally within the conversation.

## Zombie Protection

A critical concern with background tasks is **zombie tasks** -- tasks that are stuck, leaked, or forgotten. Claude Code implements several defenses:

1. **Timeout enforcement.** Each task type has a maximum duration. If exceeded, the task is automatically cancelled.
2. **Health checks.** The task registry periodically pings running tasks. If a task stops responding, it is marked as FAILED.
3. **Session cleanup.** When a session ends, all running tasks are cancelled and their resources are released.
4. **Orphan detection.** If a task's parent agent is no longer active, the task is flagged for cleanup.
5. **Shell process cleanup.** `killShellTasksForAgent()` ensures all spawned bash processes are also cleaned up when an agent task is killed, preventing resource leaks.

These protections are essential for a long-running CLI tool. Without them, forgotten background agents could accumulate, consuming memory and API quota.

## AsyncLocalStorage Isolation

Node.js `AsyncLocalStorage` is used to give each task its own isolated context. This solves a fundamental problem: in a system with multiple concurrent tasks, how do you ensure that Task A's state does not leak into Task B?

Global variables are dangerous. Passing context through every function call is invasive. `AsyncLocalStorage` provides a third option: a context that flows automatically through async operations.

```typescript
const taskStorage = new AsyncLocalStorage<TaskContext>();

function startTask(task: Task) {
  taskStorage.run({ taskId: task.id, state: {} }, async () => {
    await task.start();
  });
}

// Inside any function called during task execution:
function getCurrentTask(): TaskContext {
  return taskStorage.getStore();  // Returns the task's own context
}
```

This gives each task its own "thread-local" storage without actual threads. Any function called during task execution can access the task's context without it being passed as a parameter. Changes in one task's context do not affect another's.

## Persistence

Tasks support persistence for crash recovery and session resumption:

- **In-progress tasks** are saved to disk with their current state. If the CLI crashes and restarts, tasks can be resumed or cleaned up.
- **Completed task results** are cached so they do not need to be re-computed.
- **Task metadata** (creation time, type, description) is preserved for session history.

Persistence uses file-based storage in the `.claude/projects/` directory, enabling session continuity across restarts.

## Design Takeaways

The Task system teaches several patterns applicable to any agent framework:

1. **Separate synchronous tools from asynchronous work.** Not everything fits in a request-response pattern. Tasks provide a clean model for background work.
2. **Register all work centrally.** A task registry enables monitoring, management, and cleanup. Never have work running that the system does not know about.
3. **Enforce lifecycle states.** Explicit state machines prevent impossible transitions and make debugging easier.
4. **Protect against zombies.** Any system that spawns background work must have timeout, health-check, and cleanup mechanisms.
5. **Use AsyncLocalStorage for isolation.** It provides per-task context without polluting function signatures or using global state.
---
📁 [← Back to Tutorial Index](../../README.en.md#tutorial) | 🌐 [中文版](../../tutorial/zh-CN/04-任务生命周期.md) | [Ch.5 →](05-permission-security.md)
