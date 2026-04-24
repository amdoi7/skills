# Linters & Static Checks

Use this reference when the task mentions lint failures, `golangci-lint`, `govet`, `staticcheck`, `revive`, `errname`, `nolint`, or codebase-wide hygiene audits.

## Best Practices Summary

- Run `gofmt`, `goimports`, and `golangci-lint` as the default local hygiene path
- Treat `govet` and `staticcheck` findings as correctness signals, not style noise
- Use naming and comment linters to support clarity, not to satisfy trivia
- Prefer fixing the code over suppressing the linter
- Use `nolint` only with a narrow scope and a concrete reason
- Audit one lint category at a time when sweeping a large codebase

## Default Stack

```bash
gofmt -w .
goimports -w .
golangci-lint run
```

When isolating a specific class of issues, it is often useful to run the underlying tools directly.

```bash
go vet ./...
staticcheck ./...
```

## What to Reach For

| Concern | Tooling |
| --- | --- |
| suspicious code, bad `printf`, struct tag issues, copylocks | `govet` |
| ineffective assignments, impossible checks, API misuse, dead branches | `staticcheck` |
| naming, comments, package hygiene, exported API style | `revive` |
| sentinel error and error type naming | `errname` |
| import grouping and unused imports | `goimports` |
| broad multi-linter CI and local gating | `golangci-lint` |

## Audit Strategy

For codebase sweeps, do not chase every warning at once. Split by category:

1. correctness first: `govet`, `staticcheck`
2. error and naming clarity: `errname`, `revive`
3. formatting and imports: `gofmt`, `goimports`
4. only then decide whether remaining findings are worth cleanup now

## `nolint` Rules

Use `nolint` only when all three are true:

- the warning is understood
- the code is intentionally correct
- the suppression can be kept narrow

```go
//nolint:errcheck // response body close error is intentionally ignored here
resp.Body.Close()
```

Bad:

```go
//nolint
func doStuff() {}
```

Good suppressions are specific and justified. Broad silent suppression is design debt.

## Review Questions

- Is this finding about correctness, clarity, or only stylistic preference
- Would fixing it simplify the code or only satisfy a tool
- Is a suppression narrower than the code change needed to remove it
- Does the linter reveal a deeper boundary or ownership problem

## Cross-Links

- Naming-related findings: [naming.md](naming.md)
- Error-related findings: [error-handling.md](error-handling.md)
- Structural review after repeated linter findings: [philosophy-review.md](philosophy-review.md)
