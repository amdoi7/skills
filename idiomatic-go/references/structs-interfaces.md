# Structs, Interfaces & Receivers

Use Go's type system to make ownership and behavior obvious. Prefer small, concrete designs over speculative abstraction.

## Core Rules

- Define interfaces where they are consumed
- Accept interfaces, return structs
- Do not create an interface before a real second use or test seam appears
- Keep interfaces small and behavior-shaped
- Keep the zero value useful when possible
- Choose pointer receivers when methods mutate state or copying would be misleading
- Use embedding only when API promotion is intentional

## Define Interfaces Where Consumed

The consumer owns the contract.

```go
// package billing

type Clock interface {
    Now() time.Time
}

type Service struct {
    clock Clock
}
```

The producer package can export a concrete type. It does not need to export an interface just because one caller wants to abstract over it.

## Accept Interfaces, Return Structs

```go
func NewService(store Store, logger *slog.Logger) *Service
```

This keeps input flexible while keeping output concrete and discoverable.

Avoid returning interfaces from constructors unless the constructor truly hides multiple interchangeable implementations behind a deliberate boundary.

## Do Not Invent Interfaces Early

A single implementation rarely justifies an interface.

```go
// Prefer concrete first
type UserStore struct {
    db *sql.DB
}
```

Extract an interface later when:
- another consumer needs a narrower contract
- tests need a real seam at the boundary
- multiple implementations genuinely exist

## Keep Interfaces Small

Small interfaces are easier to understand, fake, and compose.

```go
type Reader interface {
    Read(p []byte) (int, error)
}

type ReadWriter interface {
    io.Reader
    io.Writer
}
```

Large kitchen-sink interfaces usually hide poor ownership or over-broad services.

## Useful Zero Value

Design structs so `var x T` is safe when practical.

```go
type Registry struct {
    items map[string]Item
}

func (r *Registry) Add(name string, item Item) {
    if r.items == nil {
        r.items = make(map[string]Item)
    }
    r.items[name] = item
}
```

If a type cannot have a useful zero value, make construction explicit and fail fast when required dependencies are missing.

## Pointer vs Value Receivers

Use pointer receivers when:
- the method mutates the receiver
- the receiver contains a mutex or other non-copyable state
- copying the receiver would be expensive or semantically wrong

Use value receivers when:
- the type is small and immutable by design
- copy semantics are intended and unsurprising

Be consistent across a type. Mixed receiver sets often confuse both callers and interface satisfaction.

## Embedding vs Named Fields

Use embedding only when you want the outer type to expose the embedded API.

```go
type Logger struct {
    *slog.Logger
}
```

Use named fields when the dependency is internal implementation detail.

```go
type Server struct {
    logger *slog.Logger
}
```

If the outer type is not supposed to look like the inner type, do not embed it.

## Compile-Time Interface Checks

Use compile-time assertions near the type definition when interface conformance is part of the contract.

```go
var _ http.Handler = (*Server)(nil)
```

Do not scatter these everywhere. Add them when the contract matters and breakage should fail the build clearly.

## Avoid `any` When a Type Will Do

Prefer concrete types or generics over `any`.

```go
func Contains[T comparable](xs []T, target T) bool
```

Use `any` only at true dynamic boundaries such as generic decoding, reflection, or intentionally heterogeneous containers.

## Common Mistakes

| Mistake | Better |
| --- | --- |
| defining producer-owned interfaces by reflex | define the interface in the consumer package |
| returning interface from `New...` | return `*ConcreteType` |
| one giant `Service` interface | split by actual behavior |
| broken zero value with nil-map panic | lazy init or explicit constructor |
| embedding a dependency accidentally | keep it as a named field |
| mixed pointer and value receiver semantics | choose one model per type |

## Review Questions

- Who actually owns this contract
- Is the interface discovered or imagined
- Does the zero value surprise callers
- Does the receiver choice match mutation and ownership
- Is embedding exposing too much surface area
