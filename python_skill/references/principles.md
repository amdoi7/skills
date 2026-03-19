# Core Principles

## Goals

- Newcomers understand domain intent without hidden context
- Future maintainers can find business rules before framework glue
- Implementation explains why it is written this way

## Design Principles

### Domain Language First

- Prefer names from the business domain over framework words like `handler`, `manager`, or `util`.
- If something is hard to name, the design may still be muddy.
- Keep terms consistent across code, tests, and discussions.

### Contexts Before Layers

- Split code by capability or bounded context before adding technical layers.
- A package should answer "which part of the business lives here?" before "which framework role lives here?"
- Add `service`, `repository`, or `schema` modules only when they hide real complexity.

### Types Carry Meaning

- Prefer `StrEnum`, `dataclass`, small value objects, and precise aliases over raw `str` and `dict`.
- Make illegal states harder to represent.
- Validate and normalize at the boundary, then pass trusted objects inward.

### Invariants vs Policies

- Put invariants in constructors, value objects, and domain methods.
- Keep changeable business rules in explicit policy objects or application orchestration.
- Do not bury policy branches inside generic helpers.

### Python Zen in Service of the Model

- Explicit is better than implicit.
- Flat is better than nested.
- Readability counts.
- Practicality beats purity, but not at the cost of obscuring business meaning.

## Code Style

### Naming

```python
# Classes: PascalCase
class UserRepository: ...

# Functions/variables: snake_case
def get_active_users() -> list[User]: ...

# Constants: UPPER_SNAKE_CASE
MAX_RETRY_COUNT = 3

# Private: single underscore prefix
def _validate_input(data: dict) -> bool: ...
```

### Function Design

```python
# Good: the function name says what business step happens
def place_order(command: PlaceOrderCommand) -> Order:
    order = Order.create(command.customer_id, command.items)
    credit_policy.check(order.total, command.credit_limit)
    return repository.save(order)

# Bad: mixed transport, policy, and infrastructure concerns
def handle_order(order, user, inventory, payment_gateway, emailer): ...
```

### Control Flow

```python
# Good: early return, flat structure
def get_discount(user: User) -> Decimal:
    if not user.is_active:
        return Decimal("0")
    if user.is_premium:
        return Decimal("0.2")
    if user.orders_count > 10:
        return Decimal("0.1")
    return Decimal("0.05")

# Bad: deep nesting
def get_discount(user):
    if user.is_active:
        if user.is_premium:
            return 0.2
        else:
            if user.orders_count > 10:
                return 0.1
            else:
                return 0.05
    else:
        return 0
```

### Dispatch Patterns: Dict vs Match vs If/Elif

| 方式 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| Dict mapping | 简单值→函数映射 | 可动态扩展、O(1) 查找 | 不支持复杂模式 |
| `match` | 结构解构、类型匹配 | 可读性强、支持 guard | 静态、3.10+ |
| if/elif | 复杂条件组合 | 灵活 | 冗长、易遗漏 |

```python
# ✅ Dict mapping: 简单字符串→函数分发
HANDLERS = {
    "create": handle_create,
    "update": handle_update,
    "delete": handle_delete,
}

def dispatch(action: str, data: dict) -> Result:
    if (handler := HANDLERS.get(action)) is None:
        raise ValueError(f"Unknown action: {action}")
    return handler(data)

# ✅ Match: 结构解构、类型匹配 (Python 3.10+)
def handle_event(event: dict) -> str:
    match event:
        case {"type": "click", "x": x, "y": y}:
            return f"Clicked at ({x}, {y})"
        case {"type": "keypress", "key": key} if key.isalpha():
            return f"Pressed letter: {key}"
        case {"type": "keypress", "key": key}:
            return f"Pressed: {key}"
        case _:
            return "Unknown event"

# ✅ Match: 类型分发
def process(value: int | str | list) -> str:
    match value:
        case int(n) if n > 0:
            return f"Positive: {n}"
        case int(n):
            return f"Non-positive: {n}"
        case str(s):
            return f"String: {s}"
        case [first, *rest]:
            return f"List starting with {first}"

# ❌ Avoid: long if/elif chain
def dispatch_bad(action, data):
    if action == "create":
        return handle_create(data)
    elif action == "update":
        return handle_update(data)
    elif action == "delete":
        return handle_delete(data)
    else:
        raise ValueError(f"Unknown action: {action}")
```

## Rob Pike's 5 Rules

1. **Measure first**: Bottlenecks are unpredictable; profile before optimizing
2. **Optimize only when significant**: Don't optimize non-bottlenecks
3. **Simple algorithms for small n**: Fancy algorithms have high constants
4. **Simple over complex**: Complex algorithms are bug magnets
5. **Data structures matter most**: Right data structure > clever algorithm
