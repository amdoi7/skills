# Testing

## Testing Priorities

1. Test behavior rather than implementation detail. Assert public results, visible side effects, and business outcomes.
2. Mock system boundaries rather than internal layers. Do not mock `router -> service -> repository` handoffs just to prove control flow.
3. Prefer an in-memory database over mocking the database API when real query behavior, constraints, or transactions matter.
4. Do not over-focus on taxonomy labels. If runtime and stability are acceptable, optimize for release confidence first.
5. Treat coverage as a means to expose hidden bugs, dead code, and redundant branches rather than as the goal.

Design signals from tests:

- If a scenario can only be tested by piercing many internal layers, the boundary is probably wrong.
- If one behavior requires mocking a chain of internal collaborators, the module split is probably wrong.
- If the same rule is asserted in many layers and test shapes, knowledge may be leaking.

## Choosing Test Shapes

Use test shapes as organizational tools, not as doctrine.

| Shape | Best Fit | Keep It Healthy By |
| --- | --- | --- |
| Unit | Pure domain logic, value objects, local transformations | asserting behavior, not call graphs |
| Integration | Real queries, adapters, wiring, and boundary behavior | using real boundaries where they matter |
| E2E | A few critical user journeys | keeping only flows that materially raise release confidence |
| Smoke | Shortest public paths for tools or SDKs | one or two high-frequency flows |
| Feature | Capability-focused regression for tools or SDKs | grouping by behavior, not files |
| Release | Smallest gate before a cut | blocking only on user-visible promises |

Choose the smallest mix that gives confidence without making the suite too slow or too flaky.

## Examples Are Not Tests

- Use `examples/` to teach, explore, and demonstrate realistic flows.
- Use `tests/` to make stable assertions and gate regressions.
- Let examples stay readable and scenario-rich; let tests stay short, deterministic, and specific.
- If an example becomes release-critical, extract the asserted behavior into `tests/` instead of treating the example itself as the gate.

## Behavior-First Examples

```python
import pytest
from app.services.user_service import UserService


class InMemoryUserRepo:
    def __init__(self):
        self._users = {1: User(id=1, email="test@example.com")}

    def get_by_id(self, user_id: int) -> User | None:
        return self._users.get(user_id)

    def save(self, user: User) -> User:
        return user


class TestUserService:
    def test_create_user_success(self):
        service = UserService(InMemoryUserRepo())
        user = service.create_user("test@example.com")
        assert user.email == "test@example.com"

    def test_create_user_invalid_email(self):
        service = UserService(InMemoryUserRepo())
        with pytest.raises(ValidationError, match="invalid email"):
            service.create_user("not-an-email")
```

```python
import pytest


class FakeAsyncUserRepo:
    async def get_by_id(self, user_id: int) -> User | None:
        return User(id=user_id, email="test@example.com")


@pytest.mark.asyncio
async def test_fetch_user():
    service = UserService(FakeAsyncUserRepo())
    user = await service.fetch_user(1)
    assert user is not None
    assert user.email == "test@example.com"
```

## Mock System Boundaries

Mock the email gateway, payment provider, remote API, or queue client. Do not patch each internal layer just to prove that one function called the next.

```python
from unittest.mock import patch


class InMemoryUserRepo:
    def save(self, user: User) -> User:
        return user


@patch("app.services.user_service.send_email")
def test_user_registration_sends_welcome_email(mock_send_email):
    mock_send_email.return_value = True
    service = UserService(InMemoryUserRepo())

    user = service.register("test@example.com")

    assert user.email == "test@example.com"
    mock_send_email.assert_called_once()
```

## Prefer In-Memory Databases When Query Behavior Matters

- Use an in-memory database when you need to verify joins, uniqueness, transaction behavior, or query semantics.
- Do not mock the database API if the real risk is in the query, schema, or constraint behavior.
- Keep the schema small and the fixtures focused on the behavior under test.

```python
import pytest
from sqlalchemy.exc import IntegrityError


def test_unique_email_constraint(sqlite_session):
    sqlite_session.add(User(email="a@example.com"))
    sqlite_session.commit()

    sqlite_session.add(User(email="a@example.com"))
    with pytest.raises(IntegrityError):
        sqlite_session.commit()
```

## Prefer Controlled Local Fixtures for Protocol-Heavy Code

- For HTTP, WebSocket, browser-like, filesystem, or queue-heavy code, prefer a tiny local server or fixture over mocking every low-level call.
- Keep fixtures deterministic with fixed status codes, headers, payloads, redirects, and timeouts.
- Put reusable support code under `tests/support/` or `tests/fixtures/`.

## Parity Tests for Dual APIs

- If both sync and async surfaces are public, test one shared behavior matrix first.
- Reuse the same fixtures and expected outputs where possible.
- Add surface-specific edge cases only after mirrored behavior is covered.
- If CI does not run both suites, do not claim parity with confidence.

## Optional Tools

Use these only when the problem justifies them.

| Tool | Use When | Avoid When |
| --- | --- | --- |
| Factory Boy | test data setup is noisy and repetitive | a small hand-written fixture is clearer |
| Hypothesis | you need to probe invariants or hidden edge cases | one or two example cases are enough |
| BDD / `pytest-bdd` | business-facing acceptance flows benefit from executable specs | the team only needs direct code-level tests |

## Coverage Requirements

Coverage is a signal, not the design target. Use it to find hidden bugs, dead code, and redundant branches, not to justify brittle tests or vanity thresholds.

```toml
[tool.coverage.run]
source = ["app"]
branch = true

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "if TYPE_CHECKING:",
    "raise NotImplementedError",
]
```

```bash
pytest --cov=app --cov-report=term-missing
```

If the repository uses a fail-under threshold, treat it as a guardrail for confidence, not as the purpose of the suite.

## Test Organization

Organize tests for confidence and maintainability, not for taxonomy purity.

Application-leaning variant:

```text
examples/
└── quickstart.py

tests/
├── conftest.py
├── unit/
├── integration/
└── e2e/
```

Boundary-heavy or tooling variant:

```text
examples/
├── quickstart.py
└── advanced_flow.py

tests/
├── conftest.py
├── support/
├── fixtures/
├── smoke/
├── feature/
├── integration/
└── release/
```

Pick the shape that improves release confidence without making the suite too slow or too flaky.

## Bug Fix Workflow

1. Write the failing test first.
2. Confirm it fails for the right reason.
3. Fix the bug.
4. Confirm the test passes.
5. Keep the regression test.

```python
def test_issue_123_null_handling():
    """Regression test for issue #123: NullPointerError on empty input."""
    result = process_data(None)
    assert result == []
```
