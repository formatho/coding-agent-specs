# Error Handling Specification

## Overview

This specification defines error handling patterns for the coding agent. It's designed to be language-agnostic and suitable for implementation in TypeScript, Rust, Go, or other languages.

**Design Principle**: Use explicit error types, not exceptions. Force error handling through type system.

---

## Error Type Hierarchy

### Base Error Types

```
Error (base trait/interface)
├── ToolError
│   ├── PermissionDenied(String)
│   ├── InvalidInput(String)
│   ├── ExecutionFailed(String)
│   ├── Timeout(u64 milliseconds)
│   ├── Aborted
│   └── NotFound(String)
│
├── TaskError
│   ├── TaskAborted
│   ├── TaskTimeout(u64 milliseconds)
│   ├── TaskFailed(String)
│   ├── ResourceExhausted
│   └── InvalidTaskId(String)
│
├── PermissionError
│   ├── RuleDenied(String)
│   ├── ModeRestricted(String)
│   └── InsufficientPrivileges
│
└── StateError
    ├── InvalidState
    ├── ConcurrentModification
    └── NotFound(String)
```

---

## Language-Specific Implementations

### TypeScript

```typescript
// Base error types
export type ToolError = 
  | { type: 'PermissionDenied'; message: string }
  | { type: 'InvalidInput'; message: string }
  | { type: 'ExecutionFailed'; message: string }
  | { type: 'Timeout'; milliseconds: number }
  | { type: 'Aborted' };

export type TaskError =
  | { type: 'TaskAborted' }
  | { type: 'TaskTimeout'; milliseconds: number }
  | { type: 'TaskFailed'; message: string }
  | { type: 'ResourceExhausted' };

export type PermissionError =
  | { type: 'RuleDenied'; rule: string }
  | { type: 'ModeRestricted'; mode: string };

// Result type
export type Result<T, E> = 
  | { ok: true; value: T }
  | { ok: false; error: E };

// Usage
function executeTool(input: Input): Result<Output, ToolError> {
  if (!input.path) {
    return { ok: false, error: { type: 'InvalidInput', message: 'Path required' } };
  }
  return { ok: true, value: { data: 'success' } };
}
```

### Rust

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ToolError {
    #[error("Permission denied: {0}")]
    PermissionDenied(String),
    
    #[error("Invalid input: {0}")]
    InvalidInput(String),
    
    #[error("Execution failed: {0}")]
    ExecutionFailed(String),
    
    #[error("Timeout after {0}ms")]
    Timeout(u64),
    
    #[error("Operation aborted")]
    Aborted,
}

#[derive(Debug, Error)]
pub enum TaskError {
    #[error("Task aborted")]
    TaskAborted,
    
    #[error("Task timeout after {0}ms")]
    TaskTimeout(u64),
    
    #[error("Task failed: {0}")]
    TaskFailed(String),
    
    #[error("Resource exhausted")]
    ResourceExhausted,
}

// Usage with Result<T, E>
async fn execute_tool(input: Input) -> Result<Output, ToolError> {
    if input.path.is_empty() {
        return Err(ToolError::InvalidInput("Path required".to_string()));
    }
    Ok(Output { data: "success".to_string() })
}
```

### Go

```go
package errors

import "fmt"

type ToolError interface {
    error
    ToolError()
}

type PermissionDenied struct {
    Message string
}

func (e PermissionDenied) Error() string {
    return fmt.Sprintf("Permission denied: %s", e.Message)
}

func (e PermissionDenied) ToolError() {}

type InvalidInput struct {
    Message string
}

func (e InvalidInput) Error() string {
    return fmt.Sprintf("Invalid input: %s", e.Message)
}

func (e InvalidInput) ToolError() {}

// Usage
func ExecuteTool(input Input) (Output, error) {
    if input.Path == "" {
        return Output{}, InvalidInput{Message: "Path required"}
    }
    return Output{Data: "success"}, nil
}
```

---

## Error Propagation Patterns

### Pattern 1: Propagate with Context

Always add context when propagating errors across boundaries:

**Rust**:
```rust
async fn execute_tool(tool: &Tool, input: Input) -> Result<Output, ToolError> {
    let validated = tool.validate_input(&input)
        .map_err(|e| ToolError::InvalidInput(
            format!("Validation failed for {}: {}", tool.name(), e)
        ))?;
    
    tool.call(validated).await
}
```

**Go**:
```go
func ExecuteTool(tool Tool, input Input) (Output, error) {
    validated, err := tool.ValidateInput(input)
    if err != nil {
        return Output{}, fmt.Errorf("validation failed for %s: %w", 
            tool.Name(), err)
    }
    
    return tool.Call(validated)
}
```

### Pattern 2: Convert at Boundaries

Convert errors at layer boundaries:

```rust
// API layer
fn handle_request(req: Request) -> Result<Response, ApiError> {
    let result = execute_tool(req.input)
        .map_err(|e| match e {
            ToolError::PermissionDenied(msg) => ApiError::Forbidden(msg),
            ToolError::InvalidInput(msg) => ApiError::BadRequest(msg),
            ToolError::Timeout(_) => ApiError::RequestTimeout,
            _ => ApiError::Internal,
        })?;
    
    Ok(Response { result })
}
```

---

## Error Recovery Strategies

### Strategy 1: Retry with Exponential Backoff

```
Retry Configuration:
  max_retries: 3
  initial_delay: 1000ms
  max_delay: 30000ms
  backoff_multiplier: 2.0
  retryable_errors: [Timeout, ResourceExhausted]
```

**Rust**:
```rust
async fn execute_with_retry<F, T, E>(
    f: F,
    max_retries: u32,
) -> Result<T, E>
where
    F: Fn() -> std::pin::Pin<Box<dyn Future<Output = Result<T, E>>>>,
    E: std::fmt::Debug,
{
    let mut delay = 1000;
    
    for attempt in 0..=max_retries {
        match f().await {
            Ok(result) => return Ok(result),
            Err(e) if attempt < max_retries && is_retryable(&e) => {
                tokio::time::sleep(Duration::from_millis(delay)).await;
                delay = (delay * 2).min(30000);
            }
            Err(e) => return Err(e),
        }
    }
    
    unreachable!()
}
```

**Go**:
```go
func ExecuteWithRetry(
    ctx context.Context,
    fn func() (interface{}, error),
    maxRetries int,
) (interface{}, error) {
    var lastErr error
    delay := 1 * time.Second
    
    for attempt := 0; attempt <= maxRetries; attempt++ {
        result, err := fn()
        if err == nil {
            return result, nil
        }
        
        lastErr = err
        if attempt < maxRetries && isRetryable(err) {
            time.Sleep(delay)
            delay = time.Duration(float64(delay) * 2)
            if delay > 30*time.Second {
                delay = 30 * time.Second
            }
        }
    }
    
    return nil, lastErr
}
```

### Strategy 2: Circuit Breaker

```
Circuit Breaker Configuration:
  failure_threshold: 5
  success_threshold: 2
  timeout: 60000ms
  states: [Closed, Open, HalfOpen]
```

**Rust**:
```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};

pub struct CircuitBreaker {
    failures: Arc<AtomicUsize>,
    successes: Arc<AtomicUsize>,
    failure_threshold: usize,
    success_threshold: usize,
    state: Arc<AtomicU8>, // 0=Closed, 1=Open, 2=HalfOpen
}

impl CircuitBreaker {
    pub async fn call<F, T, E>(&self, f: F) -> Result<T, E>
    where
        F: Future<Output = Result<T, E>>,
    {
        let state = self.state.load(Ordering::SeqCst);
        
        match state {
            1 => Err("Circuit breaker is open"), // Open
            _ => {
                match f.await {
                    Ok(result) => {
                        self.on_success();
                        Ok(result)
                    }
                    Err(e) => {
                        self.on_failure();
                        Err(e)
                    }
                }
            }
        }
    }
    
    fn on_success(&self) {
        self.failures.store(0, Ordering::SeqCst);
        let successes = self.successes.fetch_add(1, Ordering::SeqCst) + 1;
        
        if successes >= self.success_threshold {
            self.state.store(0, Ordering::SeqCst); // Close
        }
    }
    
    fn on_failure(&self) {
        self.successes.store(0, Ordering::SeqCst);
        let failures = self.failures.fetch_add(1, Ordering::SeqCst) + 1;
        
        if failures >= self.failure_threshold {
            self.state.store(1, Ordering::SeqCst); // Open
        }
    }
}
```

### Strategy 3: Graceful Degradation

When errors occur, fall back to safe defaults:

```rust
async fn execute_tool(tool: &Tool, input: Input) -> Result<Output, ToolError> {
    match tool.call(input.clone()).await {
        Ok(result) => Ok(result),
        Err(ToolError::Timeout(_)) => {
            // Try simplified version
            warn!("Tool timeout, using simplified mode");
            tool.call_simplified(input).await
        }
        Err(ToolError::PermissionDenied(_)) => {
            // Return safe default
            warn!("Permission denied, returning default");
            Ok(Output::default())
        }
        Err(e) => Err(e),
    }
}
```

---

## Error Logging and Monitoring

### What to Log

**Always Log**:
- Error type and message
- Timestamp
- Tool/task ID
- User ID (if applicable)

**Conditionally Log**:
- Stack traces (only in debug mode)
- Input data (only if non-sensitive)
- Full context (only for critical errors)

**Never Log**:
- Passwords or credentials
- Full user prompts (unless explicitly enabled)
- PII (Personally Identifiable Information)

### Error Metadata

```typescript
interface ErrorMetadata {
  error_type: string;
  error_message: string;
  timestamp: number;
  tool_id?: string;
  task_id?: string;
  session_id?: string;
  retry_count?: number;
  duration_ms?: number;
}
```

---

## Error Handling Best Practices

### 1. Be Explicit

✅ **Good**: Use Result types, force handling
❌ **Bad**: Throw exceptions, hope they're caught

### 2. Add Context

✅ **Good**: Wrap errors with descriptive messages
❌ **Bad**: Propagate raw errors without context

### 3. Fail Fast

✅ **Good**: Validate early, return errors immediately
❌ **Bad**: Continue with invalid state

### 4. Recover Gracefully

✅ **Good**: Fall back to safe defaults
❌ **Bad**: Crash or leave system in inconsistent state

### 5. Log Appropriately

✅ **Good**: Log errors with enough context to debug
❌ **Bad**: Log too much (noise) or too little (not helpful)

---

## Testing Error Handling

### Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_tool_error_propagation() {
        let result = execute_tool(Tool::new(), invalid_input());
        assert!(result.is_err());
        match result.unwrap_err() {
            ToolError::InvalidInput(msg) => {
                assert!(msg.contains("Path required"));
            }
            _ => panic!("Wrong error type"),
        }
    }
    
    #[test]
    fn test_retry_logic() {
        let attempts = Arc::new(AtomicUsize::new(0));
        let result = execute_with_retry(|| {
            let attempts = attempts.clone();
            async move {
                let count = attempts.fetch_add(1, Ordering::SeqCst);
                if count < 2 {
                    Err(ToolError::Timeout(1000))
                } else {
                    Ok("success")
                }
            }
        }, 3);
        
        assert!(result.is_ok());
        assert_eq!(attempts.load(Ordering::SeqCst), 3);
    }
}
```

---

## Error Handling Checklist

- [ ] All error types defined
- [ ] Error propagation uses Result types
- [ ] Context added at boundaries
- [ ] Retry logic implemented
- [ ] Circuit breaker for external calls
- [ ] Graceful degradation in place
- [ ] Error logging configured
- [ ] Unit tests for error paths
- [ ] Integration tests for error recovery
- [ ] Monitoring and alerting set up

---

## Summary

**Key Principles**:
1. **Explicit over implicit** - Use Result types, not exceptions
2. **Context is king** - Add meaningful context to errors
3. **Fail gracefully** - Recover when possible, crash safely when not
4. **Log smartly** - Enough to debug, not too much noise
5. **Test error paths** - Errors are critical functionality

**Implementation Priority**:
1. Define error types
2. Implement propagation
3. Add retry logic
4. Add circuit breaker
5. Add monitoring
6. Test thoroughly

This error handling specification ensures robust, maintainable code across all language implementations.
