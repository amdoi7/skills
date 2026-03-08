# Data, Query, Performance

## Data Format Rules

- Use JSON as default payload format.
- Use snake_case for JSON fields.
- Use RFC 3339 / ISO 8601 timestamps with explicit timezone.
- Keep enum behavior forward-compatible (unknown values must not crash old clients).

## Pagination, Filtering, Sorting

- Prefer cursor pagination for large/high-churn datasets.
- Support offset pagination when UX needs page jumps.
- Keep filtering syntax explicit and composable.
- Keep sorting syntax deterministic and allow multi-field sort.
- Return pagination metadata consistently.

## Performance Baseline

- Enable compression for sizable responses.
- Use cache headers and conditional requests where safe.
- Avoid over-fetching: support sparse fields/partial response where needed.
- Define timeout, retry, and circuit-breaker policies for upstream dependencies.
- Measure p95/p99 latency and error rates per endpoint.
