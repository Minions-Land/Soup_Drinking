🌐 [中文版](../../toolkit/zh-CN/任务系统模板.md) | [← README](../../README.en.md)  
🗺️ **Navigation**: [← Toolkit Index](../../README.en.md#toolkit) | [Tutorial EN](../../tutorial/en/) | [Reference](../../en/)

---

# Task System Template

> 🔧 [English](#english) | [中文](#中文)  
> 🗺️ **Navigation**: [← Toolkit Index](../README.md#toolkit) | [Tutorial](../tutorial/) | [Reference EN](../en/) | [Reference ZH](../zh-CN/)

A practical template for designing task systems in agent architectures. A task is any unit of work that has a lifecycle -- it starts, runs, may produce intermediate results, and eventually completes or fails. The task system manages this lifecycle, prevents zombies, and provides context isolation between concurrent tasks.

## Task Lifecycle State Machine

Every task follows a strict state machine. Transitions are one-way; a task cannot move backward except for retry (failed -> pending).

```
                 +----------+
                 |  CREATED |
                 +----+-----+
                      |
                      v
                 +----------+
            +--->| PENDING  |
            |    +----+-----+
            |         |
            |         v
            |    +----------+
            |    | RUNNING  |----> CANCELLED
            |    +----+-----+
            |         |
            |    +----+-----+
            |    |          |
            v    v          v
        +--------+    +---------+
        | FAILED |    |COMPLETED|
        +--------+    +---------+
```

```typescript
type TaskState = 'created' | 'pending' | 'running' | 'completed' | 'failed' | 'cancelled';

type Task = {
  id: string                    // Unique identifier (prefixed: "a" for agent, "b" for bash)
  state: TaskState
  type: string                  // 'agent' | 'shell' | 'build' | 'test' | 'workflow'
  description: string           // Human-readable description
  createdAt: string             // ISO timestamp
  startedAt?: string
  completedAt?: string
  result?: unknown              // Typed result on completion
  error?: string                // Error message on failure
  parentTaskId?: string         // Sub-task relationship
  agentId?: string              // Owning agent
  outputFile: string            // Disk path for streaming output
  outputOffset: number          // Read offset for incremental output reads
  notified: boolean             // Has parent been notified of completion?
  metadata: Record<string, unknown>
}
```

### State Transition Rules

| From | To | Trigger | Side Effects |
|------|----|---------|--------------|
| created | pending | Task registered with system | Assign ID, persist state, emit event |
| pending | running | Worker picks up task | Record startedAt, start heartbeat |
| running | completed | Work finishes successfully | Record result and completedAt, notify parent |
| running | failed | Work throws error | Record error, trigger retry or escalate |
| running | cancelled | User or system cancels | Signal AbortController, cleanup resources |
| failed | pending | Retry policy triggers | Increment retryCount, re-enqueue |

### Transition Guards

```typescript
function validateTransition(current: TaskState, next: TaskState): boolean {
  const allowed: Record<TaskState, TaskState[]> = {
    created:   ['pending'],
    pending:   ['running', 'cancelled'],
    running:   ['completed', 'failed', 'cancelled'],
    completed: [],          // terminal state -- no transitions out
    failed:    ['pending'], // retry is the only valid transition
    cancelled: [],          // terminal state -- no transitions out
  };
  return allowed[current].includes(next);
}
```

## Global State Registration

All tasks must be registered in a global task registry. This is the single source of truth for task tracking, UI display, cleanup, and zombie detection. The registry must be a process-wide singleton.

```typescript
class TaskRegistry {
  private tasks: Map<string, Task> = new Map();
  private watchers: Map<string, Set<(task: Task) => void>> = new Map();

  register(task: Task): void {
    if (this.tasks.has(task.id)) {
      // Re-registration (resume): carry forward UI state
      const existing = this.tasks.get(task.id)!;
      task.startedAt = existing.startedAt;  // keep original start time
    }
    this.tasks.set(task.id, task);
    this.emit('task:registered', task);
  }

  transition(taskId: string, newState: TaskState, data?: Partial<Task>): void {
    const task = this.tasks.get(taskId);
    if (!task) throw new Error(`Task ${taskId} not found`);
    if (!validateTransition(task.state, newState)) {
      throw new Error(`Invalid transition: ${task.state} -> ${newState}`);
    }
    task.state = newState;
    Object.assign(task, data);
    this.emit(`task:${newState}`, task);
    this.notifyWatchers(taskId, task);
  }

  getActiveTasks(): Task[] {
    return [...this.tasks.values()].filter(
      t => t.state === 'pending' || t.state === 'running'
    );
  }

  getTasksByAgent(agentId: string): Task[] {
    return [...this.tasks.values()].filter(t => t.agentId === agentId);
  }

  prune(maxAgeMs: number): number {
    const cutoff = Date.now() - maxAgeMs;
    let pruned = 0;
    for (const [id, task] of this.tasks) {
      const isTerminal = task.state === 'completed' || task.state === 'failed' || task.state === 'cancelled';
      if (isTerminal && task.notified && new Date(task.completedAt!).getTime() < cutoff) {
        this.tasks.delete(id);
        pruned++;
      }
    }
    return pruned;
  }
}

// Process-wide singleton
let _registry: TaskRegistry | null = null;
function getTaskRegistry(): TaskRegistry {
  if (!_registry) _registry = new TaskRegistry();
  return _registry;
}
```

## Background Execution

Background tasks run independently of the main conversation flow. They return a task ID immediately and deliver results via notifications.

```typescript
async function runBackgroundTask(config: {
  type: string
  description: string
  agentId: string
  work: (signal: AbortSignal) => Promise<unknown>
  onComplete?: (result: unknown) => void
  onError?: (error: Error) => void
}): Promise<string> {
  const registry = getTaskRegistry();
  const abortController = new AbortController();
  const outputFile = getTaskOutputPath(generateTaskId(config.type));

  const task: Task = {
    id: generateTaskId(config.type),
    state: 'created',
    type: config.type,
    description: config.description,
    agentId: config.agentId,
    createdAt: new Date().toISOString(),
    outputFile,
    outputOffset: 0,
    notified: false,
    metadata: {},
  };

  registry.register(task);
  registry.transition(task.id, 'pending');

  // Fire and forget -- do not await this promise
  (async () => {
    registry.transition(task.id, 'running', { startedAt: new Date().toISOString() });
    try {
      const result = await config.work(abortController.signal);
      registry.transition(task.id, 'completed', {
        result,
        completedAt: new Date().toISOString(),
      });
      config.onComplete?.(result);
    } catch (error) {
      const finalState = abortController.signal.aborted ? 'cancelled' : 'failed';
      registry.transition(task.id, finalState, {
        error: error instanceof Error ? error.message : String(error),
        completedAt: new Date().toISOString(),
      });
      if (!abortController.signal.aborted) config.onError?.(error as Error);
    }
  })();

  return task.id;
}
```

### Disk-Based Output Streaming

Task output is streamed to disk to avoid memory pressure. The parent reads output incrementally using offset tracking.

```typescript
async function getTaskOutputDelta(taskId: string, offset: number) {
  const outputPath = getTaskOutputPath(taskId);
  const stat = await fsStat(outputPath).catch(() => null);
  if (!stat || stat.size <= offset) return { content: null, newOffset: offset };

  const fd = await open(outputPath, 'r');
  try {
    const buffer = Buffer.alloc(stat.size - offset);
    await fd.read(buffer, 0, buffer.length, offset);
    return { content: buffer.toString('utf-8'), newOffset: stat.size };
  } finally {
    await fd.close();
  }
}
```

## Zombie Protection

A zombie task is one stuck in `running` state with no active worker. This happens when a worker crashes, the process is killed, or there is a bug in the task completion logic. Without cleanup, zombies accumulate as orphaned processes.

### When Zombies Occur

```
Parent Agent spawns Background Shell Task
  -> Parent Agent completes or is killed
  -> Shell Task is still running (ZOMBIE)

Sub-Agent spawns Background Shell Task
  -> Sub-Agent completes
  -> Shell Task is still running (ZOMBIE)
```

### Detection via Heartbeat

Long-running tasks must emit heartbeats. If a heartbeat is missed beyond a timeout threshold, the task is presumed dead.

```typescript
function withHeartbeat(taskId: string, work: (signal: AbortSignal) => Promise<unknown>,
  config: { intervalMs: number; timeoutMs: number }
): (signal: AbortSignal) => Promise<unknown> {
  return async (signal: AbortSignal) => {
    const registry = getTaskRegistry();
    const interval = setInterval(() => {
      const task = registry.get(taskId);
      if (task) task.metadata.lastHeartbeat = Date.now();
    }, config.intervalMs);
    try {
      return await work(signal);
    } finally {
      clearInterval(interval);
    }
  };
}
```

### Zombie Reaper

```typescript
function startZombieReaper(registry: TaskRegistry, config: {
  checkIntervalMs: number,    // default: 60000
  zombieTimeoutMs: number,    // default: 300000
  maxRetries: number,         // default: 3
}) {
  setInterval(() => {
    const now = Date.now();
    for (const task of registry.getActiveTasks()) {
      if (task.state !== 'running') continue;
      const lastBeat = (task.metadata.lastHeartbeat as number) || new Date(task.startedAt!).getTime();
      if (now - lastBeat > config.zombieTimeoutMs) {
        const retries = (task.metadata.retryCount as number) || 0;
        if (retries < config.maxRetries) {
          registry.transition(task.id, 'failed', { error: 'Zombie: heartbeat timeout' });
          registry.transition(task.id, 'pending', {
            metadata: { ...task.metadata, retryCount: retries + 1 },
          });
        } else {
          registry.transition(task.id, 'failed', {
            error: `Zombie: exceeded ${config.maxRetries} retries`,
          });
        }
      }
    }
  }, config.checkIntervalMs);
}
```

### Cleanup on Agent Exit

Every agent must kill its orphaned tasks in a `finally` block:

```typescript
try {
  // agent execution
} finally {
  killShellTasksForAgent(agentId, getAppState, setAppState);
  killMonitorTasksForAgent(agentId, getAppState, setAppState);
  releaseAgentStateEntries(agentId);
}
```

## AsyncLocalStorage Context Isolation

In Node.js, `AsyncLocalStorage` provides per-task context without passing context objects through every function call. This is critical for isolating state between concurrent tasks running in the same process.

```typescript
import { AsyncLocalStorage } from 'node:async_hooks';

type TaskContext = {
  taskId: string
  agentId: string
  workingDirectory: string
  permissions: PermissionState
  abortSignal: AbortSignal
  logger: Logger
}

const taskContextStorage = new AsyncLocalStorage<TaskContext>();

function getTaskContext(): TaskContext {
  const ctx = taskContextStorage.getStore();
  if (!ctx) throw new Error('Not running inside a task context');
  return ctx;
}

async function runInTaskContext(context: TaskContext, work: () => Promise<unknown>) {
  return taskContextStorage.run(context, work);
}
```

### Usage Pattern

```typescript
await runInTaskContext({
  taskId: task.id,
  agentId: agent.id,
  workingDirectory: '/project',
  permissions: agentPermissions,
  abortSignal: abortController.signal,
  logger: createTaskLogger(task.id),
}, async () => {
  // Any function anywhere in the call stack can access context
  const ctx = getTaskContext();
  ctx.logger.info('Starting work...');
  await readFile(path);   // readFile calls getTaskContext() internally
  await runShell(cmd);    // runShell checks ctx.permissions automatically
});
```

### Why AsyncLocalStorage?

- **Thread safety:** Each async task gets its own isolated context, even when multiple tasks run concurrently in the same Node.js process.
- **No prop drilling:** Context is available anywhere in the call stack without threading it through every function signature.
- **Automatic cleanup:** Context is garbage-collected when the async scope ends. No manual cleanup needed.
- **Nested contexts:** Inner `run()` calls create nested scopes that shadow outer contexts, enabling clean sub-task isolation.
- **Zero overhead:** AsyncLocalStorage is a Node.js built-in with negligible performance cost.

## Implementation Checklist

1. Define the task state machine with explicit transition rules and guard functions.
2. Build a global TaskRegistry singleton with registration, transition, query, and pruning.
3. Implement background task execution with AbortController and disk-based output streaming.
4. Add heartbeat emission for long-running tasks (30s interval, 90s timeout recommended).
5. Build a zombie reaper that detects stuck tasks and applies retry/fail policies.
6. Add cleanup in every agent's `finally` block to kill orphaned tasks.
7. Set up AsyncLocalStorage for per-task context isolation.
8. Build task notification system (XML messages injected into parent conversation).
9. Add monitoring: active task count, average duration, failure rate, zombie detections.
10. Test with concurrent tasks, simulated crashes, aborts, and retry scenarios.
---
📁 [← Toolkit Index](../../README.en.md#toolkit) | 🌐 [中文版](../../toolkit/zh-CN/任务系统模板.md)
