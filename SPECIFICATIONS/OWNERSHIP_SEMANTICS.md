# Ownership & Memory Semantics Specification

## Overview

This specification defines ownership and memory management patterns for the coding agent. It's designed for languages with explicit memory management (Rust, Go) but applicable to all implementations.

**Design Principle**: Clear ownership, explicit lifetimes, automatic cleanup.

---

## Ownership Rules

### 1. Tool Inputs

**Rule**: Immutable by default, explicit clone/move

**Pattern**:
- **Read-only access**: Pass by reference (`&T` in Rust, `*T` in Go)
- **Modification needed**: Pass mutable reference (`&mut T`) or clone
- **Transfer ownership**: Move semantics (`T` in Rust) or explicit transfer

**Rust**:
```rust
// Read-only: borrow input
fn validate_input(&self, input: &Input) -> Result<(), Error> {
    if input.path.is_empty() {
        return Err(Error::InvalidInput);
    }
    Ok(())
}

// Need to modify: clone or mutable borrow
fn canonicalize_input(&self, input: &mut Input) {
    input.path = canonicalize(&input.path);
}

// Transfer ownership: move
fn consume_input(self, input: Input) -> Result<Output, Error> {
    // input is moved, can't be used by caller
    Ok(process(input))
}
```

**Go**:
```go
// Read-only: pass pointer (convention: don't modify)
func ValidateInput(input *Input) error {
    if input.Path == "" {
        return InvalidInputError{}
    }
    return nil
}

// Need to modify: pass pointer and document mutation
func CanonicalizeInput(input *Input) {
    input.Path = canonicalize(input.Path)
}

// Transfer: create copy explicitly
func ConsumeInput(input Input) (Output, error) {
    // caller's input is copied
    return process(input), nil
}
```

---

### 2. State Management

**Rule**: Shared state requires explicit synchronization

**Pattern**:
- **Single owner**: Direct ownership (`Box<T>` in Rust)
- **Shared ownership**: Reference counting (`Arc<T>` in Rust)
- **Mutable shared state**: Locking (`Arc<Mutex<T>>` or `Arc<RwLock<T>>`)

**Rust**:
```rust
use std::sync::{Arc, RwLock};

// Single owner (thread-local)
struct LocalState {
    messages: Vec<Message>,
}

// Shared ownership (multi-threaded)
struct SharedState {
    messages: Arc<RwLock<Vec<Message>>>,
}

impl SharedState {
    fn new() -> Self {
        Self {
            messages: Arc::new(RwLock::new(Vec::new())),
        }
    }
    
    fn add_message(&self, msg: Message) {
        let mut messages = self.messages.write().unwrap();
        messages.push(msg);
    }
    
    fn get_messages(&self) -> Vec<Message> {
        let messages = self.messages.read().unwrap();
        messages.clone()
    }
}
```

**Go**:
```go
// Single owner (goroutine-local)
type LocalState struct {
    messages []Message
}

// Shared ownership (concurrent access)
type SharedState struct {
    messages []Message
    mu       sync.RWMutex
}

func NewSharedState() *SharedState {
    return &SharedState{
        messages: []Message{},
    }
}

func (s *SharedState) AddMessage(msg Message) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.messages = append(s.messages, msg)
}

func (s *SharedState) GetMessages() []Message {
    s.mu.RLock()
    defer s.mu.RUnlock()
    
    // Return copy
    result := make([]Message, len(s.messages))
    copy(result, s.messages)
    return result
}
```

---

### 3. Task Lifecycle

**Rule**: Clear ownership transfer during task lifecycle

**Pattern**:
```
1. Task Created → Owned by caller
2. Task Submitted → Ownership transferred to queue
3. Task Executing → Owned by worker
4. Task Completed → Result ownership transferred to caller
5. Task Cleaned Up → All resources freed
```

**Rust**:
```rust
struct Task {
    id: String,
    input: TaskInput,      // Owned by task
    output_file: PathBuf,  // Owned by task
}

impl Task {
    fn new(input: TaskInput) -> Self {
        Self {
            id: generate_id(),
            input,
            output_file: generate_output_path(),
        }
    }
    
    fn execute(self) -> Result<TaskResult, TaskError> {
        // self is consumed (moved)
        let result = process(self.input)?;
        
        // Cleanup happens automatically via Drop
        Ok(result)
    }
}

impl Drop for Task {
    fn drop(&mut self) {
        // Automatic cleanup
        let _ = std::fs::remove_file(&self.output_file);
    }
}
```

**Go**:
```go
type Task struct {
    ID          string
    Input       TaskInput
    OutputFile  string
    cleanupOnce sync.Once
}

func NewTask(input TaskInput) *Task {
    return &Task{
        ID:         generateID(),
        Input:      input,
        OutputFile: generateOutputPath(),
    }
}

func (t *Task) Execute() (TaskResult, error) {
    defer t.Cleanup()
    
    result, err := process(t.Input)
    if err != nil {
        return TaskResult{}, err
    }
    
    return result, nil
}

func (t *Task) Cleanup() {
    t.cleanupOnce.Do(func() {
        os.Remove(t.OutputFile)
    })
}
```

---

## Memory Safety Patterns

### 1. RAII (Resource Acquisition Is Initialization)

**Concept**: Acquire resources in constructor, release in destructor

**Rust**:
```rust
struct FileHandle {
    path: PathBuf,
    file: File,
}

impl FileHandle {
    fn open(path: PathBuf) -> Result<Self, Error> {
        let file = File::open(&path)?;
        Ok(Self { path, file })
    }
    
    fn read(&mut self) -> Result<String, Error> {
        let mut contents = String::new();
        self.file.read_to_string(&mut contents)?;
        Ok(contents)
    }
}

impl Drop for FileHandle {
    fn drop(&mut self) {
        // Automatic cleanup when out of scope
        if let Err(e) = self.file.sync_all() {
            eprintln!("Failed to sync file: {}", e);
        }
    }
}

// Usage
{
    let mut handle = FileHandle::open("test.txt".into())?;
    let contents = handle.read()?;
    // handle automatically closed here
}
```

**Go**:
```go
type FileHandle struct {
    path string
    file *os.File
}

func OpenFileHandle(path string) (*FileHandle, error) {
    file, err := os.Open(path)
    if err != nil {
        return nil, err
    }
    return &FileHandle{path: path, file: file}, nil
}

func (h *FileHandle) Read() (string, error) {
    contents, err := io.ReadAll(h.file)
    if err != nil {
        return "", err
    }
    return string(contents), nil
}

func (h *FileHandle) Close() error {
    return h.file.Close()
}

// Usage with defer
func ProcessFile(path string) error {
    handle, err := OpenFileHandle(path)
    if err != nil {
        return err
    }
    defer handle.Close() // Automatic cleanup
    
    contents, err := handle.Read()
    if err != nil {
        return err
    }
    
    // Process contents
    return nil
}
```

---

### 2. Zero-Copy Patterns

**Concept**: Avoid unnecessary cloning/copies

**Rust**:
```rust
use std::borrow::Cow;

// Use references when possible
fn process_string(input: &str) -> String {
    // No copy until needed
    input.to_uppercase()
}

// Conditional cloning
fn process_maybe_clone(input: &str) -> Cow<str> {
    if input.contains("secret") {
        // Need to modify, so clone
        Cow::Owned(input.replace("secret", "***"))
    } else {
        // No modification needed, borrow
        Cow::Borrowed(input)
    }
}

// Zero-copy deserialization
#[derive(Deserialize)]
struct ToolInput<'a> {
    #[serde(borrow)]
    path: &'a str,  // Borrows from input string
}
```

**Go**:
```go
// Use pointers to avoid copying large structs
func ProcessLargeStruct(input *LargeStruct) error {
    // input is pointer, no copy
    return nil
}

// Use slices instead of arrays
func ProcessStrings(strings []string) {
    // slices share underlying array
    for _, s := range strings {
        process(s)
    }
}

// String builder for efficient concatenation
func BuildString(parts []string) string {
    var builder strings.Builder
    for _, part := range parts {
        builder.WriteString(part)
    }
    return builder.String()
}
```

---

### 3. Memory Pooling

**Concept**: Reuse allocations to reduce GC pressure

**Rust**:
```rust
use bytes::BytesMut;

// Use BytesMut for buffer reuse
fn process_with_pool() -> Vec<u8> {
    let mut buffer = BytesMut::with_capacity(1024);
    
    // Use buffer
    buffer.extend_from_slice(b"data");
    
    // Convert to Vec (or keep reusing)
    buffer.to_vec()
}
```

**Go**:
```go
import "sync"

var bufferPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 1024)
    },
}

func ProcessWithPool() {
    // Get buffer from pool
    buffer := bufferPool.Get().([]byte)
    defer bufferPool.Put(buffer) // Return to pool
    
    // Use buffer
    copy(buffer, []byte("data"))
}
```

---

## Thread Safety

### 1. Send + Sync Traits (Rust)

**Concept**: Types safe to transfer/share between threads

```rust
// Automatically implemented for types that are thread-safe
struct ThreadSafeData {
    id: String,
    value: Arc<RwLock<i32>>,
}

// This type is Send + Sync because:
// - String is Send + Sync
// - Arc<RwLock<i32>> is Send + Sync

// Use in multi-threaded context
fn spawn_thread(data: Arc<ThreadSafeData>) {
    tokio::spawn(async move {
        // data can be used here safely
        let mut value = data.value.write().unwrap();
        *value += 1;
    });
}
```

### 2. Mutex Patterns (Go)

**Concept**: Protect shared state with mutexes

```go
type ThreadSafeData struct {
    id    string
    value int
    mu    sync.RWMutex
}

func (d *ThreadSafeData) Increment() {
    d.mu.Lock()
    defer d.mu.Unlock()
    d.value++
}

func (d *ThreadSafeData) GetValue() int {
    d.mu.RLock()
    defer d.mu.RUnlock()
    return d.value
}

// Use in goroutine
func spawnGoroutine(data *ThreadSafeData) {
    go func() {
        data.Increment()
        fmt.Println(data.GetValue())
    }()
}
```

---

## Lifecycle Management

### 1. Tool Lifecycle

```
Creation Phase:
  1. Allocate tool struct
  2. Initialize fields
  3. Register with tool registry

Usage Phase:
  1. Borrow tool from registry
  2. Execute tool operation
  3. Return result

Cleanup Phase:
  1. Unregister from registry
  2. Release resources
  3. Drop tool struct
```

### 2. Task Lifecycle

```
Creation → Submission → Execution → Completion → Cleanup

Each phase has clear ownership:
  - Created by caller
  - Submitted to queue (ownership transfer)
  - Executed by worker (ownership transfer)
  - Result returned to caller
  - Cleaned up automatically
```

### 3. Session Lifecycle

```
Start → Active → Idle → End

Resources managed across lifecycle:
  - Session ID (unique, persistent)
  - Messages (accumulated during session)
  - Tasks (spawned during session)
  - Tools (registered for session)
```

---

## Best Practices

### 1. Prefer Stack Allocation
✅ Small structs on stack
❌ Unnecessary heap allocation

### 2. Use RAII for Resources
✅ Automatic cleanup via destructors
❌ Manual cleanup (error-prone)

### 3. Minimize Cloning
✅ Use references where possible
❌ Clone without thinking

### 4. Clear Ownership
✅ Each resource has one owner
❌ Ambiguous ownership

### 5. Thread Safety by Design
✅ Use concurrency primitives
❌ Rely on "happens to work"

---

## Memory Profiling

### Rust
```rust
// Use valgrind or heaptrack for profiling
// Use std::mem::size_of for size checking

use std::mem;

fn check_sizes() {
    println!("TaskInput: {} bytes", mem::size_of::<TaskInput>());
    println!("Task: {} bytes", mem::size_of::<Task>());
}
```

### Go
```go
// Use pprof for profiling
import _ "net/http/pprof"

// Profile memory
// go tool pprof http://localhost:6060/debug/pprof/heap

// Check size
import "unsafe"

func checkSizes() {
    fmt.Printf("Task: %d bytes\n", unsafe.Sizeof(Task{}))
}
```

---

## Summary

**Ownership Principles**:
1. **Clear ownership**: Every resource has an owner
2. **Explicit transfer**: Ownership transfers are obvious
3. **Automatic cleanup**: RAII/Drop/defer
4. **Minimal cloning**: Use references/borrows
5. **Thread-safe by default**: Use proper primitives

**Language Patterns**:
- **Rust**: Ownership system + RAII + Arc/RwLock
- **Go**: Explicit copying + defer + channels/mutex

**Memory Safety**:
- No use-after-free
- No double-free
- No data races
- Automatic resource cleanup

Follow these patterns for robust, memory-safe implementations.
