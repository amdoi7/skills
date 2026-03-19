---
name: idiomatic-go
description: Idiomatic Go guidance for modeling domain types clearly, keeping package boundaries simple, and writing safe concurrent code. Use when writing or refactoring Go, reviewing design, shaping APIs or package boundaries, working with goroutines and context, or when code is becoming stringly typed, over-abstracted, or concurrency-heavy.
---

# Idiomatic Go

Use this skill to keep Go code obvious, boundary-aware, and safe under concurrency.

## Decision Order

1. Model the domain and boundaries clearly
2. Preserve correctness and concurrency safety
3. Keep the common path simple for callers
4. Optimize after measurement
5. Add abstractions only when they hide real complexity

## Core Principles

- **Domain language first**: package names, type names, and method names should reveal the business concept before the transport or storage mechanism.
- **Parse at boundaries**: convert raw input into strong types and trusted values as early as possible.
- **Concrete inside, interfaces at boundaries**: add interfaces where packages or systems meet, not for every type.
- **Recoverable failures are errors**: use `error` for domain and infrastructure failures; reserve `panic` for programmer bugs or impossible states after invariants are guaranteed.
- **Context flows down**: pass `context.Context` as the first parameter on request-scoped work.
- **Deep modules**: keep common call paths simple and hide the hard parts behind small APIs.
- **Eliminate special cases**: redesign the data shape or API before adding more branches.

## Workflow

1. Determine the primary task:
   - **Modeling, package split, API design, or review**: read [references/philosophy.md](references/philosophy.md).
   - **Errors, retries, or public error translation**: read [references/error-handling.md](references/error-handling.md).
   - **Goroutines, deadlines, or cancellation**: read [references/context-cancellation.md](references/context-cancellation.md).
   - **Shared state or primitive choice**: read [references/memory-model-sync.md](references/memory-model-sync.md).
   - **DB pooling and deadlines**: read [references/database-sql-pooling-timeouts.md](references/database-sql-pooling-timeouts.md).
   - **Runtime incident triage**: read [references/runtime-observability.md](references/runtime-observability.md).
   - **Performance work**: read [references/performance.md](references/performance.md).
   - **HTTP servers and middleware**: read [references/http-middleware.md](references/http-middleware.md).
   - **Testing strategy**: read [references/testing.md](references/testing.md).
2. Load only the reference files needed for the task.
3. Keep the common path simple; reach for advanced mechanisms only when evidence demands them.

## Quick Reference

| Topic | Use When | Reference |
| --- | --- | --- |
| Design Philosophy | Code review, refactoring, module boundaries | [references/philosophy.md](references/philosophy.md) |
| Context & Cancellation | Goroutines, HTTP handlers, timeouts | [references/context-cancellation.md](references/context-cancellation.md) |
| Error Handling | Boundary translation, retries, error semantics | [references/error-handling.md](references/error-handling.md) |
| Memory Model & Sync | Happens-before checks, primitive selection | [references/memory-model-sync.md](references/memory-model-sync.md) |
| `database/sql` Pooling | Pool sizing, queueing, timeout propagation | [references/database-sql-pooling-timeouts.md](references/database-sql-pooling-timeouts.md) |
| Runtime Observability | Triage, pprof correlation, GC or scheduler diagnostics | [references/runtime-observability.md](references/runtime-observability.md) |
| Performance | Allocation or throughput work | [references/performance.md](references/performance.md) |
| HTTP & Middleware | API servers, clients, request context | [references/http-middleware.md](references/http-middleware.md) |
| File Systems | Config loading, embeds, path handling | [references/filesystems.md](references/filesystems.md) |
| Testing | Table-driven tests, fakes, fuzzing | [references/testing.md](references/testing.md) |

## Essential Patterns

### Parse at the Boundary

```go
type OrderID string

func ParseOrderID(raw string) (OrderID, error) {
    if raw == "" {
        return "", errors.New("empty order id")
    }
    return OrderID(raw), nil
}
```

### Context-Aware Concurrency

```go
func send(ctx context.Context, ch chan<- int, val int) error {
    select {
    case ch <- val:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

### Wrap Errors at Trust Boundaries

```go
func (s *Service) ProcessOrder(ctx context.Context, id string) error {
    orderID, err := ParseOrderID(id)
    if err != nil {
        return fmt.Errorf("parse order id: %w", err)
    }

    if err := s.repo.Process(ctx, orderID); err != nil {
        return fmt.Errorf("process order %s: %w", orderID, err)
    }
    return nil
}
```

## Common Pitfalls

| Pitfall | Problem | Better Direction |
| --- | --- | --- |
| `string` and `float64` everywhere | Domain meaning leaks into comments and call sites | Introduce small strong types |
| Global `handler/service/repo` package trees | Technical structure hides domain boundaries | Split by capability or bounded context first |
| Interfaces for every type | Shallow abstractions and indirection | Add interfaces only at package or system boundaries |
| `panic` for expected failures | Callers cannot recover and semantics become unclear | Return typed or wrapped errors |
| Advanced sync primitives by default | Harder reasoning and weaker invariants | Start with ownership and `Mutex`; optimize later |

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
