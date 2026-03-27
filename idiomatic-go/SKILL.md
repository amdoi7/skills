---
name: idiomatic-go
description: >
  Idiomatic Go guidance for modeling domain types clearly, keeping package
  boundaries simple, and writing safe concurrent code. Use when writing,
  fixing, or reviewing Go code — tightening types, error semantics, context
  propagation, concurrency ownership, or tests. Do NOT use for upstream
  domain decomposition (route to ddd-skill), REST contract design (route to
  restful-api-design), or broad package taxonomy unless the user explicitly
  asks to redesign modules or services.
---

# Idiomatic Go — Implementation First

Improve concrete Go code. Do not jump to module catalogs or package-tree redesign.

## Scope

- Starts once the capability boundary is mostly known; the question is how to express it cleanly in Go.
- Route upstream boundary / bounded-context work to [[ddd-skill/SKILL.md|ddd-skill]].
- Route HTTP contract / OpenAPI work to [[restful-api-design/SKILL.md|restful-api-design]].
- If the current project has a `CLAUDE.md` with framework conventions (Uber FX, Ent ORM, etc.), those project-level patterns take precedence over this skill's general guidance.

## Escalation Policy

**Default**: stay inside the current function → file → package. Prefer local improvements.

**Escalate to structure work only when**:
- the user explicitly asks to split a package, module, or service
- duplicated rules or DTO churn span multiple packages
- pass-through layers dominate the call path and add no abstraction
- one behavior requires booting half the system to test
- state / invariant / concurrency ownership is unclear across package boundaries

If structure is wrong, name the concrete symptom first, then propose the smallest change that fixes it.

## Decision Order

1. Identify scope: function, file, package, service, or cross-package.
2. If local, stay local.
3. Clarify data shape with small strong types.
4. Make error meaning and cancellation explicit.
5. Route failure: recoverable → `error`; broken invariant → `panic`.
6. Make concurrency ownership obvious and testable.
7. Improve tests around public behavior.
8. Only then consider structural redesign, with evidence.

## Reference Loading Rules

Load **only** the references the task needs. Use the keyword table to decide.

| If the task mentions… | Load |
| --- | --- |
| error, wrap, sentinel, retry, panic, recover | [error-handling.md](references/error-handling.md) |
| test, mock, fuzz, table-driven, coverage, assert | [testing.md](references/testing.md) |
| goroutine, channel, context, cancel, timeout, deadline, errgroup | [context-cancellation.md](references/context-cancellation.md) |
| mutex, atomic, race, sync, happens-before, once | [memory-model-sync.md](references/memory-model-sync.md) |
| handler, middleware, HTTP client, CORS, rate limit, server | [http-middleware.md](references/http-middleware.md) |
| pool, connection, sql.DB, transaction, DSN | [database-sql-pooling-timeouts.md](references/database-sql-pooling-timeouts.md) |
| pprof, GC, scheduler, metrics, memory leak, runtime | [runtime-observability.md](references/runtime-observability.md) |
| allocation, benchmark, throughput, sync.Pool, escape | [performance.md](references/performance.md) |
| embed, fs, config file, build tag, cross-compile | [filesystems.md](references/filesystems.md) |
| package split, module redesign, architecture review, deep module | [philosophy.md](references/philosophy.md) — escalation path, not default |

## Panic vs Error Routing

| Situation | Choice | Why |
| --- | --- | --- |
| HTTP handler gets invalid JSON / bad params | `error` → 4xx | External input is untrusted |
| Repository returns timeout / connection error | `error` | I/O failure is recoverable |
| Business rule rejects a transition | `error` | Domain logic, not corruption |
| Internal assert finds trusted state corrupted | `panic` | Broken invariant, not ordinary failure |
| Startup route registration conflicts | `panic` / fatal exit | Miswired before serving traffic |
| Nil parent context in `WithCancel`-style call | `panic` | Fundamental precondition violated |

Ordinary failure → `error`. Broken assumption → fail hard.

## Tooling

```bash
gofmt -w .              # format
goimports -w .          # imports
golangci-lint run       # lint
go test -race ./...     # race detection
go test -bench=. -benchmem ./...  # benchmark
```
