# Coding Agent Specs 🤖

> Comprehensive specification kit for building AI-powered coding agents

This repository contains the architectural specifications, design patterns, and implementation guidelines extracted from a production coding agent system. Use these specs to understand, build, or contribute to coding agent frameworks.

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Core Systems](#core-systems)
- [Tool System](#tool-system)
- [Task Management](#task-management)
- [Permission System](#permission-system)
- [MCP Integration](#mcp-integration)
- [Agent System](#agent-system)
- [State Management](#state-management)
- [CLI Modes](#cli-modes)
- [Getting Started](#getting-started)

## Overview

This coding agent is a CLI-based AI assistant that operates in both interactive and headless modes. It provides:

- **Multi-mode operation**: Interactive REPL and script-friendly headless modes
- **Extensible tool system**: Pluggable tools with permission controls
- **Parallel task execution**: Concurrent operations with task management
- **MCP integration**: Model Context Protocol for external tool connections
- **Agent workflows**: Custom agent definitions for specialized tasks
- **Privacy-first**: Local execution with optional cloud features

## Architecture

### High-Level Design

```
┌─────────────────────────────────────────────────────────┐
│                    CLI Entry Point                       │
│                   (main.tsx)                            │
└────────────────┬────────────────────────────────────────┘
                 │
        ┌────────┴─────────┐
        │                  │
    ┌───▼────┐       ┌────▼───┐
    │ REPL   │       │ Print  │
    │ Mode   │       │ Mode   │
    └───┬────┘       └────┬───┘
        │                  │
        └────────┬─────────┘
                 │
    ┌────────────▼────────────┐
    │    Query Engine         │
    │  (Message Processing)   │
    └────────────┬────────────┘
                 │
    ┌────────────▼────────────┐
    │    Tool System          │
    │  (Tool Execution)       │
    └────────────┬────────────┘
                 │
    ┌────────────▼────────────┐
    │    Task Manager         │
    │  (Parallel Execution)   │
    └─────────────────────────┘
```

### Core Components

1. **CLI Layer** (`main.tsx`)
   - Command parsing with Commander.js
   - Mode selection (interactive vs headless)
   - Startup orchestration

2. **Query Engine** (`query.ts`)
   - Message processing
   - API communication
   - Streaming responses

3. **Tool System** (`Tool.ts`)
   - Tool definitions and schemas
   - Permission checking
   - Progress tracking
   - Result rendering

4. **Task Manager** (`Task.ts`)
   - Task lifecycle management
   - Parallel execution
   - Resource cleanup

5. **State Management** (`AppState.ts`)
   - Centralized state store
   - State mutations
   - Change notifications

## Core Systems

### Tool System

The tool system is the heart of the coding agent. Each tool implements:

```typescript
interface Tool<Input, Output, Progress> {
  // Identity
  name: string
  aliases?: string[]
  searchHint?: string
  
  // Schema
  inputSchema: ZodSchema<Input>
  outputSchema?: ZodSchema<Output>
  
  // Execution
  call(
    input: Input,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: (progress: Progress) => void
  ): Promise<ToolResult<Output>>
  
  // Permissions
  checkPermissions(
    input: Input,
    context: ToolUseContext
  ): Promise<PermissionResult>
  
  // Validation
  validateInput?(
    input: Input,
    context: ToolUseContext
  ): Promise<ValidationResult>
  
  // UI
  description(input: Input, options): Promise<string>
  renderToolUseMessage(input: Input, options): ReactNode
  renderToolResultMessage?(output: Output, ...): ReactNode
  
  // Capabilities
  isEnabled(): boolean
  isConcurrencySafe(input: Input): boolean
  isReadOnly(input: Input): boolean
  isDestructive?(input: Input): boolean
}
```

#### Tool Categories

**File Operations**
- `Read`: File reading with limits
- `Edit`: Surgical file edits
- `Write`: File creation/overwrite

**Execution**
- `Bash`: Shell command execution
- `Agent`: Subagent spawning

**Search & Discovery**
- `Glob`: File pattern matching
- `Grep`: Content search
- `WebSearch`: Web search integration

**Communication**
- `Message`: Channel messaging
- `Browser`: Browser automation
- `Canvas`: Canvas rendering

#### Tool Result Handling

```typescript
type ToolResult<T> = {
  data: T
  newMessages?: Message[]
  contextModifier?: (ctx: ToolUseContext) => ToolUseContext
  mcpMeta?: {
    _meta?: Record<string, unknown>
    structuredContent?: Record<string, unknown>
  }
}
```

### Task Management

Tasks represent asynchronous operations that can be tracked, paused, and killed.

#### Task Types

```typescript
type TaskType =
  | 'local_bash'      // Shell commands
  | 'local_agent'     // Local subagents
  | 'remote_agent'    // Remote agents
  | 'in_process_teammate'  // In-process teammates
  | 'local_workflow'  // Workflows
  | 'monitor_mcp'     // MCP monitoring
  | 'dream'           // Background tasks
```

#### Task Lifecycle

```typescript
type TaskStatus =
  | 'pending'    // Queued
  | 'running'    // Executing
  | 'completed'  // Success
  | 'failed'     // Error
  | 'killed'     // User cancelled
```

#### Task State

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

### Permission System

Three permission modes with different security postures:

#### Permission Modes

1. **Default Mode** (`'default'`)
   - Prompts for dangerous operations
   - User controls all permissions
   - Safest option

2. **Auto Mode** (`'auto'`)
   - Auto-approves safe operations
   - Prompts for risky operations
   - Balanced security

3. **Bypass Mode** (`'bypassPermissions'`)
   - No permission checks
   - For trusted sandboxes only
   - Maximum speed

#### Permission Rules

```typescript
type ToolPermissionContext = {
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
}
```

#### Permission Result

```typescript
type PermissionResult = 
  | { behavior: 'allow'; updatedInput?: unknown }
  | { behavior: 'deny'; message: string }
  | { behavior: 'ask'; message: string }
```

### MCP Integration

Model Context Protocol (MCP) enables external tool connections.

#### MCP Server Configuration

```typescript
type McpServerConfig = {
  type: 'stdio' | 'http' | 'sse' | 'sdk'
  command?: string
  args?: string[]
  env?: Record<string, string>
  url?: string
  scope: 'user' | 'project' | 'local' | 'dynamic'
}
```

#### MCP Client Lifecycle

1. **Discovery**: Load configs from files
2. **Connection**: Establish transport
3. **Initialization**: Exchange capabilities
4. **Tool Registration**: Register MCP tools
5. **Execution**: Handle tool calls
6. **Cleanup**: Disconnect on exit

### Agent System

Agents are custom workflows with specialized behaviors.

#### Agent Definition

```typescript
type AgentDefinition = {
  agentType: string
  description: string
  model?: string
  initialPrompt?: string
  prompt: string | (() => string)
  memory?: 'session' | 'project' | 'user'
  source: 'built-in' | 'custom'
  isEnabled: () => boolean
  getSystemPrompt: () => string | undefined
}
```

#### Agent Types

- **Built-in Agents**: Predefined agents for common tasks
- **Custom Agents**: User-defined agents from `.claude/agents/`

#### Agent Memory

Agents can persist context across sessions:

- **Session**: Per-session memory
- **Project**: Shared across project
- **User**: User-level memory

### State Management

Redux-like centralized state with immutable updates.

#### State Structure

```typescript
type AppState = {
  // Messages
  messages: Message[]
  
  // Tools
  toolPermissionContext: ToolPermissionContext
  tools: Tools
  
  // Tasks
  tasks: Map<string, TaskState>
  
  // MCP
  mcp: {
    clients: MCPServerConnection[]
    tools: MCPTool[]
    commands: MCPCommand[]
    resources: Record<string, ServerResource[]>
  }
  
  // UI
  notifications: Notification[]
  currentView: ViewType
  
  // Settings
  effortValue: EffortLevel
  fastMode?: FastModeState
  advisorModel?: string
  kairosEnabled?: boolean
}
```

#### State Updates

```typescript
// Functional updates
setAppState(prev => ({
  ...prev,
  messages: [...prev.messages, newMessage]
}))

// Change detection
onChangeAppState((newState, prevState) => {
  if (newState.messages !== prevState.messages) {
    // Handle message changes
  }
})
```

## CLI Modes

### Interactive Mode (REPL)

```bash
claude [prompt]
```

Features:
- Full TUI with Ink
- Real-time streaming
- Interactive permissions
- Session persistence
- Command history

### Headless Mode (Print)

```bash
claude -p "prompt"
```

Features:
- Non-interactive execution
- Structured output formats
- SDK integration
- CI/CD friendly

#### Output Formats

- `text`: Plain text (default)
- `json`: Single JSON result
- `stream-json`: Real-time streaming

## Getting Started

### Prerequisites

- Node.js 18+
- Bun or npm
- Anthropic API key

### Installation

```bash
# Clone the repository
git clone https://github.com/YOUR_USERNAME/coding-agent-specs.git

# Install dependencies
cd coding-agent-specs
bun install

# Set up environment
export ANTHROPIC_API_KEY=your_key_here
```

### Basic Usage

```bash
# Interactive mode
bun run src/main.tsx

# Headless mode
bun run src/main.tsx -p "Explain this codebase"

# With specific model
bun run src/main.tsx --model opus

# Resume previous session
bun run src/main.tsx --continue
```

### Creating Custom Tools

```typescript
import { buildTool } from './Tool.js'
import { z } from 'zod'

const MyTool = buildTool({
  name: 'MyTool',
  inputSchema: z.object({
    path: z.string(),
    content: z.string()
  }),
  
  async call(input, context, canUseTool, parentMessage) {
    // Implementation
    return {
      data: { success: true }
    }
  },
  
  async description(input) {
    return `Processing ${input.path}`
  },
  
  renderToolUseMessage(input) {
    return `Using MyTool on ${input.path}`
  }
})
```

### Creating Custom Agents

```bash
# Create agent definition
cat > .claude/agents/reviewer.md << 'EOF'
---
description: Code reviewer agent
model: sonnet
---

You are a code reviewer. Analyze code for:
- Bugs and errors
- Security issues
- Performance problems
- Style inconsistencies

Be thorough but constructive.
EOF

# Use the agent
claude --agent reviewer
```

## Contributing

Contributions are welcome! Please see our [Contributing Guide](CONTRIBUTING.md) for details.

## License

MIT License - see [LICENSE](LICENSE) for details.

---

Built with ❤️ for the AI coding community
