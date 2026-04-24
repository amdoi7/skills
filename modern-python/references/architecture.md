# Context-First Architecture

## Goals

- Make business rules easy to find.
- Keep framework, transport, and storage details at the edges.
- Let one capability evolve without dragging unrelated modules with it.

## Default Decomposition

Start by splitting the system into **bounded contexts or capabilities**, not a global `routers/`, `services/`, `repositories/` tree.

```text
app/
├── main.py
├── shared/
│   ├── config.py
│   ├── db.py
│   └── errors.py
├── ordering/
│   ├── api.py
│   ├── application.py
│   ├── domain.py
│   ├── ports.py
│   └── adapters/
│       ├── postgres.py
│       └── events.py
└── billing/
    ├── api.py
    ├── application.py
    ├── domain.py
    ├── ports.py
    └── adapters/
```

`api.py`, `application.py`, `domain.py`, `ports.py`, and `adapters/` are **options**, not mandatory layers. Add only the pieces that hide real complexity.

## Responsibilities

| Part | Role | Should Know About |
| --- | --- | --- |
| `api.py` | HTTP, CLI, RPC payload parsing and response formatting | Schemas, auth/session plumbing, application use cases |
| `application.py` | Use-case orchestration and transactions | Domain objects, ports, policy selection |
| `domain.py` | Entities, value objects, invariants, domain services | Domain language only |
| `ports.py` | Interfaces to other contexts or infrastructure | Stable domain-facing contracts |
| `adapters/` | DB, HTTP clients, queues, files | Frameworks, SQL, SDKs, transport details |
| `shared/` | Cross-cutting technical primitives | Logging, config, DB session setup, base errors |

## When a Simpler Slice Is Better

For a small CRUD surface or a short-lived script, a single `orders.py` or a tiny `orders/` package is often enough. Do not create `api/service/repository` layers unless they pull complexity downward.

## Cross-Context Collaboration

Depend on a **port or facade** exported by the other context, not on its repository or ORM model.

```python
from typing import Protocol


class CustomerCreditPort(Protocol):
    async def current_limit(self, customer_id: int) -> int: ...


class PlaceOrder:
    def __init__(
        self,
        repo: OrderRepository,
        customer_credit: CustomerCreditPort,
    ) -> None:
        self.repo = repo
        self.customer_credit = customer_credit
```

Avoid this:

```python
from app.customers.repository import CustomerRepository
```

That imports another context's storage choices instead of its domain contract.

## Boundary Schemas and DTOs

- Keep Pydantic schemas and transport DTOs at API or integration boundaries.
- Convert near the boundary.
- Do not create one DTO per domain object unless the external contract actually differs.
- If the boundary and the domain match, reuse the domain type or expose a small read model instead of building a mirror object graph.

```python
from pydantic import BaseModel


class PlaceOrderRequest(BaseModel):
    customer_id: int
    items: list[str]


@router.post("/orders")
async def place_order(
    request: PlaceOrderRequest,
    use_case: PlaceOrder = Depends(get_place_order),
) -> OrderResponse:
    order = await use_case.execute(
        customer_id=request.customer_id,
        item_codes=request.items,
    )
    return OrderResponse.from_domain(order)
```

## FastAPI Wiring

Keep framework wiring close to the boundary. Build application objects from ports and adapters there.

```python
def get_place_order(db: AsyncSession = Depends(get_db)) -> PlaceOrder:
    repo = SqlOrderRepository(db)
    customer_credit = HttpCustomerCreditClient()
    return PlaceOrder(repo=repo, customer_credit=customer_credit)
```

## Dependency Injection Guidance

- Depend on ports, protocols, or small concrete collaborators when the boundary is local and stable.
- Do not create protocols for every class by default.
- Add an interface when it isolates another context, storage mechanism, or external system.
- Testing convenience alone is not enough reason to add a new abstraction layer if a fake concrete object would be simpler.

## Public Facade and Escape Hatch

Give users a short common path first, then expose deeper control explicitly.

- Export a small stable set of high-frequency entry points from the package boundary.
- Keep advanced control behind config objects, builders, or explicit constructors.
- Do not make users import from internal modules for common tasks.
- Treat the quickstart path as an API test: if the shortest runnable example is awkward, the public surface probably needs work.

```python
from payments import connect, ClientConfig

client = connect()

config = ClientConfig(timeout=5, retries=2)
client = connect(config)
```

## Error Taxonomy

Use a single hierarchy across the skill:

```python
class AppError(Exception):
    """Base class for skill examples."""


class DomainError(AppError):
    """Domain-level failure."""


class InvariantViolation(DomainError):
    """State or input makes the model invalid."""


class PolicyViolation(DomainError):
    """Expected business rejection from a changeable rule."""


class NotFoundError(DomainError):
    """Required entity or relation is missing."""


class InfrastructureError(AppError):
    """Database, network, queue, or file boundary failure."""
```

- Use `InvariantViolation` for "this state should not exist."
- Use `PolicyViolation` for business rules that may change.
- Use `InfrastructureError` when external systems fail.
- Translate all of them at the boundary instead of leaking framework-specific exceptions upward.

## Logging Strategy

Prefer `loguru` for application logging unless the repository has already standardized another logger. Keep one logging stack across entry points, jobs, and workers.

```python
from loguru import logger
import sys


def setup_logging(level: str = "INFO") -> None:
    logger.remove()
    logger.add(sys.stderr, level=level)

logger.info("processing order {}", order_id)
```
