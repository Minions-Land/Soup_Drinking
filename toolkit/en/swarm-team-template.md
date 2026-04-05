🌐 [中文版](../zh-CN/Swarm团队模板.md) | [← README](../../README.en.md)  
🗺️ **Navigation**: [← Toolkit Index](../../README.en.md#toolkit) | [Tutorial EN](../../tutorial/en/) | [Reference](../../en/)

---

# Swarm/Team Template

> 🔧 [English](#english) | [中文](#中文)  
> 🗺️ **Navigation**: [← Toolkit Index](../../README.md#toolkit) | [Tutorial](../../tutorial/) | [Reference EN](../../en/) | [Reference ZH](../../zh-CN/)

A practical template for designing swarm and team patterns -- systems where multiple agents work together as peers, sharing a work queue and coordinating through message passing. Unlike the coordinator pattern (one leader, many workers), swarm patterns distribute authority and decision-making across agents.

## Shared Work Queue Pattern

The work queue is the central data structure that enables swarm coordination. Each agent pulls tasks, works on them, and pushes results and new tasks back.

```typescript
type WorkQueue = {
  tasks: Task[]
  completedTasks: Task[]
  claimedBy: Map<string, string>  // taskId -> agentId
}

type Task = {
  id: string
  description: string
  status: 'pending' | 'claimed' | 'in_progress' | 'completed' | 'failed'
  assignedTo?: string
  result?: string
  dependencies?: string[]   // taskIds that must complete first
  priority: number
  createdBy: string
  createdAt: string
  retryCount?: number
}
```

### Queue Implementation Options

**File-based queue (simplest):**

```
~/.agent/teams/{team_name}/queue.json
```

Every agent reads and writes this file. Use file locking or atomic writes to prevent corruption. Works for 2-5 agents. Degrades with more due to contention.

**Per-task file queue (better scaling):**

```
/tmp/scratchpad/{session}/queue/
  task-001.json
  task-002.json
  ...
```

One file per task. Agents claim tasks by renaming files (atomic on most filesystems). Scales better than a single-file queue because agents do not contend on the same file.

### Queue Protocol

```
1. Agent reads queue, finds unclaimed task with satisfied dependencies
2. Agent claims task (atomic write: status -> 'claimed', assignedTo -> agentId)
3. Agent works on task
4. Agent marks task completed with result, or failed with error
5. Agent may create new tasks discovered during work
6. Repeat from step 1 until no tasks remain
```

## Agent Autonomy vs Leader Control

The key design decision in swarm systems: how much autonomy does each agent have?

```
Full Autonomy                                    Full Control
|----+--------+---------+---------+-------------|
     |        |         |         |
   Swarm    Team     Managed    Directed    Single Agent
                      Team       Workers
```

### Swarm (High Autonomy)

- Agents decide what to work on next by pulling from the queue.
- No leader. Agents coordinate through the queue and mailbox only.
- Each agent has the same capabilities and permissions.
- Good for: parallel search, independent file processing, distributed testing.

```typescript
while (hasUncompletedTasks()) {
  const task = claimNextTask();
  if (!task) { await checkMailbox(); continue; }
  const result = await work(task);
  completeTask(task.id, result);
  const newTasks = discoverFollowUpTasks(result);
  for (const t of newTasks) addToQueue(t);
}
```

### Team (Balanced)

- A leader agent creates the initial work queue and assigns high-level tasks.
- Team members execute tasks autonomously and can create sub-tasks.
- Team members can message each other for coordination.
- Leader monitors progress and intervenes when agents get stuck.

```typescript
// Leader creates initial queue
const tasks = planWork(userRequest);
for (const t of tasks) addToQueue(t);

// Leader monitors progress
while (!allTasksCompleted()) {
  await sleep(checkInterval);
  const stuck = findStuckTasks();
  for (const t of stuck) reassignOrEscalate(t);
}
```

### Managed Team (Low Autonomy)

- Leader assigns specific tasks to specific agents.
- Agents report back to the leader after each task completes.
- Leader decides all next steps based on results.
- Essentially the coordinator pattern with peer-to-peer communication added.

## Permission Synchronization

In swarm systems, permission state must be consistent across agents. If one agent gets approval to edit files in `src/auth/`, all agents should inherit that approval. Inconsistent permissions lead to some agents being blocked while others proceed with the same operation.

```typescript
type SharedPermissionState = {
  sessionRules: PermissionRule[]      // Rules approved during this session
  approvedPatterns: string[]          // Patterns the user has approved
  deniedPatterns: string[]            // Patterns the user has denied
  lastUpdated: string                 // Timestamp for conflict resolution
}

// Stored at: ~/.agent/teams/{team_name}/permissions.json
```

### Permission Sync Protocol

```
1. Agent needs permission for action X
2. Check shared permission state for existing ruling
3. If found: apply it (no need to ask user again)
4. If not found: ask user (or bubble to leader)
5. Record the ruling in shared state
6. Other agents pick up the new ruling on next check
```

**Conflict resolution:** Last-write-wins with timestamps. If two agents ask about the same pattern simultaneously, the first user response wins and the second agent picks it up on its next permission check.

## Backend Options

The backend determines how agents are actually executed:

### In-Process (Default)

```
Main Process
+-- Agent Thread 1 (async)
+-- Agent Thread 2 (async)
+-- Agent Thread 3 (async)
```

- All agents share one Node.js process.
- Communication via in-memory queues and shared state.
- Simplest to implement. Limited by single-process memory and CPU.
- Permission prompts bubble to leader's terminal.
- Best for: 2-5 agents doing lightweight work.

### tmux Sessions

```
tmux session: team-alpha
+-- window 0: leader
+-- window 1: agent-search
+-- window 2: agent-implement
+-- window 3: agent-test
```

- Each agent runs as a separate CLI process in a tmux pane.
- Communication via file-based mailboxes.
- Agents can be individually monitored, restarted, or killed.
- Permission prompts appear in each agent's own pane.
- Sessions survive terminal close -- reconnect with `tmux attach`.
- Best for: developer-visible teams, debugging, long-running tasks.

```bash
tmux new-session -d -s team-alpha
tmux send-keys -t team-alpha "claude --agent leader --team alpha" Enter
tmux split-window -t team-alpha
tmux send-keys -t team-alpha "claude --agent worker --team alpha --name searcher" Enter
```

### iTerm2 (macOS)

- Similar to tmux but uses iTerm2's native split pane API.
- Richer UI: color-coded pane borders per agent, clickable links.
- Communication via file-based mailboxes (same protocol as tmux).
- Sessions lost on terminal close (unlike tmux).
- Best for: macOS development, visual monitoring.

### Docker/Container

- Each agent runs in an isolated container.
- Communication via shared volume or network.
- Full isolation: separate filesystems, network stacks, resource limits.
- Best for: untrusted agents, production deployments, CI/CD environments.

## Reconnection

Agents may disconnect due to network issues, crashes, or user interruption. The system must handle reconnection gracefully.

```typescript
type AgentState = {
  agentId: string
  teamName: string
  currentTask?: string          // Task being worked on when disconnected
  lastHeartbeat: string         // ISO timestamp
  status: 'active' | 'disconnected' | 'crashed'
  checkpoint?: {
    messagesProcessed: number
    lastToolCall: string
    intermediateResults: any
  }
}
```

### Reconnection Protocol

```
1. Agent starts up, checks for existing state file
2. If state exists with currentTask:
   a. Check if task is still claimed by this agent
   b. If yes: resume from checkpoint
   c. If no (reclaimed by another agent): start fresh
3. If no state exists: join team as new member
4. Register heartbeat timer (every 30 seconds)
```

## Crash Recovery

When an agent crashes mid-task, the system must detect the failure and recover the task.

```typescript
function detectCrashedAgents(team: Team) {
  const now = Date.now();
  for (const agent of team.agents) {
    const elapsed = now - new Date(agent.lastHeartbeat).getTime();
    if (elapsed > HEARTBEAT_TIMEOUT_MS) {
      agent.status = 'crashed';
      // Release claimed tasks back to queue
      for (const task of getTasksClaimedBy(agent.agentId)) {
        task.status = 'pending';
        task.assignedTo = undefined;
        task.retryCount = (task.retryCount || 0) + 1;
      }
      broadcastMessage(team, {
        from: 'system',
        text: `Agent ${agent.agentId} crashed. ${count} tasks released back to queue.`,
      });
    }
  }
}
```

### Recovery Strategies

| Strategy | When to Use | Trade-off |
|----------|------------|-----------|
| Retry task | Transient failure (network, timeout) | May re-do partial work |
| Reassign to different agent | Agent-specific failure | Other agent may hit the same issue |
| Escalate to leader | Repeated failures on same task | Leader intervention slows progress |
| Skip task | Non-critical task, deadline pressure | Incomplete result |
| Abort team | Critical task failed, no recovery path | All work lost |

## Mailbox System

Peer-to-peer communication between agents via file-based mailboxes:

```typescript
type TeammateMessage = {
  from: string
  text: string
  timestamp: string
  read: boolean
  summary?: string     // 5-10 word preview for quick scanning
}

// Each agent has an inbox:
// ~/.agent/teams/{team_name}/inboxes/{agent_name}.json

function sendToTeammate(teamName: string, to: string, message: string) {
  const inboxPath = `~/.agent/teams/${teamName}/inboxes/${to}.json`;
  // Use file locking for concurrent safety
  withLock(inboxPath, () => {
    const inbox = readJSON(inboxPath);
    inbox.push({ from: myName, text: message, timestamp: now(), read: false });
    writeJSON(inboxPath, inbox);
  });
}

function checkMailbox(teamName: string, myName: string): TeammateMessage[] {
  const inboxPath = `~/.agent/teams/${teamName}/inboxes/${myName}.json`;
  const inbox = readJSON(inboxPath);
  const unread = inbox.filter(m => !m.read);
  for (const m of unread) m.read = true;
  writeJSON(inboxPath, inbox);
  return unread;
}
```

## Implementation Checklist

1. Design the work queue data structure and storage backend (file vs per-task files).
2. Implement atomic task claiming (file lock or rename).
3. Build the mailbox system for peer-to-peer communication with file locking.
4. Implement heartbeat emission and crash detection.
5. Add permission synchronization across agents with shared state file.
6. Choose and implement the execution backend (in-process/tmux/iTerm2/container).
7. Build reconnection logic with checkpoint-based resumption.
8. Implement graceful shutdown (terminate message, timeout, force kill).
9. Add monitoring and logging for team-wide visibility.
10. Test with simulated crashes, network partitions, and concurrent task claiming.
---
📁 [← Toolkit Index](../../README.en.md#toolkit) | 🌐 [中文版](../zh-CN/Swarm团队模板.md)
