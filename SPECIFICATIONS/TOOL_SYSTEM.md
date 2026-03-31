# Tool System Specification

## Overview

The tool system is the foundation for extending the coding agent's capabilities. Each tool is a self-contained unit that performs a specific operation with built-in permission controls, progress tracking, and UI rendering.

**Language-Agnostic**: This specification defines interfaces and patterns that can be implemented in any language (TypeScript, Rust, Go, etc.).

## Core Concepts

### Tool Interface (Language-Agnostic)

Every tool must implement the `Tool` trait/interface with the following methods:

#### Required Methods

**name() → String**
- Returns the tool's unique identifier
- Must be unique across all registered tools
- Example: `"Read"`, `"Write"`, `"Bash"`

**input_schema() → JSONSchema**
- Returns JSON Schema for input validation
- Defines structure, types, and constraints
- Used for runtime validation before execution

**call(input: Input, context: ToolContext) → Result<ToolResult<Output>, ToolError>**
- Executes the tool's primary operation
- Takes validated input and execution context
- Returns either success (ToolResult) or error (ToolError)
- Must be async/non-blocking

**check_permissions(input: Input, context: PermissionContext) → Result<PermissionResult, PermissionError>**
- Determines if the operation is allowed
- Returns `Allow`, `Deny`, or `Ask` with message
- Can modify input (e.g., canonicalize paths)

#### Optional Methods

**validate_input(input: Input) → Result<ValidationResult, ValidationError>**
- Pre-execution validation
- Returns `Valid` or `Invalid` with reason
- Default: Always valid

**is_concurrency_safe(input: Input) → Boolean**
- Can this tool run in parallel with itself?
- Default: `false` (assume unsafe)

**is_read_only(input: Input) → Boolean**
- Does this tool modify system state?
- Default: `false` (assume writes)

**is_destructive(input: Input) → Boolean**
- Can this tool cause irreversible changes?
- Default: `false`

### Language-Specific Implementations

#### TypeScript Implementation
```typescript
interface Tool<Input, Output, Progress> {
  name: string;
  inputSchema: z.ZodSchema<Input>;
  
  async call(
    input: Input,
    context: ToolContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: (progress: Progress) => void
  ): Promise<ToolResult<Output>>;
  
  async checkPermissions(
    input: Input,
    context: PermissionContext
  ): Promise<PermissionResult>;
  
  isConcurrencySafe(input: Input): boolean;
  isReadOnly(input: Input): boolean;
}
```

#### Rust Implementation
```rust
#[async_trait]
pub trait Tool: Send + Sync {
    type Input: Serialize + for<'de> Deserialize<'de>;
    type Output: Serialize;
    type Progress: Serialize + Send;
    
    fn name(&self) -> &str;
    fn input_schema(&self) -> JsonSchema;
    
    async fn call(
        &self,
        input: Self::Input,
        context: ToolContext,
    ) -> Result<ToolResult<Self::Output>, ToolError>;
    
    async fn check_permissions(
        &self,
        input: &Self::Input,
        context: &PermissionContext,
    ) -> Result<PermissionResult, PermissionError>;
    
    fn is_concurrency_safe(&self, input: &Self::Input) -> bool {
        false // Default: unsafe
    }
    
    fn is_read_only(&self, _input: &Self::Input) -> bool {
        false // Default: writes
    }
}
```

#### Go Implementation
```go
type Tool interface {
    Name() string
    InputSchema() jsonschema.Schema
    
    Call(
        ctx context.Context,
        input interface{},
        toolCtx ToolContext,
    ) (ToolResult, error)
    
    CheckPermissions(
        ctx context.Context,
        input interface{},
        permCtx PermissionContext,
    ) (PermissionResult, error)
    
    IsConcurrencySafe(input interface{}) bool
    IsReadOnly(input interface{}) bool
}
```

---

## Building Tools

### Using Builder Pattern (Recommended)

Most implementations provide a builder/helper function to reduce boilerplate:

**TypeScript**: `buildTool()`
**Rust**: Implement `Tool` trait manually or use macro
**Go**: Implement `Tool` interface

### Example Tool Implementation

```typescript
interface Tool<Input extends AnyObject, Output, P extends ToolProgressData> {
  // Identity
  name: string
  aliases?: string[]
  searchHint?: string
  
  // Schema
  inputSchema: AnyObject
  inputJSONSchema?: ToolInputJSONSchema
  outputSchema?: z.ZodType<unknown>
  
  // Core Operations
  call(
    args: Input,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>
  ): Promise<ToolResult<Output>>
  
  // Permission & Validation
  checkPermissions(
    input: Input,
    context: ToolUseContext
  ): Promise<PermissionResult>
  validateInput?(
    input: Input,
    context: ToolUseContext
  ): Promise<ValidationResult>
  preparePermissionMatcher?(
    input: Input
  ): Promise<(pattern: string) => boolean>
  
  // Capabilities
  isEnabled(): boolean
  isConcurrencySafe(input: Input): boolean
  isReadOnly(input: Input): boolean
  isDestructive?(input: Input): boolean
  interruptBehavior?(): 'cancel' | 'block'
  
  // UI Rendering
  description(input: Input, options): Promise<string>
  userFacingName(input: Input): string
  renderToolUseMessage(input: Input, options): ReactNode
  renderToolResultMessage?(output: Output, ...): ReactNode
  renderToolUseProgressMessage?(progress, options): ReactNode
  renderToolUseRejectedMessage?(input, options): ReactNode
  renderToolUseErrorMessage?(result, options): ReactNode
  renderGroupedToolUse?(toolUses, options): ReactNode
  getToolUseSummary?(input: Input): string | null
  getActivityDescription?(input: Input): string | null
  
  // Result Handling
  mapToolResultToToolResultBlockParam(
    content: Output,
    toolUseID: string
  ): ToolResultBlockParam
  extractSearchText?(output: Output): string
  isResultTruncated?(output: Output): boolean
  
  // Advanced
  maxResultSizeChars: number
  strict?: boolean
  shouldDefer?: boolean
  alwaysLoad?: boolean
  mcpInfo?: { serverName: string; toolName: string }
  backfillObservableInput?(input: Record<string, unknown>): void
  inputsEquivalent?(a: Input, b: Input): boolean
}
```

## Building Tools

### Using `buildTool`

The `buildTool` helper provides sensible defaults:

```typescript
import { buildTool } from './Tool.js'
import { z } from 'zod'

const MyTool = buildTool({
  name: 'MyTool',
  
  inputSchema: z.object({
    path: z.string().describe('File path'),
    recursive: z.boolean().optional()
  }),
  
  async call(input, context, canUseTool, parentMessage, onProgress) {
    // Check permissions
    const canUse = await canUseTool('MyTool', input)
    if (!canUse.allowed) {
      return {
        data: { error: canUse.reason }
      }
    }
    
    // Report progress
    onProgress?.({
      toolUseID: context.toolUseId!,
      data: { type: 'my_progress', status: 'starting' }
    })
    
    // Do work
    const result = await performOperation(input)
    
    // Return result
    return {
      data: result,
      newMessages: [] // Optional
    }
  },
  
  async description(input) {
    return `Processing ${input.path}`
  },
  
  renderToolUseMessage(input, options) {
    return (
      <Box>
        <Text>Processing: {input.path}</Text>
      </Box>
    )
  }
})
```

### Default Behaviors

`buildTool` provides these defaults:

- `isEnabled()`: Returns `true`
- `isConcurrencySafe()`: Returns `false` (assume not safe)
- `isReadOnly()`: Returns `false` (assume writes)
- `isDestructive()`: Returns `false`
- `checkPermissions()`: Returns `{ behavior: 'allow', updatedInput }`
- `toAutoClassifierInput()`: Returns `''` (skip classifier)
- `userFacingName()`: Returns `name`

## Permission System

### Permission Flow

```
1. Tool Called
   ↓
2. validateInput() - Check if operation is valid
   ↓
3. checkPermissions() - Determine permission behavior
   ↓
4. canUseTool() - Hook + user permission checks
   ↓
5. Execute tool
```

### Permission Results

```typescript
type PermissionResult = 
  | { behavior: 'allow'; updatedInput?: Input }
  | { behavior: 'deny'; message: string }
  | { behavior: 'ask'; message: string }
```

### Permission Modes

```typescript
type PermissionMode =
  | 'default'      // Always ask
  | 'auto'         // Auto-approve safe operations
  | 'bypassPermissions'  // No checks (dangerous)
```

### Permission Rules

```typescript
type ToolPermissionRulesBySource = {
  [source: string]: {
    allow?: PermissionRule[]
    deny?: PermissionRule[]
    ask?: PermissionRule[]
  }
}

type PermissionRule = {
  tool: string
  parameters?: Record<string, unknown>
}
```

## Progress Tracking

### Progress Data

```typescript
type ToolProgressData = 
  | { type: 'bash'; command: string; status: string }
  | { type: 'web_search'; query: string }
  | { type: 'mcp'; server: string; operation: string }
  | { type: 'custom'; [key: string]: unknown }
```

### Progress Updates

```typescript
onProgress?.({
  toolUseID: 'tool_123',
  data: {
    type: 'bash',
    command: 'npm test',
    status: 'running'
  }
})
```

## Tool Registration

### Built-in Tools

Built-in tools are registered in `tools.ts`:

```typescript
export function getTools(
  permissionContext: ToolPermissionContext
): Tools {
  return [
    BashTool,
    ReadTool,
    EditTool,
    WriteTool,
    GlobTool,
    GrepTool,
    // ... more tools
  ].filter(tool => tool.isEnabled())
}
```

### MCP Tools

MCP tools are dynamically loaded from servers:

```typescript
// MCP tool from server
{
  name: 'mcp__server__tool',
  inputSchema: z.object({ ... }),
  mcpInfo: {
    serverName: 'server',
    toolName: 'tool'
  },
  isMcp: true
}
```

## Advanced Features

### Tool Deferment

Large tool schemas can be deferred to reduce prompt size:

```typescript
{
  name: 'ComplexTool',
  shouldDefer: true,  // Requires ToolSearch to load
  searchHint: 'jupyter notebook operations'
}
```

### Always Load

Critical tools can bypass deferment:

```typescript
{
  name: 'EssentialTool',
  alwaysLoad: true,  // Always in initial prompt
  shouldDefer: false
}
```

### Concurrency Safety

Mark tools safe for parallel execution:

```typescript
{
  name: 'ReadOnlyTool',
  isConcurrencySafe: (input) => input.mode === 'read',
  isReadOnly: () => true
}
```

### Interrupt Behavior

Control how tools handle interrupts:

```typescript
{
  name: 'LongRunningTool',
  interruptBehavior: () => 'block',  // or 'cancel'
}
```

## Best Practices

### 1. Validation First

```typescript
async validateInput(input, context) {
  if (!input.path.startsWith('/')) {
    return {
      result: false,
      message: 'Path must be absolute',
      errorCode: 400
    }
  }
  return { result: true }
}
```

### 2. Clear Descriptions

```typescript
async description(input) {
  // Good: Specific and actionable
  return `Editing ${input.path}: ${input.operation}`
  
  // Bad: Too vague
  return 'Editing file'
}
```

### 3. Meaningful Progress

```typescript
async call(input, context, canUseTool, parentMessage, onProgress) {
  onProgress?.({
    toolUseID: context.toolUseId,
    data: { type: 'progress', step: 'validating' }
  })
  
  // ... do work ...
  
  onProgress?.({
    toolUseID: context.toolUseId,
    data: { type: 'progress', step: 'complete', count: 42 }
  })
}
```

### 4. Safe Defaults

```typescript
{
  isDestructive: (input) => input.mode === 'delete',
  isConcurrencySafe: (input) => input.readOnly,
  maxResultSizeChars: 10000
}
```

### 5. Clear UI Feedback

```typescript
renderToolUseMessage(input) {
  return (
    <Box flexDirection="column">
      <Text bold>{this.userFacingName(input)}</Text>
      <Text dimColor>{input.path}</Text>
    </Box>
  )
}
```

## Testing Tools

### Unit Tests

```typescript
describe('MyTool', () => {
  it('should validate input', async () => {
    const result = await MyTool.validateInput(
      { path: 'relative/path' },
      mockContext
    )
    expect(result.result).toBe(false)
  })
  
  it('should execute correctly', async () => {
    const result = await MyTool.call(
      { path: '/abs/path' },
      mockContext,
      mockCanUseTool,
      mockMessage
    )
    expect(result.data.success).toBe(true)
  })
})
```

### Integration Tests

```typescript
describe('Tool Integration', () => {
  it('should check permissions', async () => {
    const permResult = await MyTool.checkPermissions(
      { path: '/protected' },
      restrictedContext
    )
    expect(permResult.behavior).toBe('ask')
  })
})
```

## Error Handling

### Tool Errors

```typescript
async call(input, context, canUseTool, parentMessage) {
  try {
    const result = await riskyOperation(input)
    return { data: result }
  } catch (error) {
    return {
      data: { error: error.message },
      newMessages: [
        {
          type: 'system',
          content: `Tool failed: ${error.message}`
        }
      ]
    }
  }
}
```

### Error Rendering

```typescript
renderToolUseErrorMessage(result, options) {
  return (
    <Box>
      <Text color="red">Error: {result.error}</Text>
    </Box>
  )
}
```

## Performance Considerations

### Large Outputs

```typescript
{
  maxResultSizeChars: 50000,  // 50KB limit
  
  mapToolResultToToolResultBlockParam(content, id) {
    if (JSON.stringify(content).length > this.maxResultSizeChars) {
      // Write to file, return path
      return {
        type: 'tool_result',
        content: `Output too large. Saved to: ${filePath}`,
        tool_use_id: id
      }
    }
    return {
      type: 'tool_result',
      content: JSON.stringify(content),
      tool_use_id: id
    }
  }
}
```

### Caching

```typescript
const cache = new Map<string, CachedResult>()

async call(input, context, canUseTool, parentMessage) {
  const key = JSON.stringify(input)
  if (cache.has(key)) {
    return cache.get(key)!
  }
  
  const result = await computeResult(input)
  cache.set(key, result)
  return result
}
```

## Summary

The tool system provides:

- **Standardized interface** for extending capabilities
- **Built-in permissions** with multiple modes
- **Progress tracking** for long operations
- **Customizable UI** for all states
- **Performance controls** for large outputs
- **Safety mechanisms** for destructive operations

Build tools using `buildTool` to get sensible defaults, then customize as needed for your specific use case.
