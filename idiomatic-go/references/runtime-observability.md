# Runtime Observability

## Table of Contents

- [Runtime Observability Fundamentals](#runtime-observability-fundamentals)
- [runtime/metrics Essentials](#runtimemetrics-essentials)
- [Key Metrics and Interpretation](#key-metrics-and-interpretation)
- [Correlating runtime/metrics with pprof](#correlating-runtimemetrics-with-pprof)
- [GC/Scheduler Triage Playbook](#gcscheduler-triage-playbook)
- [Common Misreads](#common-misreads)
- [Checklist](#checklist)

## Runtime Observability Fundamentals

### Why `runtime/metrics` + `pprof`

| Tool | Best For | Caveat |
|------|----------|--------|
| `runtime/metrics` | Low-overhead runtime counters/gauges/histograms | Many values are cumulative; need delta/rate |
| `pprof` | Root-cause in code paths (CPU/heap/block/mutex/goroutine) | Snapshot-based; must sample at right time |

Use metrics to detect **what changed**, use pprof to explain **where it changed**.

### Minimal instrumentation shape

```go
// Sample every 5-15s, convert cumulative metrics to rates.
var metricNames = []string{
    "/gc/cycles/total:gc-cycles",
    "/gc/heap/live:bytes",
    "/gc/pauses:seconds",
    "/sched/latencies:seconds",
    "/sched/goroutines:goroutines",
    "/sync/mutex/wait/total:seconds",
}
```

## runtime/metrics Essentials

### Read selected metrics

```go
func readRuntimeMetrics(names []string) map[string]metrics.Value {
    samples := make([]metrics.Sample, len(names))
    for i, name := range names {
        samples[i].Name = name
    }
    metrics.Read(samples)

    out := make(map[string]metrics.Value, len(samples))
    for _, s := range samples {
        out[s.Name] = s.Value
    }
    return out
}
```

### Convert cumulative counters to rates

```go
// For counter-like uint64/float64 metrics.
func rate(curr, prev float64, d time.Duration) float64 {
    if d <= 0 || curr < prev {
        return 0
    }
    return (curr - prev) / d.Seconds()
}
```

### Histogram percentile helper

```go
func histogramPercentile(h *metrics.Float64Histogram, p float64) float64 {
    if h == nil || len(h.Counts) == 0 {
        return 0
    }
    var total uint64
    for _, c := range h.Counts {
        total += c
    }
    if total == 0 {
        return 0
    }

    target := uint64(float64(total) * p)
    if target == 0 {
        target = 1
    }

    var seen uint64
    for i, c := range h.Counts {
        seen += c
        if seen >= target {
            return h.Buckets[i+1]
        }
    }
    return h.Buckets[len(h.Buckets)-1]
}
```

## Key Metrics and Interpretation

### GC and memory pressure

| Metric | Interpretation | Typical Signal |
|-------|----------------|----------------|
| `/gc/heap/live:bytes` | Live heap after marking | Rising baseline => retention leak risk |
| `/gc/heap/goal:bytes` | GC target heap size | Goal too close to live => frequent GC |
| `/gc/pauses:seconds` (histogram) | STW pause distribution | P99 spikes => latency tail impact |
| `/memory/classes/heap/free:bytes` vs `released` | Reserved vs returned memory | High free+low released => RSS may stay high |

### Scheduler and contention

| Metric | Interpretation | Typical Signal |
|-------|----------------|----------------|
| `/sched/latencies:seconds` | Runnable-to-running delay | P95/P99 up => CPU saturation / long-running goroutines |
| `/sched/goroutines:goroutines` | Current goroutine count | Monotonic growth => leak / blocked fan-out |
| `/sync/mutex/wait/total:seconds` | Total mutex wait time | Fast growth => lock contention, long critical section |

## Correlating runtime/metrics with pprof

### Correlation map

| Metrics symptom | Verify with pprof |
|-----------------|-------------------|
| `/sync/mutex/wait/total` rate increases | `pprof mutex`, `pprof block` |
| `/sched/latencies` tail increases | `pprof goroutine`, `pprof cpu` |
| `/gc/heap/live` baseline grows | `pprof heap` (inuse_space, alloc_space) |
| `/gc/pauses` P99 rises | `pprof cpu` around GC-heavy windows |

### Useful commands

```bash
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
go tool pprof http://localhost:6060/debug/pprof/heap
go tool pprof http://localhost:6060/debug/pprof/mutex
go tool pprof http://localhost:6060/debug/pprof/block
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

## GC/Scheduler Triage Playbook

### Step-by-step

1. **Detect anomaly**: alert on rate/percentile, not raw cumulative numbers.
2. **Confirm scope**: GC-only, scheduler-only, or both.
3. **Capture profile window**: collect CPU + heap + mutex/block during anomaly.
4. **Classify bottleneck**:
   - GC churn (alloc-heavy hot path)
   - Retention (long-lived references)
   - Lock contention (single hot mutex)
   - Scheduling starvation (too many runnable goroutines)
5. **Apply smallest fix**:
   - reduce allocations / reuse buffers
   - shorten critical section / sharded locks
   - bound fan-out (`errgroup.SetLimit`)
6. **Re-check same metrics + profiles** to confirm regression is gone.

## Common Misreads

| Misread | Why Wrong | Correct View |
|--------|-----------|--------------|
| “Counter grew, so issue exists now” | Counter is cumulative | Use delta/rate over fixed window |
| “Heap total high means leak” | Reserved memory != retained objects | Check `live` trend + heap profile retainers |
| “Goroutines increased briefly, leak” | Burst can be normal | Look for monotonic growth + blocked stack traces |
| “Mutex wait high => switch to atomics” | Primitive swap may break invariants | First reduce contention / redesign ownership |

## Checklist

- [ ] Metrics are sampled on fixed intervals and stored as rates/percentiles where needed.
- [ ] Alerts distinguish level vs growth (gauge vs counter semantics).
- [ ] pprof endpoints are enabled in non-public/debug-safe environments.
- [ ] Incident runbook ties specific metrics to specific profiles.
- [ ] Post-fix validation compares before/after metric windows.
