# Performance, Memory Allocation & Throughput

## Table of Contents

- [Memory Allocation Basics](#memory-allocation-basics)
- [sync.Pool](#syncpool)
- [Concurrent Maps](#concurrent-maps)
- [Zero-Allocation Patterns](#zero-allocation-patterns)
- [Streaming & NDJSON](#streaming--ndjson)
- [Benchmarking](#benchmarking)
- [Profiling](#profiling)

## Memory Allocation Basics

### Stack vs Heap

| Allocation | When | Performance |
|------------|------|-------------|
| Stack | Small, fixed-size, doesn't escape | Fast (no GC) |
| Heap | Escapes function, large, dynamic | Slower (GC pressure) |

### Escape Analysis

```go
// ✅ Stack allocated (doesn't escape)
func sum(a, b int) int {
    result := a + b
    return result
}

// ❌ Heap allocated (escapes via pointer return)
func newInt(val int) *int {
    n := val
    return &n  // n escapes to heap
}

// Check escape analysis
// go build -gcflags="-m" ./...
```

### Pre-allocation

```go
// ❌ Multiple allocations during append
func badCollect(n int) []int {
    var result []int
    for i := 0; i < n; i++ {
        result = append(result, i)  // May reallocate
    }
    return result
}

// ✅ Single allocation
func goodCollect(n int) []int {
    result := make([]int, 0, n)  // Pre-allocate capacity
    for i := 0; i < n; i++ {
        result = append(result, i)
    }
    return result
}

// ✅ Even better if size known
func bestCollect(n int) []int {
    result := make([]int, n)
    for i := 0; i < n; i++ {
        result[i] = i
    }
    return result
}
```

## sync.Pool

### Basic Usage

```go
var bufPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func process(data []byte) string {
    buf := bufPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()  // IMPORTANT: Reset before returning
        bufPool.Put(buf)
    }()

    // Use buf...
    buf.Write(data)
    return buf.String()
}
```

### Buffer Middleware

```go
type BufferPool struct {
    pool sync.Pool
    size int
}

func NewBufferPool(size int) *BufferPool {
    return &BufferPool{
        size: size,
        pool: sync.Pool{
            New: func() interface{} {
                return bytes.NewBuffer(make([]byte, 0, size))
            },
        },
    }
}

func (p *BufferPool) Get() *bytes.Buffer {
    return p.pool.Get().(*bytes.Buffer)
}

func (p *BufferPool) Put(buf *bytes.Buffer) {
    buf.Reset()
    p.pool.Put(buf)
}

// HTTP middleware using pool
func BufferMiddleware(pool *BufferPool) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            buf := pool.Get()
            defer pool.Put(buf)

            // Use buffered response writer
            bw := &bufferedWriter{ResponseWriter: w, buf: buf}
            next.ServeHTTP(bw, r)
            buf.WriteTo(w)
        })
    }
}
```

### sync.Pool Rules

| Rule | Reason |
|------|--------|
| Always Reset before Put | Prevent memory leaks |
| Don't assume Get returns New | Pool may have existing objects |
| Objects may be GC'd anytime | Pool is not a cache |
| Thread-safe by design | No external locking needed |

## Concurrent Maps

### sync.Map vs Sharded Locks

| Approach | Best For |
|----------|----------|
| `sync.Map` | Write-once, read-many; disjoint keys |
| Sharded locks | High write contention; predictable performance |
| `sync.RWMutex` | Simple cases; low contention |

### Sharded Lock Map

```go
import "hash/fnv"

const numShards = 256

type ShardedMap[K comparable, V any] struct {
    shards [numShards]shard[K, V]
}

type shard[K comparable, V any] struct {
    sync.RWMutex
    m map[K]V
}

func NewShardedMap[K comparable, V any]() *ShardedMap[K, V] {
    sm := &ShardedMap[K, V]{}
    for i := range sm.shards {
        sm.shards[i].m = make(map[K]V)
    }
    return sm
}

func (sm *ShardedMap[K, V]) getShard(key K) *shard[K, V] {
    h := fnv.New32a()
    h.Write([]byte(fmt.Sprint(key)))
    return &sm.shards[h.Sum32()%numShards]
}

func (sm *ShardedMap[K, V]) Get(key K) (V, bool) {
    s := sm.getShard(key)
    s.RLock()
    defer s.RUnlock()
    v, ok := s.m[key]
    return v, ok
}

func (sm *ShardedMap[K, V]) Set(key K, value V) {
    s := sm.getShard(key)
    s.Lock()
    defer s.Unlock()
    s.m[key] = value
}

func (sm *ShardedMap[K, V]) Delete(key K) {
    s := sm.getShard(key)
    s.Lock()
    defer s.Unlock()
    delete(s.m, key)
}
```

### sync.Map Usage

```go
var cache sync.Map

// Store
cache.Store("key", value)

// Load
if v, ok := cache.Load("key"); ok {
    // Use v.(Type)
}

// LoadOrStore (atomic)
actual, loaded := cache.LoadOrStore("key", newValue)

// Delete
cache.Delete("key")

// Range (not atomic across iterations)
cache.Range(func(key, value interface{}) bool {
    // Process key, value
    return true  // Continue iteration
})
```

## Zero-Allocation Patterns

### String to Bytes (Unsafe)

```go
import "unsafe"

// ⚠️ Use only for read-only operations
func stringToBytes(s string) []byte {
    return unsafe.Slice(unsafe.StringData(s), len(s))
}

// ⚠️ String must not be modified
func bytesToString(b []byte) string {
    return unsafe.String(unsafe.SliceData(b), len(b))
}
```

### Avoiding Interface Boxing

```go
// ❌ Allocates (interface boxing)
func badLog(format string, args ...interface{}) {
    fmt.Printf(format, args...)
}

// ✅ Type-specific (no boxing)
func logInt(msg string, val int) {
    // Use strconv for int to string
}

// ✅ Or use structured logging
type LogEntry struct {
    Message string
    IntVal  int
}
```

### Reusing JSON Encoder (Important Note)

`json.Encoder` in Go does **not** provide `Reset` in current Go versions, so pooling encoders directly is not practical.

```go
// ❌ PSEUDOCODE ONLY (won't compile): shown to explain the idea
// enc.Reset(w) does not exist in encoding/json
// func encodeJSON(w io.Writer, v interface{}) error {
//     enc := encoderPool.Get().(*json.Encoder)
//     defer encoderPool.Put(enc)
//     enc.Reset(w) // unavailable API
//     return enc.Encode(v)
// }
```

Use one of these **copy-paste safe** alternatives:

```go
// ✅ Simple and correct: create a new encoder per call
func encodeJSON(w io.Writer, v any) error {
    return json.NewEncoder(w).Encode(v)
}
```

```go
// ✅ Pool bytes.Buffer instead of json.Encoder
var jsonBufPool = sync.Pool{
    New: func() any { return new(bytes.Buffer) },
}

func encodeJSONWithBufferPool(w io.Writer, v any) error {
    buf := jsonBufPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        jsonBufPool.Put(buf)
    }()

    if err := json.NewEncoder(buf).Encode(v); err != nil {
        return err
    }

    _, err := w.Write(buf.Bytes())
    return err
}
```

## Streaming & NDJSON

### NDJSON Stream Reader

```go
func processNDJSON(r io.Reader, process func(json.RawMessage) error) error {
    scanner := bufio.NewScanner(r)

    // Handle long lines
    buf := make([]byte, 0, 64*1024)
    scanner.Buffer(buf, 10*1024*1024)  // Max 10MB per line

    for scanner.Scan() {
        line := scanner.Bytes()
        if len(line) == 0 {
            continue
        }

        // Avoid allocation: use RawMessage
        if err := process(json.RawMessage(line)); err != nil {
            return fmt.Errorf("process line: %w", err)
        }
    }

    return scanner.Err()
}
```

### Streaming JSON Array

```go
func streamJSONArray(w io.Writer, items <-chan Item) error {
    enc := json.NewEncoder(w)
    w.Write([]byte("["))

    first := true
    for item := range items {
        if !first {
            w.Write([]byte(","))
        }
        first = false

        if err := enc.Encode(item); err != nil {
            return err
        }
    }

    w.Write([]byte("]"))
    return nil
}
```

## Benchmarking

### Writing Benchmarks

```go
func BenchmarkProcess(b *testing.B) {
    data := generateTestData()

    b.ResetTimer()  // Exclude setup time
    for i := 0; i < b.N; i++ {
        process(data)
    }
}

// With memory stats
func BenchmarkProcessAllocs(b *testing.B) {
    data := generateTestData()

    b.ReportAllocs()
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        process(data)
    }
}

// Parallel benchmark
func BenchmarkProcessParallel(b *testing.B) {
    data := generateTestData()

    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            process(data)
        }
    })
}
```

### Running Benchmarks

```bash
# Basic benchmark
go test -bench=. -benchmem ./...

# Compare benchmarks
go test -bench=. -count=10 > old.txt
# Make changes...
go test -bench=. -count=10 > new.txt
benchstat old.txt new.txt

# CPU profile
go test -bench=BenchmarkProcess -cpuprofile=cpu.out
go tool pprof cpu.out

# Memory profile
go test -bench=BenchmarkProcess -memprofile=mem.out
go tool pprof -alloc_space mem.out
```

## Profiling

### pprof Integration

```go
import (
    "net/http"
    _ "net/http/pprof"
)

func main() {
    // Exposes /debug/pprof/*
    go func() {
        http.ListenAndServe(":6060", nil)
    }()

    // Your application...
}
```

### Profile Analysis

```bash
# CPU profile
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Heap profile
go tool pprof http://localhost:6060/debug/pprof/heap

# Goroutine profile
go tool pprof http://localhost:6060/debug/pprof/goroutine

# In pprof:
# (pprof) top10
# (pprof) web          # Opens in browser
# (pprof) list funcName
```

### Common Optimizations

| Issue | Solution |
|-------|----------|
| High allocations | Use sync.Pool, pre-allocate slices |
| String concatenation | Use strings.Builder |
| Interface conversions | Use type-specific functions |
| Map lookups | Use sharded maps for high contention |
| Large copies | Pass pointers (but watch escape analysis) |
