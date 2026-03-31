# Security Model

## Overview

The coding agent operates with significant system access. This document describes the security model, trust boundaries, and best practices for secure operation.

## Trust Model

### Trust Boundaries

```
┌─────────────────────────────────────────────┐
│           Trusted Zone                      │
│  ┌───────────────────────────────────────┐ │
│  │  Agent Core                           │ │
│  │  - Query Engine                       │ │
│  │  - Tool System                        │ │
│  │  - State Management                   │ │
│  └───────────────────────────────────────┘ │
│                    │                        │
│  ┌─────────────────▼──────────────────┐   │
│  │  Trusted Tools                     │   │
│  │  - Read (with path validation)     │   │
│  │  - Edit (with user approval)       │   │
│  │  - Bash (with sandboxing)          │   │
│  └────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│           Untrusted Zone                    │
│  - File system (outside sandbox)           │
│  - Network requests                        │
│  - User-provided code                      │
│  - External processes                      │
└─────────────────────────────────────────────┘
```

### Trust Levels

1. **Fully Trusted**
   - Agent core components
   - Built-in tools
   - User-approved operations

2. **Semi-Trusted**
   - MCP servers (user-configured)
   - Plugins (verified)
   - Skills (local files)

3. **Untrusted**
   - Network content
   - User code execution
   - File system (unapproved paths)

## Permission System

### Permission Modes

#### Default Mode
- All operations require user approval
- Safest option
- Recommended for new users

```typescript
const defaultMode = {
  mode: 'default',
  promptForAll: true,
  autoApprove: []
}
```

#### Auto Mode
- Safe operations auto-approved
- Risky operations require approval
- Balanced security

```typescript
const autoMode = {
  mode: 'auto',
  promptForAll: false,
  autoApprove: [
    'Read(*)',
    'Glob(*)',
    'Grep(*)'
  ],
  requireApproval: [
    'Edit(*)',
    'Write(*)',
    'Bash(rm:*)'
  ]
}
```

#### Bypass Mode
- No permission checks
- **Dangerous**: Only use in sandboxes
- No network access recommended

```typescript
const bypassMode = {
  mode: 'bypassPermissions',
  promptForAll: false,
  autoApprove: ['*'],
  requireApproval: [],
  warning: '⚠️  No permission checks enabled'
}
```

### Permission Rules

#### Rule Syntax

```
Tool(pattern)
Tool(arg1:pattern, arg2:pattern)
```

Examples:
```
Read(*)                    # All read operations
Read(src/**)               # Read in src directory
Bash(git:*)                # Git commands only
Bash(npm:test, npm:build)  # Specific npm scripts
Edit(package.json)         # Edit specific file
```

#### Rule Sources

1. **User Settings** (`~/.claude/settings.json`)
```json
{
  "allowedTools": ["Read(*)", "Edit(src/**)"]
}
```

2. **Project Settings** (`.claude/settings.json`)
```json
{
  "allowedTools": ["Bash(npm:*)", "Bash(git:*)"]
}
```

3. **Enterprise Policy** (managed)
```json
{
  "deniedTools": ["Bash(rm:*)", "Bash(sudo:*)"]
}
```

### Permission Flow

```
Tool Call
    │
    ▼
┌────────────────┐
│ Validate Input │
└────────┬───────┘
         │
         ▼
┌────────────────────┐
│ Check deny rules   │───── Deny ────► Reject
└────────┬───────────┘
         │ Not denied
         ▼
┌────────────────────┐
│ Check allow rules  │───── Allow ────► Execute
└────────┬───────────┘
         │ Not allowed
         ▼
┌────────────────────┐
│ Check ask rules    │───── Ask ─────► Prompt User
└────────┬───────────┘
         │ Default behavior
         ▼
┌────────────────────┐
│ Apply mode default │
└────────────────────┘
```

## Sandboxing

### Bash Sandbox

```typescript
type SandboxConfig = {
  // Allowed directories
  allowRead: string[]
  allowWrite: string[]
  
  // Environment restrictions
  env: {
    PATH: string
    HOME: string
  }
  
  // Resource limits
  timeout: number
  maxMemory: number
}
```

#### Implementation

```typescript
async function executeBashSandboxed(
  command: string,
  config: SandboxConfig
): Promise<Output> {
  // 1. Parse command
  const parsed = parseCommand(command)
  
  // 2. Validate against rules
  if (!isCommandAllowed(parsed, config)) {
    throw new Error('Command not allowed in sandbox')
  }
  
  // 3. Execute with restrictions
  const process = spawn(parsed.cmd, parsed.args, {
    cwd: config.cwd,
    env: config.env,
    timeout: config.timeout
  })
  
  // 4. Monitor resource usage
  monitorProcess(process, config.maxMemory)
  
  return collectOutput(process)
}
```

### File System Sandbox

```typescript
type FileSystemSandbox = {
  // Root directory
  root: string
  
  // Allowed paths (relative to root)
  allowRead: string[]
  allowWrite: string[]
  
  // Denied patterns
  deny: string[]
}
```

#### Path Validation

```typescript
function validatePath(
  path: string,
  sandbox: FileSystemSandbox
): string {
  // Resolve to absolute
  const resolved = resolve(sandbox.root, path)
  
  // Check for traversal
  if (!resolved.startsWith(sandbox.root)) {
    throw new Error('Path traversal attempt blocked')
  }
  
  // Check deny list
  for (const pattern of sandbox.deny) {
    if (matchPattern(resolved, pattern)) {
      throw new Error(`Access denied: ${pattern}`)
    }
  }
  
  // Check allow list
  const relativePath = relative(sandbox.root, resolved)
  const isAllowed = sandbox.allowRead.some(p => 
    matchPattern(relativePath, p)
  )
  
  if (!isAllowed) {
    throw new Error('Path not in allow list')
  }
  
  return resolved
}
```

## Input Validation

### Command Injection Prevention

```typescript
function sanitizeCommand(command: string): string[] {
  // Don't use shell interpretation
  const args = parseCommand(command)
  
  // Validate each part
  for (const arg of args) {
    if (containsShellMetacharacters(arg)) {
      throw new Error('Shell metacharacters not allowed')
    }
  }
  
  return args
}
```

### Path Traversal Prevention

```typescript
function preventTraversal(path: string): string {
  // Remove null bytes
  path = path.replace(/\0/g, '')
  
  // Resolve to absolute
  const resolved = resolve(path)
  
  // Check for directory traversal
  const parts = resolved.split('/')
  let depth = 0
  
  for (const part of parts) {
    if (part === '..') {
      depth--
      if (depth < 0) {
        throw new Error('Path traversal detected')
      }
    } else if (part !== '.' && part !== '') {
      depth++
    }
  }
  
  return resolved
}
```

### Input Size Limits

```typescript
const INPUT_LIMITS = {
  maxPromptLength: 100000,      // 100K chars
  maxFileSize: 50 * 1024 * 1024, // 50MB
  maxCommandLength: 10000,      // 10K chars
  maxGlobResults: 10000,        // 10K files
  maxReadSize: 2 * 1024 * 1024  // 2MB
}

function validateInputSize(input: string, limit: number) {
  if (input.length > limit) {
    throw new Error(`Input exceeds limit: ${limit} chars`)
  }
}
```

## Network Security

### URL Validation

```typescript
function validateURL(url: string): URL {
  const parsed = new URL(url)
  
  // Only allow safe protocols
  if (!['http:', 'https:'].includes(parsed.protocol)) {
    throw new Error(`Unsafe protocol: ${parsed.protocol}`)
  }
  
  // Block internal networks
  const hostname = parsed.hostname
  if (isInternalIP(hostname)) {
    throw new Error('Internal network access blocked')
  }
  
  return parsed
}

function isInternalIP(hostname: string): boolean {
  const internalRanges = [
    '127.0.0.0/8',    // Loopback
    '10.0.0.0/8',     // Class A
    '172.16.0.0/12',  // Class B
    '192.168.0.0/16', // Class C
    '169.254.0.0/16', // Link-local
    '::1',            // IPv6 loopback
    'fe80::/10'       // IPv6 link-local
  ]
  
  // Check if hostname resolves to internal IP
  try {
    const resolved = dns.resolve4(hostname)
    return internalRanges.some(range => 
      ipInRange(resolved, range)
    )
  } catch {
    return false
  }
}
```

### Request Limits

```typescript
const REQUEST_LIMITS = {
  maxRedirects: 5,
  timeout: 30000,       // 30 seconds
  maxResponseSize: 10 * 1024 * 1024, // 10MB
  rateLimit: {
    windowMs: 60000,    // 1 minute
    max: 60             // 60 requests
  }
}
```

## Data Protection

### Sensitive Data Detection

```typescript
const SENSITIVE_PATTERNS = [
  /api[_-]?key/i,
  /password/i,
  /secret/i,
  /token/i,
  /private[_-]?key/i,
  /[0-9]{16}/,          // Credit card
  /[0-9]{3}-[0-9]{2}-[0-9]{4}/  // SSN
]

function detectSensitiveData(content: string): boolean {
  for (const pattern of SENSITIVE_PATTERNS) {
    if (pattern.test(content)) {
      return true
    }
  }
  return false
}
```

### Data Sanitization

```typescript
function sanitizeForLogging(data: unknown): string {
  const str = JSON.stringify(data)
  
  // Redact sensitive patterns
  return str
    .replace(/"api[_-]?key":\s*"[^"]+"/gi, '"api_key":"[REDACTED]"')
    .replace(/"password":\s*"[^"]+"/gi, '"password":"[REDACTED]"')
    .replace(/"token":\s*"[^"]+"/gi, '"token":"[REDACTED]"')
}
```

## MCP Security

### Server Verification

```typescript
type MCPSecurityConfig = {
  // Only allow specific servers
  allowedServers: string[]
  
  // Verify server signatures
  verifySignatures: boolean
  
  // Sandbox server execution
  sandboxServers: boolean
}
```

### Message Validation

```typescript
function validateMCPMessage(
  message: MCPMessage,
  config: MCPSecurityConfig
): boolean {
  // Check server allowlist
  if (!config.allowedServers.includes(message.server)) {
    throw new Error(`Server not allowed: ${message.server}`)
  }
  
  // Validate message structure
  if (!isValidMessageStructure(message)) {
    throw new Error('Invalid message structure')
  }
  
  // Check for injection attempts
  if (containsInjectionPatterns(message.content)) {
    throw new Error('Potential injection detected')
  }
  
  return true
}
```

## Audit Logging

### Audit Events

```typescript
type AuditEvent = {
  timestamp: number
  type: 'tool_call' | 'permission_change' | 'config_change'
  user?: string
  tool?: string
  input?: unknown
  result?: 'allow' | 'deny' | 'error'
  reason?: string
}
```

### Logging Implementation

```typescript
function logAuditEvent(event: AuditEvent) {
  const logEntry = {
    ...event,
    // Sanitize sensitive data
    input: sanitizeForLogging(event.input)
  }
  
  // Write to audit log
  fs.appendFileSync(
    AUDIT_LOG_PATH,
    JSON.stringify(logEntry) + '\n'
  )
}
```

## Best Practices

### 1. Principle of Least Privilege

```typescript
// Bad: Grant all permissions
{
  "allowedTools": ["*"]
}

// Good: Grant only what's needed
{
  "allowedTools": [
    "Read(src/**)",
    "Edit(src/**)",
    "Bash(npm:test)"
  ]
}
```

### 2. Defense in Depth

```typescript
async function executeTool(tool: Tool, input: unknown) {
  // Layer 1: Input validation
  validateInput(input)
  
  // Layer 2: Permission check
  await checkPermission(tool, input)
  
  // Layer 3: Sandbox execution
  const result = await sandboxExecute(() => tool.call(input))
  
  // Layer 4: Output validation
  validateOutput(result)
  
  return result
}
```

### 3. Secure Defaults

```typescript
const SECURE_DEFAULTS = {
  permissionMode: 'default',     // Prompt for all
  sandboxEnabled: true,          // Use sandbox
  networkAccess: false,          // Block network
  timeout: 30000,                // 30s timeout
  maxMemory: 512 * 1024 * 1024   // 512MB limit
}
```

### 4. Explicit Over Implicit

```typescript
// Bad: Implicit allow
const tool = {
  name: 'DangerousTool',
  checkPermissions: () => ({ behavior: 'allow' })
}

// Good: Explicit check
const tool = {
  name: 'DangerousTool',
  async checkPermissions(input, context) {
    if (input.mode === 'delete') {
      return {
        behavior: 'ask',
        message: 'This will delete files. Continue?'
      }
    }
    return { behavior: 'allow' }
  }
}
```

### 5. Fail Secure

```typescript
async function checkPermission(tool: Tool, input: unknown) {
  try {
    const result = await tool.checkPermissions(input, context)
    
    // Explicit allow only
    if (result.behavior !== 'allow') {
      throw new PermissionDeniedError(result.message)
    }
    
    return result
  } catch (error) {
    // Fail secure: deny on error
    logError(error)
    throw new PermissionDeniedError('Permission check failed')
  }
}
```

## Security Checklist

### Before Running

- [ ] Review permission mode settings
- [ ] Check allowed/denied tool lists
- [ ] Verify sandbox configuration
- [ ] Review MCP server allowlist
- [ ] Check network access rules
- [ ] Review audit log configuration

### Regular Audits

- [ ] Review audit logs weekly
- [ ] Check for suspicious tool calls
- [ ] Verify permission rules are current
- [ ] Update denied patterns
- [ ] Review MCP server usage
- [ ] Check for sensitive data leaks

### Incident Response

1. **Immediate Actions**
   - Revoke compromised credentials
   - Block suspicious servers
   - Enable restrictive mode

2. **Investigation**
   - Review audit logs
   - Identify affected resources
   - Document incident

3. **Recovery**
   - Patch vulnerabilities
   - Update security rules
   - Notify affected users

## Security Contacts

- Security issues: security@example.com
- Bug reports: https://github.com/example/coding-agent/issues
- Documentation: https://docs.example.com/security

## Further Reading

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Node.js Security Best Practices](https://nodejs.org/en/docs/guides/security/)
- [Secure Coding Guidelines](https://wiki.mozilla.org/WebAppSec/Secure_Coding_Guidelines)

---

**Remember**: Security is everyone's responsibility. When in doubt, ask.
