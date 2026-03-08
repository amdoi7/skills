# Error Handling: Semantics, Wrapping & Edge Cases

## Table of Contents

- [Error Fundamentals](#error-fundamentals)
- [Error Wrapping](#error-wrapping)
- [Secure Error Handling](#secure-error-handling)
- [Error Inspection](#error-inspection)
- [Sentinel Errors](#sentinel-errors)
- [Custom Error Types](#custom-error-types)
- [Retry Patterns](#retry-patterns)
- [Defer & Cleanup](#defer--cleanup)
- [The nil != nil Gotcha](#the-nil--nil-gotcha)

## Error Fundamentals

### Core Principles

| Principle         | Description                                        |
| ----------------- | -------------------------------------------------- |
| Errors are values | Treat errors as first-class values, not exceptions |
| Handle or return  | Either handle an error or return it to caller      |
| Wrap with context | Add context when propagating errors up the stack   |
| Fail fast         | Return early on error, avoid deep nesting          |

### Basic Error Handling

```go
// ✅ Check errors immediately
result, err := doSomething()
if err != nil {
    return fmt.Errorf("do something: %w", err)
}
// Use result...

// ❌ Don't ignore errors
result, _ := doSomething()  // Bad: error ignored
```

## Error Wrapping

### fmt.Errorf with %w

```go
func fetchUser(ctx context.Context, id string) (*User, error) {
    data, err := db.Query(ctx, "SELECT * FROM users WHERE id = ?", id)
    if err != nil {
        // Wrap with context, preserve original error
        return nil, fmt.Errorf("fetch user %s: %w", id, err)
    }
    return parseUser(data)
}

// Creates error chain:
// "fetch user 123: query users: connection refused"
```

### When to Wrap vs When Not

| Situation | Action |
|-----------|--------|
| Adding context | `fmt.Errorf("context: %w", err)` |
| Crossing package boundary | Wrap with domain error |
| Internal helper | May return unwrapped |
| Hiding implementation | `fmt.Errorf("operation failed: %v", err)` (%v not %w) |
| Returning to public clients | Translate to fixed safe message/code (never raw `err.Error()`) |

```go
// ✅ Wrap at boundaries
func (s *UserService) GetUser(ctx context.Context, id string) (*User, error) {
    user, err := s.repo.Find(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("get user: %w", err) // Service boundary
    }
    return user, nil
}

// ✅ Hide implementation details from upstream callers (use %v)
func (c *Client) Fetch(url string) error {
    resp, err := c.http.Get(url)
    if err != nil {
        // Don't expose http.Client internals in this layer
        return fmt.Errorf("fetch failed: %v", err)
    }
    ...
}

// ✅ Public boundary: translate to safe response (do not return err.Error())
func writePublicError(w http.ResponseWriter, err error) {
    http.Error(w, "service temporarily unavailable", http.StatusServiceUnavailable)
    // Internal details go to logs only.
}
```

## Secure Error Handling

### SafeError: Two Channels (Public vs Internal)

Use a custom error that separates user-facing output from internal diagnostics.

```go
type SafeError struct {
    Code     string            // machine-readable code, e.g. "AUTH_FAILED"
    UserMsg  string            // safe message for clients
    Internal error             // original cause for logs/debugging
    Metadata map[string]string // sanitized context only
}

func (e *SafeError) Error() string {
    return e.UserMsg // safe by default
}
```

### Contain Errors at Trust Boundaries

| Boundary | Rule | Example |
|----------|------|---------|
| Subsystem boundary | Wrap/translate to domain error | DB duplicate key → `ErrDuplicateUser` |
| Service boundary | Translate to protocol-safe error | internal error → gRPC `Unavailable` |
| Public boundary | Return fixed safe message/code | never expose raw `err.Error()` |

```go
func translateAndRespond(w http.ResponseWriter, err error) {
    switch {
    case errors.Is(err, domain.ErrInvalidInput):
        http.Error(w, "invalid request", http.StatusBadRequest)
    default:
        http.Error(w, "internal error", http.StatusInternalServerError)
    }
}
```

### Sanitized Logging (`slog` + redactor)

Never log raw auth payloads, tokens, or sensitive headers.

```go
type Redactor interface{ Redact() any }

type LoginRequest struct {
    Username string
    Password string
}

func (r LoginRequest) Redact() any {
    return struct {
        Username string `json:"username"`
        Password string `json:"password"`
    }{Username: r.Username, Password: "***REDACTED***"}
}

func logRequest(logger *slog.Logger, r *http.Request, req LoginRequest) {
    safeHeaders := r.Header.Clone()
    safeHeaders.Del("Authorization")
    safeHeaders.Del("Cookie")

    logger.Info("login attempt",
        slog.Any("req", req.Redact()),
        slog.Any("headers", safeHeaders),
    )
}
```

## Error Inspection

### errors.Is (Value Comparison)

```go
import "errors"

// Check if error IS a specific error
if errors.Is(err, context.DeadlineExceeded) {
    // Handle timeout
}

if errors.Is(err, io.EOF) {
    // End of input
}

if errors.Is(err, sql.ErrNoRows) {
    // Not found
}
```

### errors.As (Type Assertion)

```go
// Extract error of specific type from chain
var pathErr *os.PathError
if errors.As(err, &pathErr) {
    fmt.Printf("Operation %s on %s failed: %v\n",
        pathErr.Op, pathErr.Path, pathErr.Err)
}

var netErr net.Error
if errors.As(err, &netErr) && netErr.Timeout() {
    // Handle network timeout
}
```

### errors.Join (Go 1.20+)

```go
var (
    ErrNameRequired        = errors.New("name required")
    ErrEmailRequired       = errors.New("email required")
    ErrInvalidEmailFormat  = errors.New("invalid email format")
)

// Combine multiple errors
func validateUser(u *User) error {
    var errs []error

    if u.Name == "" {
        errs = append(errs, ErrNameRequired)
    }
    if u.Email == "" {
        errs = append(errs, ErrEmailRequired)
    }
    if u.Email != "" && !isValidEmail(u.Email) {
        errs = append(errs, ErrInvalidEmailFormat)
    }

    return errors.Join(errs...) // Returns nil if no errors
}

// All joined sentinel errors are checkable
err := validateUser(user)
if errors.Is(err, ErrNameRequired) {
    // ...
}
if errors.Is(err, ErrInvalidEmailFormat) {
    // ...
}
```

## Sentinel Errors

### Defining Sentinel Errors

```go
package mypackage

import "errors"

// Exported sentinel errors
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrInvalidInput = errors.New("invalid input")
)

// Usage
func (r *Repo) Find(id string) (*Item, error) {
    item := r.items[id]
    if item == nil {
        return nil, ErrNotFound
    }
    return item, nil
}

// Caller
item, err := repo.Find(id)
if errors.Is(err, mypackage.ErrNotFound) {
    // Handle not found case
}
```

### Sentinel Error Best Practices

| Do | Don't |
|-----|-------|
| Use for stable, documented errors | Use for every error |
| Make them package-level vars | Create in functions |
| Document when they're returned | Change their meaning |
| Keep them simple (no state) | Store context in them |

## Custom Error Types

### Basic Custom Error

```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error: %s: %s", e.Field, e.Message)
}

// Usage
func validateAge(age int) error {
    if age < 0 {
        return &ValidationError{Field: "age", Message: "must be non-negative"}
    }
    return nil
}
```

### Custom Error with Is/As Support

```go
type NotFoundError struct {
    Resource string
    ID       string
}

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s %s not found", e.Resource, e.ID)
}

// Support errors.Is
func (e *NotFoundError) Is(target error) bool {
    return target == ErrNotFound
}

// Now both work:
// errors.Is(err, ErrNotFound) // true
// var nf *NotFoundError
// errors.As(err, &nf)         // true
```

### Wrapped Custom Error

```go
type OperationError struct {
    Op  string
    Err error  // Wrapped error
}

func (e *OperationError) Error() string {
    return fmt.Sprintf("%s: %v", e.Op, e.Err)
}

func (e *OperationError) Unwrap() error {
    return e.Err
}

// Usage
func process(data []byte) error {
    if err := validate(data); err != nil {
        return &OperationError{Op: "validate", Err: err}
    }
    ...
}

// errors.Is traverses the chain via Unwrap
```

## Retry Patterns

### Exponential Backoff with Context

```go
type RetryConfig struct {
    MaxAttempts int
    BaseDelay   time.Duration
    MaxDelay    time.Duration
}

func WithRetry[T any](ctx context.Context, cfg RetryConfig, fn func() (T, error)) (T, error) {
    var zero T
    var lastErr error

    for attempt := 0; attempt < cfg.MaxAttempts; attempt++ {
        if attempt > 0 {
            delay := cfg.BaseDelay * time.Duration(1<<uint(attempt-1))
            if delay > cfg.MaxDelay {
                delay = cfg.MaxDelay
            }

            select {
            case <-time.After(delay):
            case <-ctx.Done():
                return zero, ctx.Err()
            }
        }

        result, err := fn()
        if err == nil {
            return result, nil
        }

        // Check if error is retryable
        if !isRetryable(err) {
            return zero, err
        }

        lastErr = err
    }

    return zero, fmt.Errorf("max retries exceeded: %w", lastErr)
}

func isRetryable(err error) bool {
    // Network errors, timeouts are retryable
    var netErr net.Error
    if errors.As(err, &netErr) && netErr.Temporary() {
        return true
    }
    if errors.Is(err, context.DeadlineExceeded) {
        return true
    }
    return false
}
```

## Defer & Cleanup

### Defer Execution Order (LIFO)

```go
func process() error {
    fmt.Println("1")
    defer fmt.Println("4")  // Last in, first out
    fmt.Println("2")
    defer fmt.Println("3")
    return nil
}
// Output: 1, 2, 3, 4
```

### Cleanup Chain with Error Preservation

```go
func processFile(path string) (err error) {
    f, err := os.Open(path)
    if err != nil {
        return fmt.Errorf("open: %w", err)
    }
    defer func() {
        if cerr := f.Close(); cerr != nil && err == nil {
            err = fmt.Errorf("close: %w", cerr)
        }
    }()

    // Process file...
    return nil
}

// More complex cleanup chain
func transaction(ctx context.Context) (err error) {
    tx, err := db.BeginTx(ctx)
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }

    defer func() {
        if err != nil {
            if rbErr := tx.Rollback(); rbErr != nil {
                err = errors.Join(err, fmt.Errorf("rollback: %w", rbErr))
            }
        }
    }()

    // Do work...

    if err := tx.Commit(); err != nil {
        return fmt.Errorf("commit: %w", err)
    }

    return nil
}
```

### Named Return for Defer

```go
// ✅ Named return allows defer to modify return value
func readConfig(path string) (config Config, err error) {
    f, err := os.Open(path)
    if err != nil {
        return config, err
    }
    defer func() {
        if cerr := f.Close(); err == nil {
            err = cerr  // Can modify err
        }
    }()

    err = json.NewDecoder(f).Decode(&config)
    return config, err
}
```

## The nil != nil Gotcha

### The Problem

```go
type MyError struct {
    msg string
}

func (e *MyError) Error() string { return e.msg }

// ❌ WRONG: Returns typed nil
func getError() error {
    var err *MyError = nil
    return err  // BUG: interface{type: *MyError, value: nil}
}

func main() {
    err := getError()
    if err != nil {
        fmt.Println("Error is NOT nil!")  // This prints!
    }
}
```

### Why It Happens

```go
// Interface = (type, value)
// nil interface = (nil, nil)
// Typed nil    = (*MyError, nil) != (nil, nil)

var err error = nil           // (nil, nil) - truly nil
var myErr *MyError = nil
err = myErr                   // (*MyError, nil) - NOT nil!
```

### The Solution

```go
// ✅ Return explicit nil
func getError() error {
    var err *MyError = nil
    if err != nil {
        return err
    }
    return nil  // Explicit untyped nil
}

// ✅ Or check before return
func maybeError(condition bool) error {
    if condition {
        return &MyError{msg: "something went wrong"}
    }
    return nil  // Always return explicit nil
}

// ✅ Use type switch for safety
func handleError(err error) {
    if err == nil {
        return
    }

    switch e := err.(type) {
    case *MyError:
        if e == nil {
            return  // Handle typed nil case
        }
        // Handle actual error
    default:
        // Handle other errors
    }
}
```

### Prevention Checklist

| Check | Description |
|-------|-------------|
| Never return pointer error vars directly | Always check `!= nil` first |
| Use explicit `return nil` | Don't rely on zero-value interface |
| Prefer `errors.New` / `fmt.Errorf` | Returns concrete, non-nil errors |
| Test for typed nil in critical code | Use reflection if needed |
