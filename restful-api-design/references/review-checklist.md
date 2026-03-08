# REST API Review Checklist

## Contract

- Resource model is clear and stable.
- URI naming follows plural + kebab-case.
- Query naming follows snake_case.
- Method semantics and status codes are consistent.

## Error and Compatibility

- Problem JSON is used consistently.
- Error `type` values are defined and stable.
- Compatibility impact is explicitly stated for each change.

## Security

- AuthN/AuthZ boundaries are explicit.
- Input validation exists at boundary.
- Rate limit and abuse controls are defined.
- Sensitive data handling/log redaction is defined.

## Data and Query

- Timestamp/enum handling is specified.
- Pagination/filter/sort contract is explicit.
- Response size and cacheability are addressed.

## Delivery Template

When reviewing or designing, always provide:

1. Endpoint contract table.
2. Error model and domain error taxonomy.
3. Compatibility/versioning decision.
4. Security controls summary.
5. Performance and query strategy summary.
6. Language mapping notes (Python/Go/others as requested).
