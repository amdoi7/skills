---
name: idiomatic-go
description: Idiomatic Go development guidance for writing safe, efficient, and concurrent code. Use when writing new Go code, refactoring, reviewing, designing APIs, working with concurrency/context, or setting up Go projects. Triggers on .go files, go.mod, Go-related questions, HTTP handlers, goroutines, channels, error handling discussions.
---

# Idiomatic Go

Concise guidance for writing idiomatic Go code that is safe, efficient, and leverages Go's concurrency model correctly.

> "I fear not the man who has practiced 10,000 kicks once, but I fear the man who has practiced one kick 10,000 times." — Bruce Lee

## Decision Order

1. Correctness and safety (no goroutine leaks, no data races)
2. Clarity and simplicity (readable > clever)
3. Performance (profile first, optimize with intent)
4. Extensibility (interfaces at boundaries)

## Core Principles

- **Simplicity**: Go's strength is simplicity. Resist abstraction until it's earned
- **Explicit > Implicit**: No hidden control flow, no magic
- **Accept interfaces, return structs**: Flexibility in, concrete out
- **Handle errors at boundaries**: Don't propagate raw errors across packages
- **Context flows down**: Pass `context.Context` as first parameter
- **Deep modules**: 简单接口 + 强大功能，复杂性下沉
- **Eliminate special cases**: 通过设计消除边界情况，而非条件判断
- **Parse, don't validate**: 边界层解析成强类型，内部代码信任上游数据

> "Bad programmers worry about the code. Good programmers worry about data structures." — Linus Torvalds
>
> "Make illegal states unrepresentable." — Yaron Minsky

## Quick Reference

| Topic                  | When to Use                                                 | Reference                                                                                  |
| ---------------------- | ----------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **Design Philosophy**  | Code review, refactoring, design decisions                  | [references/philosophy.md](references/philosophy.md)                                       |
| Context & Cancellation | Goroutines, HTTP handlers, timeouts                         | [references/context-cancellation.md](references/context-cancellation.md)                   |
| Runtime Observability  | Runtime triage, pprof correlation, GC/scheduler diagnostics | [references/runtime-observability.md](references/runtime-observability.md)                 |
| Memory Model & Sync    | Happens-before checks, primitive selection                  | [references/memory-model-sync.md](references/memory-model-sync.md)                         |
| database/sql Pooling   | Pool sizing, queueing diagnosis, timeout propagation        | [references/database-sql-pooling-timeouts.md](references/database-sql-pooling-timeouts.md) |
| Performance            | High-throughput paths, memory-sensitive                     | [references/performance.md](references/performance.md)                                     |
| HTTP & Middleware      | API servers, clients                                        | [references/http-middleware.md](references/http-middleware.md)                             |
| Error Handling         | All code paths                                              | [references/error-handling.md](references/error-handling.md)                               |
| File Systems           | Config loading, embeds                                      | [references/filesystems.md](references/filesystems.md)                                     |
| Testing                | Quality gates                                               | [references/testing.md](references/testing.md)                                             |

## Essential Patterns

### Context & Cancellation

```go
// Always respect context cancellation
func fetchData(ctx context.Context, url string) ([]byte, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, fmt.Errorf("create request: %w", err)
    }

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("execute request: %w", err)
    }
    defer resp.Body.Close()

    return io.ReadAll(resp.Body)
}

// Context-aware channel send (no goroutine leaks)
func send(ctx context.Context, ch chan<- int, val int) error {
    select {
    case ch <- val:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

### Concurrency Patterns

```go
// Worker pool with backpressure and error collection
func processItems(ctx context.Context, items []Item) error {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(10) // Backpressure: max 10 concurrent

    for _, item := range items {
        item := item // Capture for closure (Go < 1.22)
        g.Go(func() error {
            return processItem(ctx, item)
        })
    }

    return g.Wait() // Returns first error, cancels others
}

// Singleflight for cache stampede prevention
var group singleflight.Group

func getData(key string) (interface{}, error) {
    v, err, _ := group.Do(key, func() (interface{}, error) {
        return expensiveFetch(key)
    })
    return v, err
}
```

### Runtime Observability

```go
// Read stable runtime metrics and convert cumulative values to rates.
func sampleRuntime() {
    samples := []metrics.Sample{
        {Name: "/gc/heap/live:bytes"},
        {Name: "/gc/pauses:seconds"},
        {Name: "/sched/latencies:seconds"},
        {Name: "/sync/mutex/wait/total:seconds"},
    }
    metrics.Read(samples)

    // Correlate metric anomalies with pprof snapshots.
    // e.g. mutex wait up -> capture mutex/block profile
}
```

**Rules**:
- Prefer **rate/percentile** over raw cumulative numbers.
- Use metrics for detection, `pprof` for root-cause confirmation.
- Keep an incident map: metric symptom -> profile to capture.

### Memory Model & Sync Selection

```go
// Safe publication via atomic pointer to immutable snapshot.
type Config struct{ Timeout time.Duration }

var current atomic.Pointer[Config]

func Publish(c Config) {
    cc := c
    current.Store(&cc)
}

func LoadConfig() Config {
    if p := current.Load(); p != nil {
        return *p
    }
    return Config{Timeout: time.Second}
}
```

**Rules**:
- Every shared write/read pair must have a clear happens-before edge.
- Do not mix atomic and non-atomic access on the same field.
- Start with `Mutex`; upgrade to `RWMutex/atomic` only after contention evidence.

### database/sql Pooling & Context Timeouts

```go
func queryUser(ctx context.Context, db *sql.DB, id int64) (User, error) {
    qctx, cancel := context.WithTimeout(ctx, 200*time.Millisecond)
    defer cancel()

    var u User
    err := db.QueryRowContext(qctx,
        `SELECT id, email FROM users WHERE id = $1`, id,
    ).Scan(&u.ID, &u.Email)
    return u, err
}
```

**Rules**:
- Always set pool limits (`MaxOpen`, `MaxIdle`, lifetime/idle time).
- Every DB operation must use `Context` with deadline.
- Watch `DB.Stats().WaitCount/WaitDuration` for pool saturation.

### Error Handling

```go
// Wrap errors with context at boundaries
func (s *Service) ProcessOrder(ctx context.Context, id string) error {
    order, err := s.repo.Get(ctx, id)
    if err != nil {
        return fmt.Errorf("get order %s: %w", id, err)
    }

    if err := s.validate(order); err != nil {
        return fmt.Errorf("validate order: %w", err)
    }

    return nil
}

// Check error types with errors.Is/As
if errors.Is(err, context.DeadlineExceeded) {
    // Handle timeout
}

var pathErr *os.PathError
if errors.As(err, &pathErr) {
    // Access pathErr.Path, pathErr.Op
}

// ⚠️ Typed nil gotcha - NEVER return typed nil
func getError() error {
    var err *MyError = nil
    return err // BUG: err != nil because interface has type
}
// ✅ Correct: return nil explicitly
func getError() error {
    var err *MyError = nil
    if err != nil {
        return err
    }
    return nil
}
```

### HTTP & Middleware

```go
// Interface-based middleware
type Middleware func(http.Handler) http.Handler

func Chain(h http.Handler, mws ...Middleware) http.Handler {
    for i := len(mws) - 1; i >= 0; i-- {
        h = mws[i](h)
    }
    return h
}

// HTTP client hygiene
func newClient() *http.Client {
    return &http.Client{
        Timeout: 30 * time.Second,
        Transport: &http.Transport{
            MaxIdleConns:        100,
            MaxIdleConnsPerHost: 10,
            IdleConnTimeout:     90 * time.Second,
        },
    }
}
```

### Performance Patterns

```go
// sync.Pool for buffer reuse
var bufPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func process(data []byte) {
    buf := bufPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufPool.Put(buf)
    }()
    // Use buf...
}

// Sharded locks for concurrent map
type ShardedMap struct {
    shards [256]struct {
        sync.RWMutex
        m map[string]interface{}
    }
}

func (sm *ShardedMap) getShard(key string) *struct {
    sync.RWMutex
    m map[string]interface{}
} {
    h := fnv.New32a()
    h.Write([]byte(key))
    return &sm.shards[h.Sum32()%256]
}
```

### Testing

```go
// Table-driven tests with parallel execution
func TestProcess(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    string
        wantErr bool
    }{
        {"empty", "", "", true},
        {"valid", "hello", "HELLO", false},
    }

    for _, tt := range tests {
        tt := tt // Capture
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            got, err := Process(tt.input)
            if (err != nil) != tt.wantErr {
                t.Errorf("error = %v, wantErr %v", err, tt.wantErr)
            }
            if got != tt.want {
                t.Errorf("got %q, want %q", got, tt.want)
            }
        })
    }
}

// Fuzzing
func FuzzParse(f *testing.F) {
    f.Add("valid input")
    f.Fuzz(func(t *testing.T, input string) {
        _, _ = Parse(input) // Should not panic
    })
}
```

## Common Pitfalls

### Concurrency & Safety

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Goroutine leak | Blocked send/receive forever | Always use `select` with `ctx.Done()` |
| Data race | Concurrent map access | Use `sync.Map` or mutex-protected map |
| Typed nil | `var err *MyError = nil; return err` | Return explicit `nil` for interface types |
| Closure capture | Loop variable captured by reference | `v := v` before closure (Go < 1.22) |
| Unbuffered channel | Deadlock on single goroutine | Size buffer or use separate sender/receiver |
| Missing `defer` | Resource leak | `defer closer()` immediately after acquire |
| Error shadowing | `:=` creates new variable in inner scope | Use `=` or declare explicitly |
| Metrics misread | Reading cumulative metric as instant signal | Convert counters to rate; use histogram percentiles |
| Sync primitive mixing | Mixing `atomic` and mutex/plain access on same field | Use one synchronization strategy per field |
| Missing query deadline | DB calls can hang and pin pool slots | Always use `QueryContext/ExecContext` with timeout |
| Unbounded DB pool | Traffic bursts overload DB or queue indefinitely | Set `MaxOpenConns` and observe `WaitCount/WaitDuration` |

### Design & Structure (See [philosophy.md](references/philosophy.md))

| Pitfall | Threshold | Solution |
|---------|-----------|----------|
| Deep nesting | > 3 levels | Early return, extract functions |
| Long functions | > 100 lines | Split into smaller focused functions |
| Too many locals | > 10 variables | Extract struct or split function |
| Shallow modules | Interface ≈ Implementation | Pull complexity down, simplify API |
| Defensive defaults | `?? 0` masks bugs | Fail fast, expose upstream errors |
| Special case handling | Many `if` branches | Redesign to eliminate special cases |
| Repeated validation | `if x == nil` everywhere | Parse at boundary, trust upstream |
| Primitive obsession | `string` for ID, `float64` for money | NewType: `type UserID string` |
| Validate then discard | Check valid but keep raw type | Parse into strong type, carry proof |

## Tooling

```bash
# Format
gofmt -w .
goimports -w .

# Lint
golangci-lint run

# Race detection
go test -race ./...

# Benchmark
go test -bench=. -benchmem ./...
```
