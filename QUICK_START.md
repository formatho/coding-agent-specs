# Quick Start Guide

Get up and running with the coding agent in 5 minutes!

## 🚀 Installation

### Prerequisites

- Node.js 18+ or Bun
- Anthropic API key

### Install

```bash
# Clone the repository
git clone https://github.com/YOUR_USERNAME/coding-agent-specs.git
cd coding-agent-specs

# Install dependencies
bun install
# or: npm install

# Set your API key
export ANTHROPIC_API_KEY=your_key_here
```

## 💬 Basic Usage

### Interactive Mode

```bash
# Start interactive REPL
bun run src/main.tsx

# With a specific model
bun run src/main.tsx --model opus

# Continue previous session
bun run src/main.tsx --continue
```

### Headless Mode

```bash
# Single query
bun run src/main.tsx -p "Explain this codebase"

# JSON output
bun run src/main.tsx -p "List all TODOs" --output-format json

# With context file
bun run src/main.tsx -p "Review this code" --file code.ts
```

## 🔧 Common Tasks

### Code Review

```bash
# Review current directory
bun run src/main.tsx -p "Review this code for security issues"

# Review specific file
bun run src/main.tsx -p "Review authentication.ts" --file auth/authentication.ts
```

### Refactoring

```bash
# Refactor with guidance
bun run src/main.tsx -p "Refactor the database layer to use connection pooling"

# With specific goals
bun run src/main.tsx -p "Optimize this function for performance" --file utils/heavy.ts
```

### Testing

```bash
# Generate tests
bun run src/main.tsx -p "Write unit tests for payment processing"

# Run existing tests
bun run src/main.tsx -p "Run the test suite and fix any failures"
```

### Documentation

```bash
# Generate docs
bun run src/main.tsx -p "Add JSDoc comments to all exported functions"

# Create README
bun run src/main.tsx -p "Create a README.md for this module"
```

## 🛠️ Configuration

### Settings File

Create `.claude/settings.json`:

```json
{
  "model": "sonnet",
  "permissionMode": "auto",
  "allowedTools": [
    "Read(*)",
    "Edit(src/**)",
    "Bash(npm:*)",
    "Bash(git:*)"
  ],
  "effort": "medium",
  "thinking": "adaptive"
}
```

### Environment Variables

```bash
# API Configuration
export ANTHROPIC_API_KEY=your_key
export ANTHROPIC_MODEL=claude-sonnet-4-6

# Behavior
export CLAUDE_CODE_PERMISSION_MODE=auto
export CLAUDE_CODE_EFFORT=high

# Debugging
export CLAUDE_CODE_DEBUG=1
export CLAUDE_CODE_VERBOSE=1
```

## 🔐 Permissions

### Permission Modes

1. **Default** (safest)
   - Prompts for all operations
   - Best for exploration

2. **Auto** (balanced)
   - Auto-approves safe operations
   - Prompts for risky ones

3. **Bypass** (dangerous)
   - No permission checks
   - Sandbox only!

### Setting Permissions

```bash
# Command line
bun run src/main.tsx --permission-mode auto

# Settings file
{
  "permissionMode": "auto"
}

# Environment variable
export CLAUDE_CODE_PERMISSION_MODE=auto
```

## 📝 Example Workflows

### 1. Bug Investigation

```bash
# Start session
bun run src/main.tsx

> I'm seeing an error in production: "Cannot read property 'id' of undefined"
> The error happens in the user service when updating profiles
> 
> Can you investigate?
```

### 2. Feature Development

```bash
# With specific requirements
bun run src/main.tsx -p "
Add rate limiting to the API with these requirements:
- 100 requests per minute per IP
- Redis-backed for distributed systems
- Include IP whitelist support
- Add unit tests
"
```

### 3. Code Cleanup

```bash
# Multiple tasks
bun run src/main.tsx -p "
Please:
1. Find all TODO comments
2. Remove console.log statements
3. Fix TypeScript strict mode errors
4. Update dependencies
5. Run tests to ensure nothing broke
"
```

## 🎯 Best Practices

### 1. Be Specific

❌ Bad:
```
"Fix the bug"
```

✅ Good:
```
"Fix the null pointer exception in UserService.updateProfile()
The error occurs when user.preferences is undefined"
```

### 2. Provide Context

❌ Bad:
```
"Optimize this"
```

✅ Good:
```
"Optimize the database query in getUserAnalytics()
Currently takes 5+ seconds. We need it under 500ms.
The function is in src/services/analytics.ts"
```

### 3. Use Constraints

❌ Bad:
```
"Add authentication"
```

✅ Good:
```
"Add JWT authentication with these constraints:
- Use Passport.js
- Include refresh tokens
- Support role-based access
- Don't modify the database schema
"
```

### 4. Review Changes

```bash
# Always review what changed
git diff

# Run tests
bun test

# Check for issues
bun run lint
```

## 🔍 Debugging

### Enable Debug Mode

```bash
# Debug everything
export CLAUDE_CODE_DEBUG=1

# Debug specific category
export CLAUDE_CODE_DEBUG=api,tools

# Verbose output
bun run src/main.tsx --verbose
```

### Common Issues

#### 1. Permission Denied

```bash
# Check permission mode
cat .claude/settings.json | grep permissionMode

# Or run with explicit mode
bun run src/main.tsx --permission-mode auto
```

#### 2. Model Not Found

```bash
# Check available models
bun run src/main.tsx --help | grep model

# Use explicit model
bun run src/main.tsx --model claude-sonnet-4-6
```

#### 3. Context Too Large

```bash
# Reduce context
bun run src/main.tsx -p "Explain this" --file specific-file.ts

# Or use smaller model
bun run src/main.tsx --model haiku
```

## 📚 Next Steps

1. **Read the Specs**
   - [Tool System](SPECIFICATIONS/TOOL_SYSTEM.md)
   - [Task Management](SPECIFICATIONS/TASK_MANAGEMENT.md)
   - [Architecture](SPECIFICATIONS/ARCHITECTURE.md)

2. **Explore Advanced Features**
   - Create custom agents
   - Add MCP servers
   - Write custom tools

3. **Join the Community**
   - Star the repo
   - Report issues
   - Contribute improvements

## 💡 Pro Tips

### Session Management

```bash
# Name your sessions
bun run src/main.tsx --name "feature-auth"

# Resume by name
bun run src/main.tsx --resume feature-auth

# List sessions
bun run src/main.tsx --resume
```

### Parallel Tasks

```bash
# Run multiple operations
bun run src/main.tsx -p "
In parallel:
1. Run npm test
2. Run npm lint
3. Run npm build
Then report all results
"
```

### Memory Files

Create `CLAUDE.md` in your project:

```markdown
# Project Context

## Architecture
- Frontend: React
- Backend: Node.js + Express
- Database: PostgreSQL

## Conventions
- Use TypeScript strict mode
- Follow Airbnb style guide
- Test coverage > 80%

## Important Files
- src/index.ts: Entry point
- src/config.ts: Configuration
- src/database.ts: DB connection
```

## 🆘 Getting Help

- **Documentation**: Check the `SPECIFICATIONS/` directory
- **Issues**: Open a GitHub issue
- **Community**: Join our Discord (coming soon)

## 🎉 You're Ready!

Start with a simple query:

```bash
bun run src/main.tsx -p "What can you help me with?"
```

Then explore more complex tasks as you get comfortable!

---

**Happy coding!** 🚀
