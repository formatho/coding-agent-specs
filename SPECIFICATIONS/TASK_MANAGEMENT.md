# Task Management Specification

## Overview

The task management system handles asynchronous operations that can be tracked, paused, resumed, and killed. Tasks enable parallel execution of operations like shell commands, subagent spawning, and remote operations.

## Core Concepts

### Task Types

```typescript
type TaskType =
  | 'local_bash'           // Shell command execution
  | 'local_agent'          // Local subagent
  | 'remote_agent'         // Remote/cloud agent
  | 'in_process_teammate'  // In-process teammate
  | 'local_workflow'       // Workflow execution
  | 'monitor_mcp'          // MCP server monitoring
  | 'dream'                // Background speculation
```

### Task Status

```typescript
type TaskStatus =
  | 'pending'    // Queued, not started
  | 'running'    // Currently executing
  | 'completed'  // Finished successfully
  | 'failed'     // Errored out
  | 'killed'     // User cancelled
```

### Task State

```typescript
type TaskStateBase = {
  id: string
  type: TaskType
  status: TaskStatus
  description: string
  toolUseId?: string
  startTime: number
  endTime?: number
  totalPausedMs?: number
  outputFile: string
  outputOffset: number
  notified: boolean
}
```

## Task ID Generation

Tasks use prefixed IDs for identification:

```typescript
const TASK_ID_PREFIXES: Record<string, string> = {
  local_bash: 'b',         // babc123def
  local_agent: 'a',        // aabc123def
  remote_agent: 'r',       // rabc123def
  in_process_teammate: 't', // tabc123def
  local_workflow: 'w',     // wabc123def
  monitor_mcp: 'm',        // mabc123def
  dream: 'd'               // dabc123def
}

export function generateTaskId(type: TaskType): string {
  const prefix = TASK_ID_PREFIXES[type] ?? 'x'
  const bytes = randomBytes(8)
  let id = prefix
  for (let i = 0; i < 8; i++) {
    id += TASK_ID_ALPHABET[bytes[i]! % TASK_ID_ALPHABET.length]
  }
  return id
}
```

## Task Lifecycle

### 1. Task Creation

```typescript
type LocalShellSpawnInput = {
  command: string
  description: string
  timeout?: number
  toolUseId?: string
  agentId?: AgentId
  kind?: 'bash' | 'monitor'
}

function createTaskStateBase(
  id: string,
  type: TaskType,
  description: string,
  toolUseId?: string
): TaskStateBase {
  return {
    id,
    type,
    status: 'pending',
    description,
    toolUseId,
    startTime: Date.now(),
    outputFile: getTaskOutputPath(id),
    outputOffset: 0,
    notified: false
  }
}
```

### 2. Task Execution

```typescript
async function executeTask(task: Task, input: TaskInput): Promise<TaskResult> {
  // Create task state
  const taskId = generateTaskId(task.type)
  const taskState = createTaskStateBase(
    taskId,
    task.type,
    input.description,
    input.toolUseId
  )
  
  // Add to app state
  setAppState(prev => ({
    ...prev,
    tasks: [...prev.tasks, taskState]
  }))
  
  // Execute
  try {
    updateTaskStatus(taskId, 'running')
    const result = await task.execute(input)
    updateTaskStatus(taskId, 'completed')
    return result
  } catch (error) {
    updateTaskStatus(taskId, 'failed')
    throw error
  }
}
```

### 3. Task Monitoring

```typescript
type TaskContext = {
  abortController: AbortController
  getAppState: () => AppState
  setAppState: SetAppState
}

function monitorTask(taskId: string, context: TaskContext) {
  const interval = setInterval(() => {
    const task = getTaskById(taskId)
    
    // Check for abort
    if (context.abortController.signal.aborted) {
      killTask(taskId)
      clearInterval(interval)
      return
    }
    
    // Update progress
    if (task.status === 'running') {
      updateTaskProgress(taskId)
    }
    
    // Check for completion
    if (isTerminalTaskStatus(task.status)) {
      clearInterval(interval)
    }
  }, 1000)
}
```

### 4. Task Completion

```typescript
function completeTask(taskId: string, result: TaskResult) {
  setAppState(prev => ({
    ...prev,
    tasks: prev.tasks.map(task =>
      task.id === taskId
        ? {
            ...task,
            status: 'completed',
            endTime: Date.now(),
            result
          }
        : task
    )
  }))
}
```

## Task Operations

### Kill Task

```typescript
type Task = {
  name: string
  type: TaskType
  kill(taskId: string, setAppState: SetAppState): Promise<void>
}

// Example: Kill bash task
async function killBashTask(
  taskId: string,
  setAppState: SetAppState
): Promise<void> {
  const process = getProcessForTask(taskId)
  
  if (process) {
    process.kill('SIGTERM')
    
    // Wait for graceful shutdown
    await new Promise(resolve => {
      const timeout = setTimeout(() => {
        process.kill('SIGKILL')
        resolve(undefined)
      }, 5000)
      
      process.on('exit', () => {
        clearTimeout(timeout)
        resolve(undefined)
      })
    })
  }
  
  setAppState(prev => ({
    ...prev,
    tasks: prev.tasks.map(task =>
      task.id === taskId
        ? { ...task, status: 'killed', endTime: Date.now() }
        : task
    )
  }))
}
```

### Pause/Resume Task

```typescript
function pauseTask(taskId: string) {
  const process = getProcessForTask(taskId)
  if (process) {
    process.kill('SIGSTOP')
    updateTaskPauseTime(taskId, Date.now())
  }
}

function resumeTask(taskId: string) {
  const process = getProcessForTask(taskId)
  if (process) {
    process.kill('SIGCONT')
    updateTaskResumeTime(taskId)
  }
}

function updateTaskResumeTime(taskId: string) {
  setAppState(prev => ({
    ...prev,
    tasks: prev.tasks.map(task => {
      if (task.id !== taskId || !task.pauseStartTime) return task
      
      const pauseDuration = Date.now() - task.pauseStartTime
      return {
        ...task,
        pauseStartTime: undefined,
        totalPausedMs: (task.totalPausedMs || 0) + pauseDuration
      }
    })
  }))
}
```

## Task Output Management

### Output Storage

```typescript
function getTaskOutputPath(taskId: string): string {
  return join(
    getAppDataDir(),
    'tasks',
    `${taskId}.output`
  )
}
```

### Output Streaming

```typescript
async function streamTaskOutput(
  taskId: string,
  outputFile: string
): Promise<void> {
  const stream = createReadStream(outputFile, {
    start: getTaskOutputOffset(taskId)
  })
  
  for await (const chunk of stream) {
    // Process chunk
    processOutputChunk(taskId, chunk)
    
    // Update offset
    updateTaskOutputOffset(taskId, chunk.length)
  }
}
```

### Output Cleanup

```typescript
function cleanupTaskOutput(taskId: string) {
  const outputFile = getTaskOutputPath(taskId)
  
  // Keep output for completed tasks
  if (isTerminalTaskStatus(getTaskStatus(taskId))) {
    // Archive or delete after retention period
    scheduleOutputCleanup(taskId, RETENTION_PERIOD)
  }
}
```

## Task Queue Management

### Priority Queue

```typescript
type TaskQueue = {
  pending: TaskStateBase[]
  running: TaskStateBase[]
  completed: TaskStateBase[]
}

function addTaskToQueue(task: TaskStateBase, priority: number = 0) {
  setAppState(prev => {
    const tasks = [...prev.tasks, task]
    
    // Sort by priority and creation time
    tasks.sort((a, b) => {
      if (a.priority !== b.priority) {
        return (b.priority || 0) - (a.priority || 0)
      }
      return a.startTime - b.startTime
    })
    
    return { ...prev, tasks }
  })
}
```

### Concurrency Limits

```typescript
const MAX_CONCURRENT_TASKS = {
  local_bash: 5,
  local_agent: 3,
  remote_agent: 10,
  in_process_teammate: 5,
  local_workflow: 2,
  monitor_mcp: 1,
  dream: 2
}

function canStartTask(type: TaskType): boolean {
  const state = getAppState()
  const running = state.tasks.filter(
    t => t.type === type && t.status === 'running'
  )
  
  return running.length < MAX_CONCURRENT_TASKS[type]
}
```

## Task Notifications

### Notification System

```typescript
function notifyTaskCompletion(taskId: string) {
  const task = getTaskById(taskId)
  
  if (!task.notified) {
    // Mark as notified
    setAppState(prev => ({
      ...prev,
      tasks: prev.tasks.map(t =>
        t.id === taskId ? { ...t, notified: true } : t
      )
    }))
    
    // Send notification
    sendNotification({
      title: `Task ${task.status}`,
      message: task.description,
      type: task.status === 'completed' ? 'success' : 'error'
    })
  }
}
```

### Progress Notifications

```typescript
function notifyTaskProgress(taskId: string, progress: number) {
  const task = getTaskById(taskId)
  
  updateNotification({
    id: `task-${taskId}`,
    title: task.description,
    message: `${Math.round(progress * 100)}% complete`,
    progress
  })
}
```

## Task State Queries

### Get Tasks

```typescript
function getTasksByType(type: TaskType): TaskStateBase[] {
  return getAppState().tasks.filter(t => t.type === type)
}

function getTasksByStatus(status: TaskStatus): TaskStateBase[] {
  return getAppState().tasks.filter(t => t.status === status)
}

function getTaskById(id: string): TaskStateBase | undefined {
  return getAppState().tasks.find(t => t.id === id)
}
```

### Terminal Status Check

```typescript
export function isTerminalTaskStatus(status: TaskStatus): boolean {
  return status === 'completed' || status === 'failed' || status === 'killed'
}

function isTaskFinished(taskId: string): boolean {
  const task = getTaskById(taskId)
  return task ? isTerminalTaskStatus(task.status) : true
}
```

## Error Handling

### Task Failures

```typescript
async function handleTaskFailure(
  taskId: string,
  error: Error
): Promise<void> {
  setAppState(prev => ({
    ...prev,
    tasks: prev.tasks.map(task =>
      task.id === taskId
        ? {
            ...task,
            status: 'failed',
            endTime: Date.now(),
            error: error.message
          }
        : task
    )
  }))
  
  // Log error
  logError(error, { taskId })
  
  // Notify user
  notifyTaskCompletion(taskId)
}
```

### Retry Logic

```typescript
async function retryTask(
  taskId: string,
  maxRetries: number = 3
): Promise<void> {
  const task = getTaskById(taskId)
  const retryCount = task.retryCount || 0
  
  if (retryCount < maxRetries) {
    // Reset task state
    setAppState(prev => ({
      ...prev,
      tasks: prev.tasks.map(t =>
        t.id === taskId
          ? {
              ...t,
              status: 'pending',
              retryCount: retryCount + 1,
              startTime: Date.now()
            }
          : t
      )
    }))
    
    // Re-execute
    await executeTask(task)
  } else {
    // Mark as failed
    await handleTaskFailure(taskId, new Error('Max retries exceeded'))
  }
}
```

## Performance Monitoring

### Task Metrics

```typescript
type TaskMetrics = {
  totalTasks: number
  completedTasks: number
  failedTasks: number
  averageDuration: number
  typeBreakdown: Record<TaskType, number>
}

function calculateTaskMetrics(): TaskMetrics {
  const tasks = getAppState().tasks
  
  const completed = tasks.filter(t => t.status === 'completed')
  const failed = tasks.filter(t => t.status === 'failed')
  
  const durations = completed
    .filter(t => t.endTime)
    .map(t => t.endTime! - t.startTime)
  
  return {
    totalTasks: tasks.length,
    completedTasks: completed.length,
    failedTasks: failed.length,
    averageDuration: durations.length > 0
      ? durations.reduce((a, b) => a + b, 0) / durations.length
      : 0,
    typeBreakdown: tasks.reduce((acc, t) => {
      acc[t.type] = (acc[t.type] || 0) + 1
      return acc
    }, {} as Record<TaskType, number>)
  }
}
```

## Best Practices

### 1. Use Descriptive Descriptions

```typescript
// Good
const task = createTaskStateBase(
  id,
  'local_bash',
  'Running tests for authentication module'
)

// Bad
const task = createTaskStateBase(
  id,
  'local_bash',
  'Running command'
)
```

### 2. Set Appropriate Timeouts

```typescript
const task = {
  type: 'local_bash',
  timeout: input.timeout ?? 30000  // Default 30s
}
```

### 3. Clean Up Resources

```typescript
async function killTask(taskId: string) {
  // Kill process
  await killBashTask(taskId)
  
  // Clean up output file
  await fs.unlink(getTaskOutputPath(taskId))
  
  // Remove from state
  setAppState(prev => ({
    ...prev,
    tasks: prev.tasks.filter(t => t.id !== taskId)
  }))
}
```

### 4. Handle Abort Signals

```typescript
async function executeTask(
  input: TaskInput,
  signal: AbortSignal
): Promise<TaskResult> {
  if (signal.aborted) {
    throw new Error('Task aborted')
  }
  
  // Periodically check signal
  const checkAbort = () => {
    if (signal.aborted) {
      throw new Error('Task aborted')
    }
  }
  
  return performWork(input, checkAbort)
}
```

### 5. Track Progress

```typescript
async function executeLongTask(taskId: string) {
  const total = 100
  
  for (let i = 0; i < total; i++) {
    await processItem(i)
    
    // Update progress
    notifyTaskProgress(taskId, i / total)
  }
}
```

## Summary

The task management system provides:

- **Typed task definitions** for different operations
- **Lifecycle management** from creation to completion
- **Parallel execution** with concurrency controls
- **Output streaming** with persistent storage
- **Notification system** for status updates
- **Error handling** with retry logic
- **Performance monitoring** with metrics

Use tasks for any operation that needs tracking, cancellation, or parallel execution.
