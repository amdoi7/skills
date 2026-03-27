# Testing

## Test Pyramid

```
        /\
       /  \     E2E (few)
      /----\
     /      \   Integration (some)
    /--------\
   /          \ Unit (many)
  --------------
```

| Level | Scope | Speed | Isolation |
|-------|-------|-------|-----------|
| Unit | Single function/class | Fast | Full mocking |
| Integration | Module interactions | Medium | Real deps where practical |
| E2E | Full system | Slow | Production-like |

## Pytest Basics

```python
# test_user_service.py
import pytest
from app.services.user_service import UserService

class TestUserService:
    def test_create_user_success(self):
        service = UserService(mock_repo)
        user = service.create_user("test@example.com")
        assert user.email == "test@example.com"

    def test_create_user_invalid_email(self):
        service = UserService(mock_repo)
        with pytest.raises(ValidationError, match="invalid email"):
            service.create_user("not-an-email")

    @pytest.mark.parametrize("email,expected", [
        ("user@example.com", True),
        ("user@sub.example.com", True),
        ("invalid", False),
        ("", False),
    ])
    def test_validate_email(self, email: str, expected: bool):
        assert validate_email(email) == expected
```

## Async Testing

```python
import pytest

# Mark async tests
@pytest.mark.asyncio
async def test_fetch_user():
    service = UserService(mock_repo)
    user = await service.fetch_user(1)
    assert user.id == 1

# Async fixtures
@pytest.fixture
async def db_session():
    async with async_session() as session:
        yield session
        await session.rollback()

@pytest.mark.asyncio
async def test_with_db(db_session):
    repo = UserRepository(db_session)
    user = await repo.get_by_id(1)
    assert user is not None
```

## Fixtures

```python
import pytest
from unittest.mock import AsyncMock, MagicMock

@pytest.fixture
def mock_user_repo():
    repo = MagicMock()
    repo.get_by_id = AsyncMock(return_value=User(id=1, email="test@example.com"))
    repo.save = AsyncMock(side_effect=lambda u: u)
    return repo

@pytest.fixture
def user_service(mock_user_repo):
    return UserService(mock_user_repo)

def test_get_user(user_service, mock_user_repo):
    user = user_service.get_user(1)
    mock_user_repo.get_by_id.assert_called_once_with(1)
    assert user.id == 1
```

## Mocking

```python
from unittest.mock import patch, MagicMock, AsyncMock

# Patch external dependencies
@patch("app.services.user_service.send_email")
def test_user_registration(mock_send_email):
    mock_send_email.return_value = True
    service = UserService(mock_repo)
    service.register("test@example.com")
    mock_send_email.assert_called_once()

# Async mock
@patch("app.services.user_service.fetch_external_data")
@pytest.mark.asyncio
async def test_with_external_api(mock_fetch):
    mock_fetch.return_value = {"status": "ok"}
    result = await process_data()
    assert result.status == "ok"

# Context manager style
def test_with_context():
    with patch.object(UserRepository, "get_by_id") as mock_get:
        mock_get.return_value = User(id=1)
        user = service.get_user(1)
        assert user.id == 1
```

## Factory Boy for Test Data

```python
import factory
from factory import fuzzy

class UserFactory(factory.Factory):
    class Meta:
        model = User

    id = factory.Sequence(lambda n: n)
    email = factory.LazyAttribute(lambda o: f"user{o.id}@example.com")
    name = factory.Faker("name")
    is_active = True
    created_at = factory.LazyFunction(datetime.utcnow)

# Usage
def test_user_activation():
    user = UserFactory(is_active=False)
    user.activate()
    assert user.is_active

def test_bulk_users():
    users = UserFactory.create_batch(10)
    assert len(users) == 10
```

## Hypothesis for Property-Based Testing

```python
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_sort_preserves_length(items):
    sorted_items = sorted(items)
    assert len(sorted_items) == len(items)

@given(st.text(min_size=1))
def test_email_validation_no_crash(text):
    # Should never crash, regardless of input
    result = validate_email(text)
    assert isinstance(result, bool)

@given(
    st.dictionaries(
        keys=st.text(min_size=1, max_size=10),
        values=st.integers(),
        min_size=1
    )
)
def test_dict_operations(d):
    keys = list(d.keys())
    for key in keys:
        assert key in d
```

## BDD (Behavior-Driven Development)

BDD 将测试用例写成自然语言规范，提升业务人员与开发者的沟通效率。

### pytest-bdd 基础

```bash
uv add --group dev pytest-bdd
```

```gherkin
# tests/features/user_registration.feature
Feature: User Registration
    As a new user
    I want to register an account
    So that I can access the system

    Scenario: Successful registration
        Given a valid email "test@example.com"
        And a valid password "SecurePass123!"
        When I register the user
        Then the user should be created
        And a welcome email should be sent

    Scenario Outline: Invalid email formats
        Given an invalid email "<email>"
        When I try to register
        Then registration should fail with "invalid email"

        Examples:
            | email           |
            | not-an-email    |
            | missing@domain  |
            |                 |
```

### Step Definitions

```python
# tests/step_defs/test_user_registration.py
import pytest
from pytest_bdd import scenarios, given, when, then, parsers

# Link to feature file
scenarios("../features/user_registration.feature")

@pytest.fixture
def context():
    """Shared state between steps."""
    return {}

@given(parsers.parse('a valid email "{email}"'))
def valid_email(context, email):
    context["email"] = email

@given(parsers.parse('a valid password "{password}"'))
def valid_password(context, password):
    context["password"] = password

@given(parsers.parse('an invalid email "{email}"'))
def invalid_email(context, email):
    context["email"] = email

@when("I register the user")
def register_user(context, user_service):
    context["user"] = user_service.register(
        context["email"], context["password"]
    )

@when("I try to register")
def try_register(context, user_service):
    try:
        user_service.register(context["email"], "password")
        context["error"] = None
    except ValidationError as e:
        context["error"] = str(e)

@then("the user should be created")
def user_created(context):
    assert context["user"] is not None
    assert context["user"].email == context["email"]

@then("a welcome email should be sent")
def welcome_email_sent(context, mock_email):
    mock_email.send_welcome.assert_called_once()

@then(parsers.parse('registration should fail with "{message}"'))
def registration_failed(context, message):
    assert context["error"] is not None
    assert message in context["error"]
```

### BDD vs TDD

| 维度 | TDD | BDD |
|------|-----|-----|
| 焦点 | 代码实现细节 | 业务行为 |
| 语言 | 编程语言 | 自然语言 (Gherkin) |
| 受众 | 开发者 | 开发者 + 业务人员 |
| 文档 | 测试代码即文档 | Feature 文件即规范 |
| 适用 | 单元测试、技术逻辑 | 验收测试、业务流程 |

### 项目结构

```
tests/
├── conftest.py
├── features/              # Gherkin feature files
│   ├── user_registration.feature
│   └── order_checkout.feature
├── step_defs/             # Step implementations
│   ├── conftest.py        # Shared fixtures
│   ├── test_user_steps.py
│   └── test_order_steps.py
└── unit/                  # Traditional unit tests
```

### Async BDD Steps

```python
from pytest_bdd import given, when, then
import pytest

@when("I fetch user data asynchronously")
@pytest.mark.asyncio
async def fetch_user_async(context, async_service):
    context["user"] = await async_service.fetch_user(context["user_id"])
```

## Coverage Requirements

```toml
# pyproject.toml
[tool.coverage.run]
source = ["app"]
branch = true

[tool.coverage.report]
fail_under = 90
exclude_lines = [
    "pragma: no cover",
    "if TYPE_CHECKING:",
    "raise NotImplementedError",
]
```

```bash
# Run with coverage
pytest --cov=app --cov-report=term-missing --cov-fail-under=90
```

## Test Organization

```
tests/
├── conftest.py          # Shared fixtures
├── unit/
│   ├── test_services.py
│   └── test_models.py
├── integration/
│   ├── test_api.py
│   └── test_database.py
└── e2e/
    └── test_workflows.py
```

## Bug Fix Workflow

1. **Write failing test first**
2. Fix the bug
3. Verify test passes
4. Add regression test to prevent recurrence

```python
def test_issue_123_null_handling():
    """Regression test for issue #123: NullPointerError on empty input."""
    result = process_data(None)
    assert result == []  # Should handle None gracefully
```
