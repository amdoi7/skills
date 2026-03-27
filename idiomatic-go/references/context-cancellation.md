# Context, Cancellation & Concurrency

## Table of Contents

- [Context Fundamentals](#context-fundamentals)
- [Cancellation Patterns](#cancellation-patterns)
- [Concurrency Patterns](#concurrency-patterns)
- [Worker Pools](#worker-pools)
- [Singleflight & Cache Stampede](#singleflight--cache-stampede)
- [Graceful Shutdown](#graceful-shutdown)
- [Common Pitfalls](#common-pitfalls)

## Context Fundamentals

### When to Use What

| Context Type | Use Case |
|--------------|----------|
| `context.Background()` | Top-level entry points (main, init) |
| `context.TODO()` | Placeholder when unsure (refactor later) |
| `context.WithCancel()` | Manual cancellation control |
| `context.WithTimeout()` | Deadline-based cancellation |
| `context.WithDeadline()` | Specific time-based cancellation |
| `context.WithValue()` | Request-scoped data (sparingly) |

### Context Rules

```go
// ✅ Context as first parameter, named ctx
func ProcessData(ctx context.Context, data []byte) error

// ❌ Never store context in structs
type BadService struct {
    ctx context.Context  // WRONG
}

// ✅ Pass context to methods
type GoodService struct{}

func (s *GoodService) Process(ctx context.Context, data []byte) error
```

## Cancellation Patterns

### Basic Cancellation

```go
func worker(ctx context.Context, jobs <-chan Job) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()  // context.Canceled or context.DeadlineExceeded
        case job, ok := <-jobs:
            if !ok {
                return nil  // Channel closed
            }
            if err := processJob(ctx, job); err != nil {
                return err
            }
        }
    }
}
```

### Context-Aware Channel Send (Leak Prevention)

```go
// ❌ May block forever if receiver is gone
func badSend(ch chan<- int, val int) {
    ch <- val  // Potential goroutine leak
}

// ✅ Respects cancellation, no leak
func safeSend(ctx context.Context, ch chan<- int, val int) error {
    select {
    case ch <- val:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

// ✅ With timeout
func sendWithTimeout(ch chan<- int, val int, timeout time.Duration) error {
    ctx, cancel := context.WithTimeout(context.Background(), timeout)
    defer cancel()
    return safeSend(ctx, ch, val)
}
```

### Timeout Management

```go
func fetchWithTimeout(url string) ([]byte, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()  // ALWAYS defer cancel

    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            return nil, fmt.Errorf("request timed out: %w", err)
        }
        return nil, err
    }
    defer resp.Body.Close()

    return io.ReadAll(resp.Body)
}
```

## Concurrency Patterns

### errgroup for Fan-Out

```go
import "golang.org/x/sync/errgroup"

func fetchAll(ctx context.Context, urls []string) ([][]byte, error) {
    g, ctx := errgroup.WithContext(ctx)
    results := make([][]byte, len(urls))

    for i, url := range urls {
        i, url := i, url  // Capture (Go < 1.22)
        g.Go(func() error {
            data, err := fetchWithContext(ctx, url)
            if err != nil {
                return err
            }
            results[i] = data
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err  // First error cancels all
    }
    return results, nil
}
```

### errgroup with Backpressure

```go
func processItems(ctx context.Context, items []Item) error {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(10)  // Max 10 concurrent goroutines

    for _, item := range items {
        item := item
        g.Go(func() error {
            return processItem(ctx, item)
        })
    }

    return g.Wait()
}
```

### Semaphore Pattern

```go
func processWithLimit(ctx context.Context, items []Item, limit int) error {
    sem := make(chan struct{}, limit)

    g, ctx := errgroup.WithContext(ctx)
    for _, item := range items {
        item := item
        g.Go(func() error {
            select {
            case sem <- struct{}{}:
                defer func() { <-sem }()
                return processItem(ctx, item)
            case <-ctx.Done():
                return ctx.Err()
            }
        })
    }

    return g.Wait()
}
```

## Worker Pools

### Basic Worker Pool

```go
func workerPool(ctx context.Context, jobs <-chan Job, results chan<- Result, workers int) {
    var wg sync.WaitGroup

    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for {
                select {
                case <-ctx.Done():
                    return
                case job, ok := <-jobs:
                    if !ok {
                        return
                    }
                    result := process(job)
                    select {
                    case results <- result:
                    case <-ctx.Done():
                        return
                    }
                }
            }
        }()
    }

    wg.Wait()
    close(results)
}
```

### Worker Pool with errors.Join (Go 1.20+)

Be explicit about cancellation semantics. Two common models:

```go
// ✅ Drain model: keep consuming queued jobs even if ctx is canceled.
// Use when work should be best-effort completed once accepted.
func workerPoolDrain(ctx context.Context, items []Item, workers int) error {
    jobs := make(chan Item, len(items))
    errCh := make(chan error, len(items))

    // Feed jobs
    for _, item := range items {
        jobs <- item
    }
    close(jobs)

    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs { // drains until jobs closed
                if err := processItem(ctx, job); err != nil {
                    errCh <- err
                }
            }
        }()
    }

    wg.Wait()
    close(errCh)

    var errs []error
    for err := range errCh {
        errs = append(errs, err)
    }
    return errors.Join(errs...)
}
```

```go
// ✅ Cancel-fast model: stop workers quickly on ctx cancellation.
// Use for latency-sensitive workloads or when stale work should be dropped.
func workerPoolCancelFast(ctx context.Context, items []Item, workers int) error {
    jobs := make(chan Item, len(items))
    errCh := make(chan error, len(items))

    for _, item := range items {
        jobs <- item
    }
    close(jobs)

    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for {
                select {
                case <-ctx.Done():
                    return
                case job, ok := <-jobs:
                    if !ok {
                        return
                    }
                    if err := processItem(ctx, job); err != nil {
                        errCh <- err
                    }
                }
            }
        }()
    }

    wg.Wait()
    close(errCh)

    var errs []error
    for err := range errCh {
        errs = append(errs, err)
    }

    // Preserve cancellation signal alongside worker errors.
    if ctxErr := ctx.Err(); ctxErr != nil {
        errs = append(errs, ctxErr)
    }

    return errors.Join(errs...)
}
```

## Singleflight & Cache Stampede

### Basic Singleflight

```go
import "golang.org/x/sync/singleflight"

var group singleflight.Group

func getData(key string) (interface{}, error) {
    v, err, shared := group.Do(key, func() (interface{}, error) {
        // Only ONE call executes, others wait
        return expensiveFetch(key)
    })
    if shared {
        log.Printf("Result was shared with other callers")
    }
    return v, err
}
```

### Singleflight with TTL Cache

```go
type TTLCache struct {
    group singleflight.Group
    cache sync.Map
    ttl   time.Duration
}

type cacheEntry struct {
    value     interface{}
    expiresAt time.Time
}

func (c *TTLCache) Get(key string, fetch func() (interface{}, error)) (interface{}, error) {
    // Check cache first
    if entry, ok := c.cache.Load(key); ok {
        e := entry.(*cacheEntry)
        if time.Now().Before(e.expiresAt) {
            return e.value, nil
        }
    }

    // Singleflight fetch
    v, err, _ := c.group.Do(key, func() (interface{}, error) {
        // Double-check cache (another goroutine may have populated it)
        if entry, ok := c.cache.Load(key); ok {
            e := entry.(*cacheEntry)
            if time.Now().Before(e.expiresAt) {
                return e.value, nil
            }
        }

        value, err := fetch()
        if err != nil {
            return nil, err
        }

        c.cache.Store(key, &cacheEntry{
            value:     value,
            expiresAt: time.Now().Add(c.ttl),
        })
        return value, nil
    })

    return v, err
}
```

## Graceful Shutdown

### HTTP Server Graceful Shutdown

```go
func main() {
    srv := &http.Server{
        Addr:    ":8080",
        Handler: handler,
    }

    // Start server
    go func() {
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatalf("Server error: %v", err)
        }
    }()

    // Wait for interrupt
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    log.Println("Shutting down server...")

    // Graceful shutdown with timeout
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatalf("Server forced to shutdown: %v", err)
    }

    log.Println("Server exited")
}
```

### Leak-Free Scheduler

```go
type Scheduler struct {
    tasks  chan func()
    done   chan struct{}
    wg     sync.WaitGroup
}

func NewScheduler(workers int) *Scheduler {
    s := &Scheduler{
        tasks: make(chan func(), 100),
        done:  make(chan struct{}),
    }

    for i := 0; i < workers; i++ {
        s.wg.Add(1)
        go s.worker()
    }

    return s
}

func (s *Scheduler) worker() {
    defer s.wg.Done()
    for {
        select {
        case <-s.done:
            return
        case task, ok := <-s.tasks:
            if !ok {
                return
            }
            task()
        }
    }
}

func (s *Scheduler) Schedule(task func()) {
    select {
    case s.tasks <- task:
    case <-s.done:
    }
}

func (s *Scheduler) Shutdown() {
    close(s.done)
    close(s.tasks)
    s.wg.Wait()
}
```

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Forgetting `defer cancel()` | Context leak | Always `defer cancel()` after `WithCancel/WithTimeout` |
| Blocking channel send | Goroutine leak | Use `select` with `ctx.Done()` |
| Not checking `ctx.Done()` in loops | Won't respond to cancellation | Check at start of each iteration |
| Using `context.Background()` in handlers | Can't cancel | Use request context `r.Context()` |
| Storing context in struct | Wrong lifetime | Pass as parameter |
| Ignoring `ctx.Err()` | Lost error info | Return `ctx.Err()` or wrap it |

### Leak Detection

```go
// In tests, use goleak
import "go.uber.org/goleak"

func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}
```
