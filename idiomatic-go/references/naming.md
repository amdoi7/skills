# Naming & Call-Site Readability

Good Go naming reduces explanation pressure at the call site. Prefer names that make code read naturally in context.

## Core Rules

- Packages are lowercase, short, and specific
- Exported names use `UpperCamelCase`; unexported names use `lowerCamelCase`
- Avoid stuttering at the call site
- Constructor names are usually `New` unless the package exports multiple primary types
- Boolean names should read like predicates
- Error variables use `ErrXxx`; error types use `XxxError`
- Error strings are lowercase and have no trailing punctuation
- Acronyms are consistently cased: `URL`, `HTTP`, `ID`

## Package Names

Prefer semantic package names over buckets such as `util`, `common`, or `helpers`.

```go
// Good
package auth
package retry
package pricing

// Bad
package utils
package helpers
package common
```

A package name should tell the reader what concept lives there, not that it contains miscellaneous leftovers.

## Avoid Stuttering

The package name already appears at the call site. Do not repeat it in the exported identifier.

```go
// Good
http.Client
json.Decoder
user.New()

// Bad
http.HTTPClient
json.JSONDecoder
user.NewUser()
```

## Constructors

When a package has one primary constructible type, prefer `New()`.

```go
func New(store Store) *Service
```

If the package exports several major constructible types, use `NewTypeName()`.

```go
func NewRequest(method, url string) (*Request, error)
func NewServeMux() *ServeMux
```

## Boolean Names

Boolean names should read as true-or-false questions.

```go
isReady bool
hasQuota bool
canRetry bool
```

For methods, keep the predicate form.

```go
func (s *Server) IsHealthy() bool
func (u *User) HasRole(role Role) bool
```

Do not hide booleans behind vague names like `state`, `flag`, or bare adjectives that lose meaning at the call site.

## Error Names

```go
var ErrNotFound = errors.New("user: not found")

type ValidationError struct {
    Field string
    Err   error
}
```

Guidelines:
- sentinel variables: `ErrTimeout`, `ErrConflict`, `ErrInvalidState`
- custom types: `PathError`, `ValidationError`
- strings: lowercase, no punctuation

## Interfaces, Types, and Methods

- Interface names usually follow behavior: `Reader`, `Store`, `Clock`
- Type names should reflect domain concepts, not storage shape
- Method names should expose behavior, not internal mechanism

```go
// Good
type InvoiceStore interface {
    Save(ctx context.Context, invoice Invoice) error
}

// Bad
type InvoiceRepositoryInterface interface {
    SaveInvoiceRecord(ctx context.Context, invoice Invoice) error
}
```

## Acronyms

Use one casing convention per acronym and stay consistent.

```go
UserID
ParseURL
HTTPClient
```

Not:

```go
UserId
ParseUrl
HttpClient
```

## Test Names

- Test functions: `TestXxx` or `TestType_Method`
- Benchmarks: `BenchmarkXxx`
- Fuzz tests: `FuzzXxx`
- Example docs: `ExampleXxx`
- Subtest names should be short descriptive phrases

```go
func TestParser_Parse(t *testing.T) {
    t.Run("valid header", func(t *testing.T) {})
    t.Run("empty input", func(t *testing.T) {})
}
```

## Common Mistakes

| Mistake | Better |
| --- | --- |
| `util`, `common`, `helper` packages | use the actual concept name |
| `http.HTTPClient` | `http.Client` |
| `GetName()` getter | `Name()` |
| `UserId` | `UserID` |
| `connected bool` | `isConnected bool` |
| `NewUser()` in package `user` | `New()` |
| `ValidationErr` type | `ValidationError` |
| `"Invalid ID"` | `"invalid id"` |

## Quick Review Checklist

- Does the name reduce or add ambiguity at the call site
- Does it repeat the package name unnecessarily
- Does it encode behavior instead of implementation detail
- Is the acronym casing consistent with the rest of the repo
- Does the boolean read like a predicate
