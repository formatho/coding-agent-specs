# Type Definitions Specification

## Overview

Language-agnostic type definitions for the coding agent. All types are defined without language-specific syntax.

---

## Core Types

### Task Types

```
enum TaskType {
    LocalBash,           // Shell command execution
    LocalAgent,          // Local subagent
    RemoteAgent,         // Remote/cloud agent
    InProcessTeammate,   // In-process teammate
    LocalWorkflow,       // Workflow execution
    MonitorMcp,          // MCP server monitoring
    Dream,               // Background speculation
}

enum TaskStatus {
    Pending,    // Queued, not started
    Running,    // Currently executing
    Completed,  // Finished successfully
    Failed,     // Errored out
    Killed,     // User cancelled
}

struct TaskState {
    id: String
    task_type: TaskType
    status: TaskStatus
    description: String
    tool_use_id: Option<String>
    start_time: Timestamp
    end_time: Option<Timestamp>
    total_paused_ms: Option<u64>
    output_file: Path
    output_offset: u64
    notified: Boolean
}
```

### Permission Types

```
enum PermissionMode {
    Default,    // Prompt for all operations
    Auto,       // Auto-approve safe operations
    Bypass,     // No permission checks
}

enum PermissionResult<Input> {
    Allow { updated_input: Option<Input> }
    Deny { message: String }
    Ask { message: String }
}

struct PermissionRule {
    tool: String
    parameters: Option<Map<String, Value>>
}

struct PermissionContext {
    mode: PermissionMode
    allow_rules: List<PermissionRule>
    deny_rules: List<PermissionRule>
    ask_rules: List<PermissionRule>
    additional_working_dirs: Map<String, WorkingDirectory>
}
```

### Tool Types

```
struct ToolResult<Output> {
    data: Output
    new_messages: Option<List<Message>>
    context_modifier: Option<Function(Context -> Context)>
    mcp_meta: Option<McpMeta>
}

struct ToolContext {
    tool_use_id: String
    session_id: String
    permission_context: PermissionContext
    working_directory: Path
    abort_signal: AbortSignal
}

struct McpMeta {
    _meta: Option<Map<String, Value>>
    structured_content: Option<Map<String, Value>>
}
```

### Message Types

```
enum Message {
    User(UserMessage)
    Assistant(AssistantMessage)
    System(SystemMessage)
    Tool(ToolMessage)
    Attachment(AttachmentMessage)
    Progress(ProgressMessage)
}

struct UserMessage {
    content: String | List<ContentBlock>
    timestamp: Timestamp
    uuid: Option<String>
    permission_mode: Option<PermissionMode>
    is_meta: Boolean
}

struct AssistantMessage {
    content: String
    tool_uses: List<ToolUse>
    timestamp: Timestamp
}

struct ToolUse {
    id: String
    name: String
    input: Value
}

struct ToolMessage {
    tool_use_id: String
    content: String | Value
    is_error: Boolean
}
```

### State Types

```
struct AppState {
    messages: List<Message>
    tasks: Map<String, TaskState>
    tool_permission_context: PermissionContext
    tools: List<Tool>
    mcp: McpState
    ui: UiState
    settings: Settings
}

struct McpState {
    clients: List<McpClient>
    tools: List<McpTool>
    resources: Map<String, List<Resource>>
    commands: List<McpCommand>
}

struct McpClient {
    name: String
    config: McpServerConfig
    status: McpStatus
    tools: List<McpTool>
}

enum McpStatus {
    Disconnected
    Connecting
    Connected
    Error
}

struct McpServerConfig {
    type: Stdio | Http | Sse | Sdk
    command: Option<String>
    args: Option<List<String>>
    env: Option<Map<String, String>>
    url: Option<String>
    scope: User | Project | Local | Dynamic
}
```

### Input/Output Types

```
struct ToolInput {
    // Generic input structure
    // Specific tools define their own schemas
}

struct ReadInput {
    path: String
    offset: Option<u64>
    limit: Option<u64>
}

struct WriteInput {
    path: String
    content: String
}

struct BashInput {
    command: String
    description: Option<String>
    timeout: Option<u64>
    tool_use_id: Option<String>
}

struct AgentInput {
    agent_type: String
    prompt: String
    model: Option<String>
    memory: Option<MemoryType>
}

enum MemoryType {
    Session
    Project
    User
}
```

---

## Result Types

### Standard Result Pattern

```
type Result<T, E> = 
    | Ok(T)
    | Err(E)
```

### Async Result

```
type AsyncResult<T, E> = Future<Output = Result<T, E>>
```

### Option Type

```
type Option<T> = 
    | Some(T)
    | None
```

---

## Collection Types

### Map Types

- **HashMap<K, V>**: Unordered, O(1) lookup, hash-based
- **BTreeMap<K, V>**: Ordered, O(log n) lookup, tree-based

### Sequence Types

- **Vec<T> / List<T>**: Dynamic array, O(1) append
- **LinkedList<T>**: Linked list, O(1) insert/delete

### Set Types

- **HashSet<T>**: Unordered, O(1) lookup
- **BTreeSet<T>**: Ordered, O(log n) lookup

---

## Language Mappings

### Rust

```
TaskType → enum TaskType
TaskState → struct TaskState
Result<T, E> → std::result::Result<T, E>
Option<T> → std::option::Option<T>
HashMap<K, V> → std::collections::HashMap<K, V>
Vec<T> → Vec<T>
```

### Go

```
TaskType → type TaskType string (const)
TaskState → type TaskState struct
Result<T, E> → (T, error)
Option<T> → *T (nil for None)
HashMap<K, V> → map[K]V
Vec<T> → []T
```

### TypeScript

```
TaskType → type TaskType = 'local_bash' | ...
TaskState → interface TaskState
Result<T, E> → type Result<T, E> = {ok: true, value: T} | {ok: false, error: E}
Option<T> → T | null | undefined
HashMap<K, V> → Map<K, V> or Record<K, V>
Vec<T> → T[]
```

---

## JSON Schema

All types should have corresponding JSON Schema for validation:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "id": {"type": "string"},
    "task_type": {
      "type": "string",
      "enum": ["local_bash", "local_agent", "remote_agent"]
    },
    "status": {
      "type": "string",
      "enum": ["pending", "running", "completed", "failed", "killed"]
    }
  },
  "required": ["id", "task_type", "status"]
}
```

---

## Type Guards (Runtime Validation)

### Pattern

```
function is_valid_task_state(value: Any): Boolean {
    return (
        is_string(value.id) &&
        is_task_type(value.task_type) &&
        is_task_status(value.status)
    )
}
```

### Rust Implementation

```rust
fn validate_task_state(value: &serde_json::Value) -> Result<TaskState, Error> {
    let id = value["id"].as_str()
        .ok_or(Error::InvalidField("id"))?
        .to_string();
    
    let task_type = TaskType::from_str(
        value["task_type"].as_str()
            .ok_or(Error::InvalidField("task_type"))?
    )?;
    
    // ... validate all fields
    
    Ok(TaskState { id, task_type, ... })
}
```

---

## Summary

These type definitions provide:
- ✅ Language-agnostic interface definitions
- ✅ Clear data structures
- ✅ Type safety across implementations
- ✅ JSON Schema compatibility
- ✅ Runtime validation patterns

Use these types as the foundation for all implementations.
