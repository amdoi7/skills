# Resource & HTTP Rules

## URI and Naming

- Use plural nouns for resource collections: `/users`, `/orders`.
- Use kebab-case for path segments: `/user-profiles/{user_id}`.
- Use snake_case for query parameters: `page_size`, `created_at_gte`.
- Avoid verbs in URI; express action through HTTP methods.
- Limit nesting depth; prefer query-based linking over deep path trees.

## HTTP Method Semantics

- `GET`: read-only, safe, cacheable where possible.
- `POST`: create or command; support idempotency key when retries are likely.
- `PUT`: full replacement, idempotent.
- `PATCH`: partial update, idempotent when contract allows deterministic merge.
- `DELETE`: delete/deactivate; remain idempotent from client perspective.
- `HEAD`/`OPTIONS`: support when relevant for metadata/capability discovery.

## Status Code Baseline

- 200: successful read/update
- 201: created
- 202: accepted (async)
- 204: success without body
- 400: invalid request syntax/parameters
- 401: unauthenticated
- 403: unauthorized
- 404: not found
- 409: conflict/state violation
- 412: precondition failed
- 422: semantically invalid payload
- 429: rate limited
- 500/503: server/service failure

## Idempotency Rules

- Define whether each mutating endpoint is idempotent.
- For retry-prone creates, accept an idempotency key and return same logical result.
- Do not tie idempotency behavior to a specific language implementation detail.
