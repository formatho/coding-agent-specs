# Concurrency Patterns Specification

## Overview

This specification defines concurrency patterns for the coding agent. It provides language-agnostic patterns with implementations in Rust and Go.

**Design Principle**: Use message passing and explicit concurrency primitives. Avoid shared mutable state.

---

## Async Runtime Selection

### Rust
- **Recommended**: Tokio
- **Alternative**: async-std
- **Reasoning**: Mature ecosystem, good performance

### Go
- **Built-in**: Goroutines + channels
- **No external runtime needed**

---

## Core Concurrency Concepts

### 1. Async/Await Pattern

**Language-Agnostic**:
```
async function execute_tool(tool, input) -> Result<Output, Error> {
    validated = await tool.validate_input(input)
    if (validated.is_err()) {
        return validated.error
    }
    
    permission = await tool.check_permissions(input, context)
    match permission {
        Allow => return await tool.call(input),
        Deny(msg) => return Error::Permission(msg),
        Ask(msg) => {
            approved = await prompt_user(msg)
            if (approved) {
                return await tool.call(input)
            } else {
                return Error::Permission("Denied")
            }
        }
    }
}
```

**Rust (Tokio)**:
```rust
use tokio::time::{timeout, Duration};

async fn execute_tool(
    tool: &Tool,
    input: Input,
) -> Result<Output, ToolError> {
    // Validate input
    let validated = tool.validate_input(&input).await
        .map_err(|e| ToolError::InvalidInput(e.to_string()))?;
    
    // Check permissions
    let permission = tool.check_permissions(&input, &context).await
        .map_err(|e| ToolError::PermissionDenied(e.to_string()))?;
    
    match permission {
        Permission::Allow => {
            // Execute with timeout
            timeout(Duration::from_secs(30), tool.call(input))
                .await
                .map_err(|_| ToolError::Timeout(30000))?
        }
        Permission::Deny(msg) => Err(ToolError::PermissionDenied(msg)),
        Permission::Ask(msg) => {
            let approved = prompt_user(&msg).await?;
            if approved {
                tool.call(input).await
            } else {
                Err(ToolError::PermissionDenied("User denied".to_string()))
            }
        }
    }
}
```

**Go**:
```go
func ExecuteTool(
    ctx context.Context,
    tool Tool,
    input Input,
) (Output, error) {
    // Validate input
    validated, err := tool.ValidateInput(input)
    if err != nil {
        return Output{}, InvalidInputError{err.Error()}
    }
    
    // Check permissions
    permission, err := tool.CheckPermissions(ctx, input)
    if err != nil {
        return Output{}, PermissionError{err.Error()}
    }
    
    switch permission.(type) {
    case Allow:
        // Execute with timeout
        ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
        defer cancel()
        
        result, err := tool.Call(ctx, validated)
        if err != nil {
            if ctx.Err() == context.DeadlineExceeded {
                return Output{}, TimeoutError{30000}
            }
            return Output{}, err
        }
        return result, nil
        
    case Deny:
        return Output{}, PermissionError{permission.Message}
        
    case Ask:
        approved := PromptUser(ctx, permission.Message)
        if !approved {
            return Output{}, PermissionError{"User denied"}
        }
        return tool.Call(ctx, validated)
    }
}
```

---

## Concurrent Task Execution

### Pattern: Parallel Execution with Semaphore

Execute multiple tasks concurrently with a maximum concurrency limit.

**Rust**:
```rust
use tokio::sync::Semaphore;
use std::sync::Arc;
use futures::future::join_all;

async fn execute_tasks_concurrently(
    tasks: Vec<Task>,
    max_concurrent: usize,
) -> Vec<TaskResult> {
    let semaphore = Arc::new(Semaphore::new(max_concurrent));
    
    let handles: Vec<_> = tasks.into_iter().map(|task| {
        let sem = semaphore.clone();
        tokio::spawn(async move {
            // Acquire permit
            let _permit = sem.acquire().await.unwrap();
            
            // Execute task
            execute_task(task).await
        })
    }).collect();
    
    // Wait for all tasks
    let results: Vec<_> = join_all(handles).await
        .into_iter()
        .filter_map(|r| r.ok())
        .collect();
    
    results
}
```

**Go**:
```go
func ExecuteTasksConcurrently(
    ctx context.Context,
    tasks []Task,
    maxConcurrent int,
) []TaskResult {
    var wg sync.WaitGroup
    sem := make(chan struct{}, maxConcurrent)
    results := make([]TaskResult, len(tasks))
    
    for i, task := range tasks {
        wg.Add(1)
        go func(idx int, t Task) {
            defer wg.Done()
            
            // Acquire semaphore
            sem <- struct{}{}
            defer func() { <-sem }()
            
            // Execute task
            result := ExecuteTask(ctx, t)
            results[idx] = result
        }(i, task)
    }
    
    wg.Wait()
    return results
}
```

---

## Task Queue Pattern

### Pattern: Producer-Consumer with Channels

Process tasks through a queue with worker goroutines/tasks.

**Rust**:
```rust
use tokio::sync::{mpsc, oneshot};

enum TaskCommand {
    Submit {
        task: Task,
        response: oneshot::Sender<TaskResult>,
    },
    Kill {
        task_id: String,
    },
}

pub struct TaskQueue {
    sender: mpsc::Sender<TaskCommand>,
}

impl TaskQueue {
    pub fn new(max_concurrent: usize) -> Self {
        let (sender, receiver) = mpsc::channel(100);
        
        // Spawn worker
        tokio::spawn(async move {
            task_worker(receiver, max_concurrent).await;
        });
        
        Self { sender }
    }
    
    pub async fn submit(&self, task: Task) -> Result<TaskResult, Error> {
        let (response_tx, response_rx) = oneshot::channel();
        
        self.sender.send(TaskCommand::Submit {
            task,
            response: response_tx,
        }).await?;
        
        response_rx.await.map_err(|_| Error::ChannelClosed)
    }
}

async fn task_worker(
    mut receiver: mpsc::Receiver<TaskCommand>,
    max_concurrent: usize,
) {
    let semaphore = Arc::new(Semaphore::new(max_concurrent));
    
    while let Some(cmd) = receiver.recv().await {
        match cmd {
            TaskCommand::Submit { task, response } => {
                let sem = semaphore.clone();
                tokio::spawn(async move {
                    let _permit = sem.acquire().await.unwrap();
                    let result = execute_task(task).await;
                    let _ = response.send(result);
                });
            }
            TaskCommand::Kill { task_id } => {
                // Handle kill
            }
        }
    }
}
```

**Go**:
```go
type TaskCommand struct {
    Task     Task
    Response chan TaskResult
}

type TaskQueue struct {
    commands chan TaskCommand
}

func NewTaskQueue(maxConcurrent int) *TaskQueue {
    commands := make(chan TaskCommand, 100)
    
    queue := &TaskQueue{commands: commands}
    
    // Start worker
    go taskWorker(commands, maxConcurrent)
    
    return queue
}

func (q *TaskQueue) Submit(ctx context.Context, task Task) (TaskResult, error) {
    response := make(chan TaskResult, 1)
    
    select {
    case q.commands <- TaskCommand{Task: task, Response: response}:
        select {
        case result := <-response:
            return result, nil
        case <-ctx.Done():
            return TaskResult{}, ctx.Err()
        }
    case <-ctx.Done():
        return TaskResult{}, ctx.Err()
    }
}

func taskWorker(commands <-chan TaskCommand, maxConcurrent int) {
    sem := make(chan struct{}, maxConcurrent)
    
    for cmd := range commands {
        sem <- struct{}{}
        
        go func(task Task, response chan TaskResult) {
            defer func() { <-sem }()
            
            result := ExecuteTask(context.Background(), task)
            response <- result
            close(response)
        }(cmd.Task, cmd.Response)
    }
}
```

---

## Message Passing Patterns

### Channel Types

**1. mpsc (Multiple Producer, Single Consumer)**
- Use for: Task submission, event streams
- Rust: `tokio::sync::mpsc`
- Go: `chan T`

**2. oneshot (Single Producer, Single Consumer)**
- Use for: Request/response, cancellation
- Rust: `tokio::sync::oneshot`
- Go: `chan T` (buffer size 1)

**3. broadcast (Multiple Producer, Multiple Consumer)**
- Use for: Notifications, state changes
- Rust: `tokio::sync::broadcast`
- Go: Multiple channels or pub/sub

---

## Cancellation

### Pattern: Cancellation Token

**Rust**:
```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicBool, Ordering};

pub struct CancellationToken {
    is_cancelled: Arc<AtomicBool>,
}

impl CancellationToken {
    pub fn new() -> Self {
        Self {
            is_cancelled: Arc::new(AtomicBool::new(false)),
        }
    }
    
    pub fn cancel(&self) {
        self.is_cancelled.store(true, Ordering::SeqCst);
    }
    
    pub fn is_cancelled(&self) -> bool {
        self.is_cancelled.load(Ordering::SeqCst)
    }
    
    pub async fn cancelled(&self) {
        while !self.is_cancelled() {
            tokio::time::sleep(Duration::from_millis(10)).await;
        }
    }
}

// Usage
async fn execute_task(
    task: Task,
    token: CancellationToken,
) -> Result<Output, Error> {
    if token.is_cancelled() {
        return Err(Error::Cancelled);
    }
    
    let result = do_work(&task).await;
    
    if token.is_cancelled() {
        cleanup_partial_work();
        return Err(Error::Cancelled);
    }
    
    result
}
```

**Go**:
```go
type CancellationToken struct {
    cancelled int32
}

func NewCancellationToken() *CancellationToken {
    return &CancellationToken{cancelled: 0}
}

func (t *CancellationToken) Cancel() {
    atomic.StoreInt32(&t.cancelled, 1)
}

func (t *CancellationToken) IsCancelled() bool {
    return atomic.LoadInt32(&t.cancelled) == 1
}

// Usage
func ExecuteTask(ctx context.Context, task Task, token *CancellationToken) (Output, error) {
    if token.IsCancelled() {
        return Output{}, errors.New("cancelled")
    }
    
    result, err := DoWork(ctx, task)
    if err != nil {
        return Output{}, err
    }
    
    if token.IsCancelled() {
        CleanupPartialWork()
        return Output{}, errors.New("cancelled")
    }
    
    return result, nil
}
```

---

## Rate Limiting

### Pattern: Token Bucket

**Rust**:
```rust
use std::time::{Duration, Instant};

pub struct RateLimiter {
    permits_per_second: u32,
    interval: Duration,
    last_permit: std::sync::Mutex<Instant>,
}

impl RateLimiter {
    pub fn new(permits_per_second: u32) -> Self {
        Self {
            permits_per_second,
            interval: Duration::from_millis(1000 / permits_per_second as u64),
            last_permit: std::sync::Mutex::new(Instant::now()),
        }
    }
    
    pub async fn acquire(&self) {
        let duration = {
            let mut last = self.last_permit.lock().unwrap();
            let elapsed = last.elapsed();
            
            if elapsed < self.interval {
                self.interval - elapsed
            } else {
                *last = Instant::now();
                Duration::ZERO
            }
        };
        
        if !duration.is_zero() {
            tokio::time::sleep(duration).await;
        }
    }
}
```

**Go**:
```go
type RateLimiter struct {
    permitsPerSecond int
    interval         time.Duration
    lastPermit       time.Time
    mu               sync.Mutex
}

func NewRateLimiter(permitsPerSecond int) *RateLimiter {
    return &RateLimiter{
        permitsPerSecond: permitsPerSecond,
        interval:         time.Second / time.Duration(permitsPerSecond),
        lastPermit:       time.Now(),
    }
}

func (r *RateLimiter) Acquire() {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    elapsed := time.Since(r.lastPermit)
    if elapsed < r.interval {
        time.Sleep(r.interval - elapsed)
    }
    
    r.lastPermit = time.Now()
}
```

---

## Thread Safety

### Pattern: Thread-Safe State

**Rust (Arc<RwLock<T>>)**:
```rust
use std::sync::{Arc, RwLock};

pub struct AppState {
    tasks: Arc<RwLock<HashMap<String, TaskState>>>,
}

impl AppState {
    pub fn new() -> Self {
        Self {
            tasks: Arc::new(RwLock::new(HashMap::new())),
        }
    }
    
    pub fn add_task(&self, task: TaskState) {
        let mut tasks = self.tasks.write().unwrap();
        tasks.insert(task.id.clone(), task);
    }
    
    pub fn get_task(&self, id: &str) -> Option<TaskState> {
        let tasks = self.tasks.read().unwrap();
        tasks.get(id).cloned()
    }
    
    pub fn update_task<F>(&self, id: &str, f: F)
    where
        F: FnOnce(&mut TaskState),
    {
        let mut tasks = self.tasks.write().unwrap();
        if let Some(task) = tasks.get_mut(id) {
            f(task);
        }
    }
}
```

**Go**:
```go
type AppState struct {
    tasks map[string]TaskState
    mu    sync.RWMutex
}

func NewAppState() *AppState {
    return &AppState{
        tasks: make(map[string]TaskState),
    }
}

func (s *AppState) AddTask(task TaskState) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.tasks[task.ID] = task
}

func (s *AppState) GetTask(id string) (TaskState, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    task, ok := s.tasks[id]
    return task, ok
}

func (s *AppState) UpdateTask(id string, update func(*TaskState)) {
    s.mu.Lock()
    defer s.mu.Unlock()
    if task, ok := s.tasks[id]; ok {
        update(&task)
        s.tasks[id] = task
    }
}
```

---

## Best Practices

### 1. Prefer Message Passing
✅ Use channels for communication
❌ Avoid shared mutable state

### 2. Use Structured Concurrency
✅ Spawn tasks with clear parent/child relationships
❌ Fire-and-forget goroutines/tasks

### 3. Handle Cancellation
✅ Check cancellation tokens regularly
❌ Ignore cancellation requests

### 4. Limit Concurrency
✅ Use semaphores to prevent resource exhaustion
❌ Spawn unlimited concurrent tasks

### 5. Handle Errors
✅ Propagate errors from concurrent tasks
❌ Ignore errors in spawned tasks

### 6. Use Timeouts
✅ Set timeouts on all async operations
❌ Wait indefinitely for completion

---

## Testing Concurrency

### Unit Tests

```rust
#[tokio::test]
async fn test_concurrent_execution() {
    let tasks = vec
![Task::new(); 10];
    let results = execute_tasks_concurrently(tasks, 5)
.await;
    
    assert_eq!(results.len(), 10);
    assert!(results.iter().all(|r| r.is_ok()));
}

#[tokio::test]
async fn test_cancellation() {
    let token = CancellationToken::new();
    
    let handle = tokio::spawn(async move {
        let mut counter = 0;
        while !token.is_cancelled() {
            counter += 1;
            tokio::time::sleep(Duration::from_millis(10)).await;
        }
        counter
    });
    
    tokio::time::sleep(Duration::from_millis(50)).await;
    token.cancel();
    
    let count = handle.await.unwrap();
    assert!(count > 0 && count < 100);
}
```

---

## Summary

**Key Patterns**:
1. Async/await for non-blocking operations
2. Semaphore for concurrency limits
3. Channels for message passing
4. Cancellation tokens for graceful shutdown
5. Rate limiting for resource protection
6. Thread-safe state with proper locking

**Language Choice**:
- **Rust**: Tokio + channels + Arc<RwLock>
- **Go**: Goroutines + channels + mutex

Both approaches provide robust, scalable concurrency suitable for production use.
