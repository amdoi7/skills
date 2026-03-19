---
name: restful-api-design
description: Design, review, and refactor RESTful APIs using Zalando-style standards in a language-agnostic way, with concrete rules for URI naming, HTTP method semantics, Problem JSON error handling, versioning, security, data formats, pagination/filtering, and performance. Use when users ask to design API endpoints, write OpenAPI contracts, standardize backend API conventions, or review REST API quality.
---

# RESTful API Design

## Overview

Use this skill to produce consistent, production-ready REST API designs and review feedback based on the bundled Zalando-oriented guide. Treat specification and behavior as the source of truth.

## Workflow

1. Clarify API scope.  
Identify resource boundaries, actor roles, auth model, backward-compat constraints, and non-functional requirements.

2. Define API contract first.  
Draft endpoint inventory and OpenAPI-first contract before implementation.

3. Apply design rules by topic.  
Use the reference matrix below and only load files relevant to the current task.

4. Produce implementation-ready artifacts.  
Deliver URI table, method semantics, request/response schemas, error model, pagination/filtering, and versioning strategy.

5. Run compliance check.  
Validate against `references/review-checklist.md` before finalizing.

## Reference Matrix

- `references/principles.md`: Core principles, API-first mindset, consistency and conservative evolution.
- `references/resource-http-rules.md`: URI/resource naming, method semantics, status codes, idempotency.
- `references/error-version-security.md`: Problem JSON, compatibility/versioning rules, security baseline.
- `references/data-query-performance.md`: Data format, pagination/filtering/sorting, caching/compression/performance.
- `references/review-checklist.md`: Review checklist and required output template.

## Fast Path By Request Type

- New API design: Load `principles`, `resource-http-rules`, then `error-version-security`.
- API review/audit: Load `review-checklist`, then `resource-http-rules`, `error-version-security`, `data-query-performance`.
- Implementation guidance: Load `resource-http-rules`, `error-version-security`, and `data-query-performance`.
- Pagination/search tasks: Load `data-query-performance` first, then `resource-http-rules`.

## Output Requirements

Always return concrete deliverables instead of abstract advice.

1. Endpoint contract table.  
Include method, path, purpose, idempotency, request schema, response schema, error codes.

2. Error model.  
Use Problem JSON shape and define domain error types.

3. Compatibility plan.  
State whether change is backward compatible and show migration/versioning approach.

4. Security controls.  
State authN/authZ model, rate limit policy, and sensitive data handling.

5. Performance notes.  
State pagination strategy, cacheability, and response size controls.

## Quick Search Hints

Use targeted grep on references to reduce context load.

```bash
rg -n "原则|一致性|兼容|API First" references/principles.md
rg -n "URI|kebab|snake|POST|PUT|PATCH|状态码|幂等" references/resource-http-rules.md
rg -n "Problem JSON|RFC 7807|version|认证|授权|限流" references/error-version-security.md
rg -n "分页|游标|过滤|排序|缓存|压缩|性能" references/data-query-performance.md
```
