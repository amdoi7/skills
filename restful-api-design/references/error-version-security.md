# Error, Versioning, Security

## Error Model (Problem JSON)

Use RFC 7807-compatible payload:

```json
{
  "type": "https://api.example.com/problems/invalid-parameter",
  "title": "Invalid Parameter",
  "status": 400,
  "detail": "page_size must be between 1 and 100",
  "instance": "/orders?page_size=500"
}
```

Rules:
- Keep `type` stable and machine-meaningful.
- Keep `title` human-readable and short.
- Do not leak stack trace/internal infrastructure details.

## Versioning Strategy

- Prefer compatible evolution: add optional fields, preserve existing meanings.
- Reserve explicit version change for true breaking behavior.
- Keep one policy across all language implementations.
- If both media-type and URL versioning exist, document precedence clearly.

## Security Baseline

- Enforce authentication on all non-public endpoints.
- Enforce authorization by scope/role at boundary layer.
- Validate all input (path, query, header, body) before business logic.
- Apply rate limiting and abuse controls.
- Use HTTPS everywhere; define sensitive-field redaction in logs.
