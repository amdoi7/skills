# Safety & Defensive Coding

Safety work prevents crashes, silent corruption, and misleadingly valid code paths. Prefer explicit invariants over post-hoc damage control.

## Core Rules

- Never rely on a typed nil inside an interface being `nil`
- Never write to a nil map
- Assume `append` may share backing storage
- Return defensive copies from exported accessors when callers must not mutate internals
- Do not hide overflow or narrowing conversions
- Do not `defer` resource cleanup inside long loops without extracting a helper
- Keep zero values useful or fail fast with an explicit constructor

## Nil Interface Trap

An interface is `nil` only when both dynamic type and value are nil.

```go
func maybeHandler(enabled bool) http.Handler {
    if !enabled {
        return nil
    }

    var h *myHandler
    return h // non-nil interface holding a nil pointer
}
```

If `nil` is the intended meaning, return `nil` explicitly.

## Nil Map, Slice, and Channel Behavior

| Type | Read | Write | Range | Notes |
| --- | --- | --- | --- | --- |
| map | zero value on lookup | panic | zero iterations | initialize before write |
| slice | index panic | index panic | zero iterations | append on nil slice is fine |
| channel | blocks | blocks | blocks | nil channel is often a bug |

```go
var m map[string]int
m["x"] = 1 // panic
```

## `append` Aliasing

Slices may share the same backing array.

```go
a := make([]int, 2, 4)
b := append(a, 3)
b[0] = 99 // may also change a[0]
```

If the new slice must not mutate the old one, force a copy.

```go
b := append(a[:len(a):len(a)], 3)
```

## Defensive Copies

Do not leak internal mutable state through exported fields or accessors.

```go
type Config struct {
    hosts []string
}

func (c *Config) Hosts() []string {
    return slices.Clone(c.hosts)
}
```

The same rule applies to maps. Return a clone when callers should not own the original state.

## Numeric Conversions

Go silently truncates narrowing integer conversions.

```go
if x > math.MaxInt32 || x < math.MinInt32 {
    return fmt.Errorf("value %d overflows int32", x)
}
small := int32(x)
```

Use explicit bounds checks before narrowing. For money, prefer integer units over `float64`.

## `defer` in Loops

`defer` runs at function exit, not at loop iteration end.

```go
for _, path := range paths {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close() // accumulates until the whole function returns
}
```

Extract the loop body into a helper when each iteration owns a resource.

## Useful Zero Values

A good zero value avoids surprise.

```go
var mu sync.Mutex
var buf bytes.Buffer
```

But this is unsafe until initialized lazily or via constructor:

```go
type Cache struct {
    items map[string]Item
}
```

If the zero value cannot be useful, make construction explicit and validate required dependencies immediately.

## Concurrency Safety Signals

Use this reference for local safety rules. For deeper synchronization semantics, also load [memory-model-sync.md](memory-model-sync.md).

Watch for:
- shared maps without synchronization
- copying values that contain `sync.Mutex`
- goroutines that outlive their owner without cancellation
- channels with unclear close ownership

## Common Mistakes

| Mistake | Better |
| --- | --- |
| returning typed nil as interface | return plain `nil` |
| writing to nil map | lazy init or constructor init |
| exposing mutable slice field | unexported field plus clone accessor |
| `float64` money | integer cents or domain type |
| `defer` inside large loop | extract helper |
| assuming `append` copies | reason about capacity or force copy |

## Review Questions

- Could this type panic at its zero value
- Is internal mutable state escaping
- Are nils represented honestly
- Could a conversion silently corrupt data
- Does resource cleanup happen at the intended lifetime boundary
