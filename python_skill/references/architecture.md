# Architecture & Organization

## Repository-Service-Router (RSR) Pattern

```
┌─────────────┐
│   Router    │  HTTP/RPC/CLI entry points
├─────────────┤
│   Service   │  Business logic
├─────────────┤
│ Repository  │  Data access (DB, cache, APIs)
└─────────────┘
```

### Layer Responsibilities

| Layer      | Responsibility                        | Dependencies     |
| ---------- | ------------------------------------- | ---------------- |
| Router     | Request/response handling, validation | Service          |
| Service    | Business logic, orchestration         | Repository       |
| Repository | Data access, queries                  | External systems |


### Module-Based Structure (Recommended)

**高内聚、低耦合**：每个模块自包含完整的 RSR 层级

```
app/
├── main.py                    # Application entry, mount routers
├── config.py                  # Global settings
├── core/                      # Shared infrastructure
│   ├── __init__.py
│   ├── database.py            # DB connection, session
│   ├── exceptions.py          # Base exceptions
│   └── dependencies.py        # Common DI (get_db, get_current_user)
│
├── users/                     # User module (self-contained)
│   ├── __init__.py
│   ├── router.py              # /users endpoints
│   ├── service.py             # User business logic
│   ├── repository.py          # User data access
│   ├── models.py              # User ORM model
│   ├── schemas.py             # UserCreate, UserResponse
│   └── exceptions.py          # UserNotFoundError
│
├── orders/                    # Order module (self-contained)
│   ├── __init__.py
│   ├── router.py
│   ├── service.py
│   ├── repository.py
│   ├── models.py
│   ├── schemas.py
│   └── exceptions.py
│
└── products/                  # Product module
    ├── __init__.py
    ├── router.py
    ├── service.py
    ├── repository.py
    ├── models.py
    └── schemas.py
```

### Module Wiring

```python
# app/main.py
from fastapi import FastAPI
from app.users.router import router as users_router
from app.orders.router import router as orders_router
from app.products.router import router as products_router

app = FastAPI()

app.include_router(users_router, prefix="/users", tags=["users"])
app.include_router(orders_router, prefix="/orders", tags=["orders"])
app.include_router(products_router, prefix="/products", tags=["products"])
```

### Cross-Module Dependencies

```python
# app/orders/service.py
from app.users.repository import UserRepository  # Import from other module

class OrderService:
    def __init__(
        self,
        order_repo: OrderRepository,
        user_repo: UserRepository,  # Cross-module dependency
    ):
        self.order_repo = order_repo
        self.user_repo = user_repo

    async def create_order(self, user_id: int, items: list) -> Order:
        user = await self.user_repo.get_by_id(user_id)
        if user is None:
            raise UserNotFoundError(user_id)
        return await self.order_repo.create(user_id, items)
```

## Import Organization

```python
# 1. Standard library
import asyncio
from pathlib import Path

# 2. Third-party
from fastapi import FastAPI, Depends
from pydantic import BaseModel

# 3. Local modules (absolute imports preferred)
from app.users.service import UserService
from app.users.repository import UserRepository
from app.orders.schemas import OrderCreate
```

**Rules:**
- One import per line (no `import os, sys`)
- Absolute imports preferred over relative
- Imports at module top (except for lazy loading or circular deps)
- Group with blank lines: stdlib → third-party → local

## Dependency Injection

### FastAPI Pattern (Module-Based)

```python
# app/users/repository.py
class UserRepository:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def get_by_id(self, user_id: int) -> User | None:
        return await self.db.get(User, user_id)

# app/users/service.py
from app.users.repository import UserRepository
from app.users.exceptions import UserNotFoundError

class UserService:
    def __init__(self, repo: UserRepository):
        self.repo = repo

    async def get_user(self, user_id: int) -> User:
        user = await self.repo.get_by_id(user_id)
        if user is None:
            raise UserNotFoundError(user_id)
        return user

# app/users/router.py
from fastapi import APIRouter, Depends
from app.core.dependencies import get_db
from app.users.service import UserService
from app.users.repository import UserRepository
from app.users.schemas import UserResponse

router = APIRouter()

def get_user_service(db: AsyncSession = Depends(get_db)) -> UserService:
    repo = UserRepository(db)
    return UserService(repo)

@router.get("/{user_id}")
async def get_user(
    user_id: int,
    service: UserService = Depends(get_user_service)
) -> UserResponse:
    user = await service.get_user(user_id)
    return UserResponse.model_validate(user)
```

### Dependency Inversion

```python
# Define interface (Protocol)
from typing import Protocol

class UserRepositoryProtocol(Protocol):
    async def get_by_id(self, user_id: int) -> User | None: ...
    async def save(self, user: User) -> User: ...

# Service depends on abstraction
class UserService:
    def __init__(self, repo: UserRepositoryProtocol):
        self.repo = repo

# Concrete implementation
class PostgresUserRepository:
    async def get_by_id(self, user_id: int) -> User | None: ...
    async def save(self, user: User) -> User: ...

# Easy to swap implementations for testing
class InMemoryUserRepository:
    def __init__(self):
        self.users: dict[int, User] = {}

    async def get_by_id(self, user_id: int) -> User | None:
        return self.users.get(user_id)
```

## Configuration Management

```python
from pydantic_settings import BaseSettings
from pydantic import Field

class Settings(BaseSettings):
    # Required
    database_url: str
    secret_key: str

    # With defaults
    debug: bool = False
    log_level: str = "INFO"

    # Nested/computed
    redis_url: str = "redis://localhost:6379"

    model_config = {
        "env_prefix": "APP_",
        "env_file": ".env",
        "env_file_encoding": "utf-8",
    }

# Usage
settings = Settings()
```

## Error Handling Strategy

```python
# app/core/exceptions.py - Base exceptions
class DomainError(Exception):
    """Base for domain errors."""

class ValidationError(DomainError):
    def __init__(self, field: str, message: str):
        self.field = field
        super().__init__(f"{field}: {message}")

# app/users/exceptions.py - Module-specific exceptions
from app.core.exceptions import DomainError

class UserNotFoundError(DomainError):
    def __init__(self, user_id: int):
        self.user_id = user_id
        super().__init__(f"User {user_id} not found")

# app/main.py - Handle at app boundary
from app.users.exceptions import UserNotFoundError

@app.exception_handler(UserNotFoundError)
async def user_not_found_handler(request: Request, exc: UserNotFoundError):
    return JSONResponse(
        status_code=404,
        content={"error": "user_not_found", "user_id": exc.user_id}
    )
```

## Logging Strategy

```python
from loguru import logger
import sys

# Configure once at startup
def setup_logging(level: str = "INFO"):
    logger.remove()
    logger.add(
        sys.stderr,
        level=level,
        format="<green>{time:YYYY-MM-DD HH:mm:ss}</green> | <level>{level: <8}</level> | <cyan>{name}</cyan>:<cyan>{function}</cyan> | <level>{message}</level>",
    )

# Usage in code
logger.info("Processing order {}", order_id)
logger.bind(user_id=user_id, action="login").info("User logged in")
```
