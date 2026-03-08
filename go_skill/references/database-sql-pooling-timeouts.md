# database/sql Pooling & Context Timeouts

## Table of Contents

- [Pool Fundamentals and Defaults](#pool-fundamentals-and-defaults)
- [Tuning Knobs](#tuning-knobs)
- [Context-Aware Query/Exec/Tx Patterns](#context-aware-queryexectx-patterns)
- [DB.Stats Indicators and Diagnosis](#dbstats-indicators-and-diagnosis)
- [Failure Modes & Recovery Checklist](#failure-modes--recovery-checklist)
- [Common Pitfalls](#common-pitfalls)

## Pool Fundamentals and Defaults

`*sql.DB` is a **concurrency-safe connection pool**, not a single connection.

Default behavior (if not configured):

| Setting | Default | Effect |
|---------|---------|--------|
| `MaxOpenConns` | `0` (unlimited) | Can flood DB under traffic spikes |
| `MaxIdleConns` | `2` | Idle reuse is limited by default |
| `ConnMaxLifetime` | `0` (no limit) | Connections may live forever |
| `ConnMaxIdleTime` | `0` (no limit) | Idle connections may linger forever |

### Baseline setup

```go
func newDB(dsn string) (*sql.DB, error) {
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, fmt.Errorf("open db: %w", err)
    }

    db.SetMaxOpenConns(50)
    db.SetMaxIdleConns(25)
    db.SetConnMaxLifetime(30 * time.Minute)
    db.SetConnMaxIdleTime(5 * time.Minute)

    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()
    if err := db.PingContext(ctx); err != nil {
        _ = db.Close()
        return nil, fmt.Errorf("ping db: %w", err)
    }
    return db, nil
}
```

## Tuning Knobs

### Practical tuning heuristics

| Knob | Too Low | Too High | Starting Point |
|------|---------|----------|----------------|
| `MaxOpenConns` | App waits, latency spikes | DB saturation, queueing, timeouts | Align with DB server/concurrency budget |
| `MaxIdleConns` | Frequent reconnect/TLS/auth overhead | Wasted server connections | ~30-60% of MaxOpen |
| `ConnMaxLifetime` | Excess reconnect churn | Stale/poisoned conns survive too long | 15-60m (below infra hard limits) |
| `ConnMaxIdleTime` | Extra reconnects when bursty | Idle hoarding | 2-10m |

Tune with production traffic shape, not synthetic assumptions.

## Context-Aware Query/Exec/Tx Patterns

### Per-operation deadline (required)

```go
func (r *Repo) GetUser(ctx context.Context, id int64) (User, error) {
    qctx, cancel := context.WithTimeout(ctx, 200*time.Millisecond)
    defer cancel()

    var u User
    err := r.db.QueryRowContext(qctx,
        `SELECT id, email FROM users WHERE id = $1`, id,
    ).Scan(&u.ID, &u.Email)
    if err != nil {
        return User{}, fmt.Errorf("get user %d: %w", id, err)
    }
    return u, nil
}
```

### Transaction helper with timeout and rollback safety

```go
func withTx(ctx context.Context, db *sql.DB, fn func(*sql.Tx) error) error {
    txCtx, cancel := context.WithTimeout(ctx, 500*time.Millisecond)
    defer cancel()

    tx, err := db.BeginTx(txCtx, &sql.TxOptions{Isolation: sql.LevelReadCommitted})
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }
    defer tx.Rollback() // no-op after commit

    if err := fn(tx); err != nil {
        return err
    }
    if err := tx.Commit(); err != nil {
        return fmt.Errorf("commit tx: %w", err)
    }
    return nil
}
```

### Rules

- `ctx` always flows from request boundary down to DB call.
- Never issue DB calls without deadline in server code.
- Close `Rows` promptly; always check `rows.Err()`.

## DB.Stats Indicators and Diagnosis

### Key indicators

| Field | Meaning | Common Interpretation |
|------|---------|-----------------------|
| `OpenConnections` | total open conns | Near `MaxOpen` for long periods => pressure |
| `InUse` | actively used | Sustained high + wait growth => pool bottleneck |
| `Idle` | reusable conns | Always 0 under stable load => reconnect churn risk |
| `WaitCount` | waits due to no free conn | Increasing quickly => pool too small or slow queries |
| `WaitDuration` | total wait time | Rising p95 latency likely from connection queue |
| `MaxIdleClosed` | closed due to idle cap | Very high => `MaxIdleConns` too low |
| `MaxLifetimeClosed` / `MaxIdleTimeClosed` | closed by limits | Correlate with reconnect spikes |

### Lightweight monitor

```go
func logDBStats(db *sql.DB) {
    s := db.Stats()
    log.Printf("db open=%d inuse=%d idle=%d wait_count=%d wait=%s idle_closed=%d life_closed=%d",
        s.OpenConnections, s.InUse, s.Idle,
        s.WaitCount, s.WaitDuration,
        s.MaxIdleClosed, s.MaxLifetimeClosed,
    )
}
```

## Failure Modes & Recovery Checklist

### Typical failure modes

1. **Pool exhaustion**: `WaitCount/WaitDuration` surge, request latency up.
2. **Slow query cascade**: few long queries occupy many connections.
3. **No deadline propagation**: hung downstream holds connections indefinitely.
4. **Connection churn**: aggressive lifetime/idle limits trigger reconnect storms.
5. **DB failover stale conns**: old sockets fail until recycled.

### Recovery checklist

- [ ] Confirm whether bottleneck is DB CPU/IO, query plan, or pool sizing.
- [ ] Set explicit deadlines for all query/exec/tx paths.
- [ ] Cap `MaxOpenConns` to protect database.
- [ ] Set `ConnMaxLifetime` below network/LB/server hard disconnect thresholds.
- [ ] Add observability for `DB.Stats` and query latency/error labels.
- [ ] Validate rollback/commit paths and ensure rows are always closed.

## Common Pitfalls

| Pitfall | Problem | Fix |
|--------|---------|-----|
| Treating `*sql.DB` as a single connection | Misunderstands pool behavior | Think in pooled concurrency + queueing |
| No per-query timeout | Stuck queries consume pool slots | `QueryContext/ExecContext` with bounded `ctx` |
| Unlimited open connections | Traffic spike can overload DB | Set `SetMaxOpenConns` |
| Ignoring `WaitCount`/`WaitDuration` | Pool contention goes unnoticed | Alert on growth rate and latency correlation |
| Not closing `Rows` | Connection returned late or never | `defer rows.Close()` + check `rows.Err()` |
