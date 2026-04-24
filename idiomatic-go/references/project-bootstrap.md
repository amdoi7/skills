# Project Bootstrap & Layout

Use this reference only when the user explicitly asks to start a Go repo, scaffold a service, or review top-level layout. Bootstrap is an explicit mode, not the default path for ordinary code work.

## Best Practices Summary

- Ask first: module path, project type, transport, stateful dependencies, architecture, and DI preference
- Start with the smallest working layout that matches the project type
- Keep `cmd/` thin and put reusable application code under `internal/`
- Prefer capability-first packages over copying `handler/service/repository` shells everywhere
- Use `pkg/` only for code intentionally exported outside the repo
- Verify the repo with `go test`, `go build`, and lint before calling the scaffold done

## Ask First

Before scaffolding, clarify the minimum facts that affect structure:

1. module path
2. project type: CLI, library, service, or monorepo
3. Go version
4. expected transport: HTTP, gRPC, worker, or none
5. database or external stateful dependencies
6. architecture preference, if any
7. dependency injection preference, if any
8. required CI, Docker, or editor setup

Do not import a large architecture before these are known.

## Right-Size the Layout

### Small CLI

```text
.
├── cmd/<name>/main.go
├── internal/
├── go.mod
└── README.md
```

### Library

```text
.
├── <package files>
├── internal/        # only if private helpers are needed
├── go.mod
└── README.md
```

### Service

```text
.
├── cmd/<service>/main.go
├── internal/
│   ├── auth/
│   │   ├── http.go
│   │   ├── users.go
│   │   └── tokens.go
│   ├── billing/
│   │   ├── http.go
│   │   ├── invoices.go
│   │   └── payments.go
│   └── platform/
│       ├── config/
│       ├── db/
│       └── httpserver/
├── go.mod
└── README.md
```

Split by business capability first. Keep cross-cutting technical packages under `internal/platform/` or another clearly technical home.

If one capability grows large, split inside that capability by real responsibility. Do not clone the same layer tree into every folder by reflex.

## Baseline Files

Most Go repos need only a small baseline at first:

- `go.mod`
- `cmd/<name>/main.go` for executables
- `internal/` for private packages
- `.gitignore`
- `.golangci.yml` if linting is expected
- `README.md` with run, test, and generate commands

Add Docker, CI, code generation, or editor config only when the project actually uses them.

## Bootstrap Rules

- `main.go` should stay thin: parse config, wire dependencies, call `Run()`
- Business logic should not live in `cmd/`
- Prefer manual construction first; add DI frameworks only with a clear need
- Use `internal/` by default; use `pkg/` only for code meant for external consumers
- Keep generated code isolated and clearly marked
- Keep boundary translation at the edge of each capability
- Verify the minimal path before adding auxiliary tooling

## Minimal Verification

After bootstrap, prove the repo is real, not decorative.

```bash
go mod tidy
gofmt -w .
go test ./...
go build ./...
```

If linting is configured, run it too.

```bash
golangci-lint run
```

## Common Mistakes

| Mistake | Better |
| --- | --- |
| starting with heavyweight clean architecture for a tiny tool | keep a flat layout first |
| putting business logic in `cmd/` | move it into `internal/` |
| copying `handler/service/repository` into every feature folder | group by capability first, split later only when earned |
| adding `pkg/` for everything | use `internal/` unless external reuse is intentional |
| adding FX, Wire, or Dig before wiring pain exists | construct manually first |
| copying a full microservice template blindly | ask first, then right-size |

## Escalate Carefully

If the user wants a full production service template with CI, Docker, DI, ORM, and codegen, that is no longer ordinary idiomatic-Go implementation work. Treat it as explicit bootstrap design and keep the checklist visible instead of burying decisions in generated scaffolding.
