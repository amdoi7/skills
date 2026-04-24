---
name: idiomatic-go
description: >
  Use for Go code work: implementing, debugging, refactoring, or reviewing types, interfaces, error semantics, context propagation, goroutine ownership, synchronization, package boundaries, tests, benchmarks, and explicit Go repo bootstrap.
---

# Idiomatic Go

Use this skill to keep Go work local, explicit, and easy to verify.

## When to Use

- Writing, fixing, reviewing, or debugging concrete Go code
- Tightening types, interfaces, naming, error semantics, or cancellation flow
- Clarifying goroutine ownership, synchronization, or testability
- Reviewing package design only after a concrete structural symptom appears
- Bootstrapping a Go repo or service only when the user explicitly asks

## When Not to Use

- Upstream domain decomposition before the capability boundary is known
- Pure OpenAPI or HTTP contract design without implementation pressure
- Non-Go tasks
- Framework-level repo rules that are already defined by project docs

If the current project has a `CLAUDE.md` or other local engineering guide, follow that first and use this skill as the Go-specific layer underneath it.

## Defaults

- Prefer the smallest change inside the current function → file → package
- Load only the reference files the task actually needs
- Name the concrete symptom before proposing a structural rewrite
- Prefer strong types and explicit control flow over clever helpers
- Treat tests as behavior constraints, not coverage decoration

## Local-First Routing

- Stay in the current function, file, or package before proposing package or layout changes.
- Load only the reference that matches the named symptom.
- Verify with the narrowest command that proves the change.
- Escalate to structure only when a concrete symptom crosses a local boundary.

## Modes

### Implementation Mode

Default mode. Make the smallest code change that fixes the concrete problem.

### Review Mode

Focus on observable risk:
- incorrect type or interface boundaries
- unclear error semantics
- unsafe concurrency ownership
- brittle or implementation-coupled tests
- unnecessary package churn

### Audit Mode

Use for codebase sweeps. Audit one concern at a time:
- boundary leaks and weak types
- error or logging misuse
- context and goroutine ownership
- brittle tests and weak verification
- pass-through layers and structural smells

Group findings by category, then propose the smallest repeated fix.

### Debug Mode

Reproduce first, narrow the failing path, then load only the references for the failing concern.

### Design-Escalation Mode

Use only when the problem is clearly structural: duplicated rules across packages, pass-through layers, unclear ownership, or tests that require booting half the system.

### Bootstrap Mode

Use only when the user explicitly asks to initialize a Go repo, scaffold a service, or review project layout. Right-size the structure; do not import a heavyweight architecture by reflex.

## Decision Order

1. Identify scope: function, file, package, repo bootstrap, or structural redesign.
2. If the issue is local, stay local.
3. Identify the primary concern and load only the matching references.
4. Clarify data shape, names, and boundary types.
5. Make error meaning, cancellation, and ownership explicit.
6. Strengthen tests around public behavior and failure paths.
7. Escalate to package or repo design only if the symptom is structural.

## Core Go Rules

- `ctx context.Context` is the first parameter. Do not store context in a struct.
- Define interfaces where they are consumed. Accept interfaces, return structs.
- Do not create interfaces before a real second use or a concrete test seam.
- Keep the zero value useful when possible.
- Return `error` for ordinary failure. Use `panic` only for broken invariants or startup-fatal states.
- Log an error or return it at a layer, not both.
- Prefer behavior-focused tests over implementation-detail tests.
- Prefer specific domain types over stringly typed parameters.

## Workflow

1. Determine the primary task.
2. Read only the matching references.
3. Make the smallest change that fixes the named symptom.
4. Verify with the narrowest commands that prove the change.
5. Escalate only if the evidence says the structure is wrong.

## Reference Loading Rules

- Choose one primary concern before reading references.
- For large references, read the table of contents first, then only the needed heading or file slice.
- In `testing.md`, choose the relevant slice: fundamentals, table tests, parallel/subtests, fuzzing, mocks, fixtures, benchmarks, or leak checks.
- In `error-handling.md`, choose the relevant slice: taxonomy, panic policy, wrapping, inspection, sentinel/custom errors, retry, cleanup, or nil interface traps.
- In `context-cancellation.md`, choose the relevant slice: context rules, cancellation sends, timeouts, errgroup, worker pools, singleflight, shutdown, or leak detection.
- If the concern is still unclear, inspect the failing code or reproduction before loading multiple long references.

## Quick Reference

| Topic | Use When | Reference |
| --- | --- | --- |
| Naming | packages, constructors, acronyms, boolean names, stuttering | [references/naming.md](references/naming.md) |
| Structs & Interfaces | interface placement, receiver choice, embedding, zero value, compile-time checks | [references/structs-interfaces.md](references/structs-interfaces.md) |
| Error Handling | wrapping, sentinel errors, `errors.Is/As`, panic routing, retries | [references/error-handling.md](references/error-handling.md) |
| Context & Cancellation | `context.Context`, timeouts, errgroup, goroutine lifetime | [references/context-cancellation.md](references/context-cancellation.md) |
| Safety | nil traps, `append` aliasing, defensive copies, conversion hazards, defer-in-loop | [references/safety.md](references/safety.md) |
| Testing | table-driven tests, race detection, fuzzing, mocks, examples | [references/testing.md](references/testing.md) |
| Synchronization | mutex, atomic, happens-before, ownership of shared state | [references/memory-model-sync.md](references/memory-model-sync.md) |
| HTTP | handlers, clients, middleware, timeouts, request boundaries | [references/http-middleware.md](references/http-middleware.md) |
| Database | `sql.DB`, pool sizing, transactions, deadlines, DSN concerns | [references/database-sql-pooling-timeouts.md](references/database-sql-pooling-timeouts.md) |
| Runtime & Observability | pprof, scheduler, metrics, leak investigation, tracing signals | [references/runtime-observability.md](references/runtime-observability.md) |
| Performance | benchmarks, allocations, escape analysis, `sync.Pool` | [references/performance.md](references/performance.md) |
| Filesystems & Build | `embed`, `fs`, config files, build tags, cross-compilation | [references/filesystems.md](references/filesystems.md) |
| Linters & Static Checks | `golangci-lint`, `govet`, `staticcheck`, `revive`, `errname`, `nolint` | [references/linters.md](references/linters.md) |
| Project Bootstrap | `go mod init`, repo scaffold, CLI/service layout, essential files | [references/project-bootstrap.md](references/project-bootstrap.md) |
| Structural Router | package split, deep modules, DTO churn, pass-through layers | [references/philosophy.md](references/philosophy.md) |
| Boundary Modeling | parse-don't-validate, DTO vs domain, trusted internal data | [references/philosophy-boundaries.md](references/philosophy-boundaries.md) |
| Good Taste | special cases, nesting, awkward local control flow | [references/philosophy-good-taste.md](references/philosophy-good-taste.md) |
| Complexity | deep modules, information hiding, change amplification | [references/philosophy-complexity.md](references/philosophy-complexity.md) |
| Design Review | structural review order, red flags, small CLs | [references/philosophy-review.md](references/philosophy-review.md) |

## Panic vs Error Routing

| Situation | Choice | Why |
| --- | --- | --- |
| Invalid external input at handler or CLI boundary | `error` | External input is untrusted |
| Timeout, DB error, network failure, cancellation | `error` | Operational failure is recoverable or translatable |
| Business rule rejects a transition | `error` | Domain logic, not corruption |
| Trusted internal invariant is broken | `panic` | The program is in a lying state |
| Startup wiring or route registration is invalid | `panic` / fatal exit | Serving traffic would be broken from the start |
| Nil parent context in `WithCancel`-style code | `panic` | Fundamental precondition violated |

Ordinary failure should stay explicit. Broken assumptions should fail hard.

## Review Output Contract

For review tasks, report in this order:

1. `Finding` — what is wrong
2. `Why it matters` — behavior, correctness, maintenance, or operability impact
3. `Minimal change` — the smallest fix that removes the risk
4. `Validation` — how to prove the fix

## Trigger Eval Prompts

Use these prompts when tuning routing or checking neighboring skill competition.

Should trigger:

- Review this Go code for context leaks and goroutine ownership.
- Fix Go error wrapping around repository calls so callers can use `errors.Is`.
- Refactor these Go interfaces; mocks are driving awkward package boundaries.
- Add behavior tests and a race check for worker-pool cancellation.

Should stay quiet:

- Rewrite this README to sound less AI.
- Design OpenAPI endpoints and Problem JSON for orders.
- Split this domain into bounded contexts before choosing language.
- Commit these changes as two Conventional Commits.
- Explain CPython GIL internals.
- Draw an ASCII architecture diagram.

## Tooling

```bash
gofmt -w .
goimports -w .
golangci-lint run
go test ./...
go test -race ./...
go test -bench=. -benchmem ./...
```
