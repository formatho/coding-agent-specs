# Architecture Specification

## Overview

This document describes the high-level architecture of the coding agent system, including component interactions, data flow, and design decisions.

## System Architecture

### Component Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Interface Layer                      │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │   REPL   │  │  Print   │  │   SDK    │  │    UI    │       │
│  │   Mode   │  │   Mode   │  │   Mode   │  │Components│       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
└───────┼─────────────┼─────────────┼─────────────┼───────────────┘
        │             │             │             │
        └─────────────┴─────────────┴─────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                      Application Layer                           │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ Query Engine │  │   Session    │  │   Settings   │         │
│  │              │  │   Manager    │  │   Manager    │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
│         │                  │                  │                 │
│  ┌──────▼──────┐  ┌───────▼───────┐  ┌──────▼──────┐         │
│  │ Tool System │  │Task Management│  │   Storage   │         │
│  └─────────────┘  └───────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                      Integration Layer                           │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │   MCP    │  │   LSP    │  │   Git    │  │   API    │       │
│  │ Clients  │  │  Manager │  │  Helper  │  │  Client  │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
└───────┼─────────────┼─────────────┼─────────────┼───────────────┘
        │             │             │             │
        └─────────────┴─────────────┴─────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       Platform Layer                             │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │   File   │  │  Shell   │  │ Process  │  │  System  │       │
│  │  System  │  │  Exec    │  │ Manager  │  │  Utils   │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
└─────────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Entry Points

#### Main Entry (`main.tsx`)

```typescript
// Pseudocode
async function main() {
  // 1. Parse CLI arguments
  const args = parseArgv(process.argv)
  
  // 2. Determine mode
  const mode = args.print ? 'headless' : 'interactive'
  
  // 3. Initialize
  await init()
  
  // 4. Run appropriate mode
  if (mode === 'headless') {
    await runHeadless(args)
  } else {
    await runInteractive(args)
  }
}
```

**Responsibilities:**
- CLI argument parsing
- Mode determination (interactive vs headless)
- Initialization orchestration
- Graceful shutdown

### 2. Query Engine

```typescript
type QueryEngine = {
  // Process user input
  query(
    messages: Message[],
    context: ToolUseContext
  ): Promise<AssistantMessage>
  
  // Stream responses
  streamQuery(
    messages: Message[],
    context: ToolUseContext,
    onChunk: (chunk: StreamChunk) => void
  ): Promise<AssistantMessage>
}
```

**Responsibilities:**
- Message processing
- API communication
- Response streaming
- Tool orchestration
- Context management

### 3. Tool System

```typescript
type ToolSystem = {
  // Tool registry
  getTools(permissionContext: ToolPermissionContext): Tools
  
  // Tool execution
  executeTool(
    tool: Tool,
    input: unknown,
    context: ToolUseContext
  ): Promise<ToolResult>
  
  // Permission checking
  checkToolPermission(
    tool: Tool,
    input: unknown,
    context: ToolUseContext
  ): Promise<PermissionResult>
}
```

**Responsibilities:**
- Tool registration
- Tool execution
- Permission enforcement
- Progress tracking
- Result handling

### 4. State Management

```typescript
type StateManager = {
  // Get current state
  getState(): AppState
  
  // Update state
  setState(updater: (prev: AppState) => AppState): void
  
  // Subscribe to changes
  subscribe(listener: StateListener): Unsubscribe
}
```

**Responsibilities:**
- Central state store
- State updates
- Change notifications
- Persistence

### 5. Task Manager

```typescript
type TaskManager = {
  // Task creation
  createTask(
    type: TaskType,
    input: TaskInput
  ): Promise<TaskHandle>
  
  // Task control
  killTask(taskId: string): Promise<void>
  pauseTask(taskId: string): Promise<void>
  resumeTask(taskId: string): Promise<void>
  
  // Task queries
  getTask(taskId: string): TaskState | undefined
  getTasksByStatus(status: TaskStatus): TaskState[]
}
```

**Responsibilities:**
- Task lifecycle management
- Parallel execution
- Output streaming
- Notification dispatch

### 6. MCP Integration

```typescript
type MCPManager = {
  // Server management
  connectServer(config: McpServerConfig): Promise<void>
  disconnectServer(name: string): Promise<void>
  
  // Resource access
  listTools(): Promise<MCPTool[]>
  listResources(): Promise<MCPResource[]>
  callTool(name: string, args: unknown): Promise<unknown>
}
```

**Responsibilities:**
- MCP server connections
- Tool/resource discovery
- Message routing
- Error handling

### 7. Session Manager

```typescript
type SessionManager = {
  // Session persistence
  saveSession(session: Session): Promise<void>
  loadSession(sessionId: string): Promise<Session>
  
  // Session history
  listSessions(): Promise<SessionMetadata[]>
  deleteSession(sessionId: string): Promise<void>
}
```

**Responsibilities:**
- Session persistence
- Session recovery
- History management
- Metadata indexing

## Data Flow

### Interactive Mode Flow

```
User Input
    │
    ▼
┌─────────────┐
│ Parse Input │
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│ Add to Messages │
└────────┬────────┘
         │
         ▼
┌──────────────────┐
│  Query Engine    │
│  - Build context │
│  - Call API      │
└────────┬─────────┘
         │
         ▼
┌───────────────────┐
│ Stream Response   │
│ - Parse chunks    │
│ - Update UI       │
└────────┬──────────┘
         │
         ▼
┌────────────────────┐
│ Execute Tools      │◄────┐
│ - Check perms      │     │
│ - Run tool         │     │
│ - Get result       │     │
└────────┬───────────┘     │
         │                 │
         ▼                 │
┌────────────────────┐     │
│ Add Tool Results   │     │
└────────┬───────────┘     │
         │                 │
         └─────────────────┘
         │
         ▼
┌────────────────────┐
│ Complete Response  │
│ - Save session     │
│ - Update UI        │
└────────────────────┘
```

### Headless Mode Flow

```
CLI Input
    │
    ▼
┌─────────────────┐
│ Validate Args   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Create Context  │
└────────┬────────┘
         │
         ▼
┌──────────────────┐
│ Execute Query    │
│ - Single turn    │
│ - All tools      │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Format Output    │
│ - text           │
│ - json           │
│ - stream-json    │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Write to Stdout  │
└──────────────────┘
```

## Design Patterns

### 1. Repository Pattern (State)

```typescript
// Centralized state access
const store = createStore(initialState, onChangeAppState)

// Functional updates
store.setState(prev => ({
  ...prev,
  messages: [...prev.messages, newMessage]
}))

// Subscribe to changes
store.subscribe((state, prevState) => {
  if (state.messages !== prevState.messages) {
    // Handle message changes
  }
})
```

### 2. Factory Pattern (Tools)

```typescript
// Tool factory
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def
  }
}
```

### 3. Strategy Pattern (Permissions)

```typescript
// Permission strategies
const permissionStrategies = {
  default: new DefaultPermissionStrategy(),
  auto: new AutoPermissionStrategy(),
  bypass: new BypassPermissionStrategy()
}

// Execute with strategy
const strategy = permissionStrategies[context.mode]
const result = await strategy.checkPermission(tool, input)
```

### 4. Observer Pattern (State Changes)

```typescript
// Observable state
type ObservableState = {
  subscribe(listener: StateListener): Unsubscribe
  notify(newState: AppState): void
}

// Usage
const unsubscribe = store.subscribe((state) => {
  updateUI(state)
})

// Later
unsubscribe()
```

### 5. Command Pattern (Task Execution)

```typescript
// Task command
type TaskCommand = {
  execute(): Promise<TaskResult>
  undo(): Promise<void>
  canUndo(): boolean
}

// Usage
const command = new BashCommand(input)
const result = await command.execute()
```

## Key Decisions

### 1. Why TypeScript?

**Decision:** Use TypeScript for the entire codebase.

**Rationale:**
- Type safety catches errors early
- Better IDE support (autocomplete, refactoring)
- Self-documenting code with interfaces
- Easier onboarding for new developers

**Trade-offs:**
- Build step required
- Slightly more verbose
- Type inference can be complex

### 2. Why Zod for Schemas?

**Decision:** Use Zod for input validation and schema definition.

**Rationale:**
- Runtime validation
- TypeScript type inference
- Composable schemas
- Detailed error messages

**Trade-offs:**
- Runtime overhead
- Additional dependency
- Learning curve

### 3. Why Immutable State?

**Decision:** Use immutable state updates.

**Rationale:**
- Predictable state changes
- Easy change detection
- Enables time-travel debugging
- Thread-safe (important for parallel tasks)

**Trade-offs:**
- More allocations
- Need to spread objects
- Can be verbose

### 4. Why Event-Driven Architecture?

**Decision:** Use events for cross-component communication.

**Rationale:**
- Loose coupling between components
- Easy to add new listeners
- Async-friendly
- Enables plugins/hooks

**Trade-offs:**
- Flow can be harder to follow
- Debugging events can be tricky
- Need to manage subscriptions

### 5. Why Separate Tool Execution?

**Decision:** Tools are separate from the query engine.

**Rationale:**
- Tools can be tested independently
- Easy to add new tools
- Clear permission boundaries
- Can be loaded dynamically

**Trade-offs:**
- More abstraction
- Need to pass context around
- Tool discovery overhead

## Performance Considerations

### 1. Lazy Loading

```typescript
// Defer heavy imports
const heavyModule = await import('./heavyModule.js')

// Load MCP servers on demand
if (needsMCP) {
  await loadMCPServers()
}
```

### 2. Caching

```typescript
// Cache expensive computations
const cache = new Map<string, CachedResult>()

function getCached(key: string, compute: () => Result): Result {
  if (cache.has(key)) {
    return cache.get(key)!
  }
  
  const result = compute()
  cache.set(key, result)
  return result
}
```

### 3. Streaming

```typescript
// Stream large responses
async function* streamResponse(query: string): AsyncIterator<Chunk> {
  const stream = await api.stream(query)
  
  for await (const chunk of stream) {
    yield chunk
  }
}
```

### 4. Parallelization

```typescript
// Execute independent operations in parallel
const [users, repos, branches] = await Promise.all([
  fetchUsers(),
  fetchRepos(),
  fetchBranches()
])
```

## Security Considerations

### 1. Permission System

```typescript
// Defense in depth
async function executeTool(tool: Tool, input: unknown) {
  // 1. Validate input
  await tool.validateInput(input)
  
  // 2. Check permissions
  const perm = await tool.checkPermissions(input, context)
  if (perm.behavior !== 'allow') {
    throw new PermissionDeniedError(perm.message)
  }
  
  // 3. Execute with sandboxing
  return sandboxExecute(() => tool.call(input))
}
```

### 2. Input Sanitization

```typescript
// Sanitize all user input
function sanitizeInput(input: string): string {
  return input
    .replace(/<script.*?>.*?<\/script>/gi, '')
    .replace(/javascript:/gi, '')
    .trim()
}
```

### 3. Path Validation

```typescript
// Validate file paths
function validatePath(path: string): string {
  const resolved = resolve(path)
  const cwd = process.cwd()
  
  if (!resolved.startsWith(cwd)) {
    throw new Error('Path traversal detected')
  }
  
  return resolved
}
```

### 4. Command Injection Prevention

```typescript
// Use safe command execution
function safeExec(command: string, args: string[]) {
  // Don't use shell: true
  return spawn(command, args, {
    shell: false,
    env: sanitizedEnv
  })
}
```

## Scalability

### Horizontal Scaling

- Stateless query engine
- External session storage
- Distributed task queue
- Load-balanced API calls

### Vertical Scaling

- Connection pooling
- Result caching
- Output streaming
- Efficient state updates

## Testing Strategy

### Unit Tests

```typescript
describe('Tool System', () => {
  it('should validate tool input', async () => {
    const result = await tool.validateInput(input)
    expect(result.result).toBe(true)
  })
})
```

### Integration Tests

```typescript
describe('Query Flow', () => {
  it('should execute query end-to-end', async () => {
    const response = await queryEngine.query(messages)
    expect(response.content).toBeDefined()
  })
})
```

### E2E Tests

```typescript
describe('CLI', () => {
  it('should run in headless mode', async () => {
    const output = await exec('claude -p "test"')
    expect(output).toContain('test')
  })
})
```

## Monitoring

### Metrics

```typescript
type Metrics = {
  queryLatency: Histogram
  toolExecutionTime: Histogram
  apiCalls: Counter
  errorRate: Gauge
  activeTasks: Gauge
}
```

### Logging

```typescript
// Structured logging
logger.info('Query executed', {
  queryId: '123',
  duration: 1234,
  toolCalls: 3
})
```

### Tracing

```typescript
// Distributed tracing
const span = tracer.startSpan('query')
// ... work ...
span.finish()
```

## Summary

This architecture provides:

- **Modularity**: Clear separation of concerns
- **Extensibility**: Easy to add tools, agents, MCP servers
- **Performance**: Streaming, caching, parallelization
- **Security**: Defense in depth, input validation
- **Scalability**: Stateless design, external storage
- **Testability**: Clear interfaces, dependency injection

Follow the patterns and conventions established here when extending the system.
