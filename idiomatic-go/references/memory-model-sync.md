# Memory Model & Sync Primitive Selection

## Table of Contents

- [Go Memory Model in Practice](#go-memory-model-in-practice)
- [Happens-Before Checklist](#happens-before-checklist)
- [Primitive Selection Matrix](#primitive-selection-matrix)
- [Publication Patterns](#publication-patterns)
- [Race Anti-Patterns](#race-anti-patterns)
- [Checklist](#checklist)

## Go Memory Model in Practice

The key rule: if there is no **happens-before** edge between write and read, the read is not guaranteed to observe that write.

Common happens-before edges in Go:

- `Mutex.Unlock` happens-before later `Mutex.Lock`
- `RWMutex.Unlock` happens-before later `Lock`/`RLock`
- Channel send happens-before corresponding receive
- Closing a channel happens-before receive that observes closed state
- `WaitGroup.Done` happens-before `Wait` unblocks
- `Once.Do(f)` completion happens-before later `Do` returns
- Atomic operations provide ordering guarantees when used consistently

## Happens-Before Checklist

Before sharing state across goroutines, check:

- [ ] Is there an explicit synchronization edge between producer and consumer?
- [ ] Are all accesses to shared mutable state protected by the same primitive?
- [ ] Are atomics used consistently (no mixed atomic + plain load/store on same field)?
- [ ] Is object publication done only after full initialization?
- [ ] Does shutdown path preserve ordering (close/signal then observe completion)?

## Primitive Selection Matrix

| Primitive | Pick When | Avoid When | Notes |
|----------|-----------|------------|-------|
| `sync.Mutex` | Single shared mutable state, moderate contention | Mostly read-only workloads | Default choice for correctness |
| `sync.RWMutex` | Read-heavy and read section is non-trivial | High write ratio or tiny read critical section | Writer preference may block readers |
| `sync/atomic` | Single-word counters/flags/pointers | Multi-field invariants | Great for fast paths, poor for complex state |
| Channel | Ownership transfer, pipelines, backpressure | Random shared-state mutation | Prefer channels for coordination/dataflow |
| `sync.Cond` | Repeated signaling on condition predicates | Simple one-shot notifications | Must use `for !cond { Wait() }` |
| `sync.WaitGroup` | Wait for task set completion | Cancellation/timeout control | Pair with `context` for stop semantics |
| `sync.Once` | One-time init publication | Need retries after failure | `OnceValue/OnceValues` for lazy values |

## Publication Patterns

### Immutable snapshot + atomic pointer

```go
type Config struct {
    Timeout time.Duration
    Retries int
}

var cfg atomic.Pointer[Config]

func PublishConfig(c Config) {
    // Publish fully initialized immutable snapshot.
    cc := c
    cfg.Store(&cc)
}

func CurrentConfig() Config {
    if p := cfg.Load(); p != nil {
        return *p
    }
    return Config{Timeout: 2 * time.Second, Retries: 1}
}
```

### Once-based safe initialization

```go
var (
    once sync.Once
    cli  *Client
    err0 error
)

func ClientInstance() (*Client, error) {
    once.Do(func() {
        cli, err0 = newClient()
    })
    return cli, err0
}
```

### Channel close as broadcast barrier

```go
// close(done) publishes shutdown signal to all goroutines.
func worker(done <-chan struct{}, in <-chan Job) {
    for {
        select {
        case <-done:
            return
        case j, ok := <-in:
            if !ok {
                return
            }
            _ = j
        }
    }
}
```

## Race Anti-Patterns

### 1) Mixed atomic and non-atomic access

```go
// ❌ Data race / undefined behavior.
if enabled { // plain read
    atomic.StoreInt32(&enabledFlag, 1)
}

// ✅ Use one strategy consistently.
if atomic.LoadInt32(&enabledFlag) == 1 {
    // ...
}
```

### 2) Double-checked locking without memory ordering

```go
// ❌ Not safe unless all accesses use proper synchronization.
if obj == nil {
    mu.Lock()
    if obj == nil {
        obj = &T{} // publication race risk if fields mutate later
    }
    mu.Unlock()
}
```

Prefer `sync.Once`, or immutable object + atomic pointer publication.

### 3) Copying sync primitives after first use

```go
// ❌ Copying struct containing Mutex/WaitGroup/Cond after use can panic or race.
type Cache struct {
    mu sync.Mutex
    m  map[string]string
}
```

Pass pointers, avoid value copies.

### 4) Assuming goroutine scheduling order

Do not infer ordering from timing/sleep. Use explicit synchronization (`channel`, locks, condition, `WaitGroup`).

## Checklist

- [ ] Shared mutable state has a single synchronization owner.
- [ ] Atomic use is narrow and documented (what invariant it protects).
- [ ] `sync` values are pointer-owned and never copied after use.
- [ ] Tests run with `go test -race` in CI.
- [ ] Lock contention and goroutine blocking are profiled before primitive rewrites.
