# Error Handling

> References:
> - https://frostming.com/posts/2023/error-handling/
> - https://blog.yanli.one/ideas-about-exception-catch
> - https://blog.miguelgrinberg.com/post/the-ultimate-guide-to-error-handling-in-python

## Goals

- Make failure paths explicit and understandable
- Catch exceptions only where you can recover
- Preserve error context for debugging

## Core Principles

| Principle | Description |
|-----------|-------------|
| Catch specific | Never bare `except:`, always specify exception type |
| Preserve context | Use `raise ... from e` to chain exceptions |
| Fail fast | Don't catch if you can't handle |
| EAFP over LBYL | "Easier to Ask Forgiveness than Permission" |

## EAFP vs LBYL

```python
# LBYL (Look Before You Leap) - 多次检查
if key in dictionary:
    if dictionary[key] is not None:
        value = dictionary[key]
        process(value)

# EAFP (Easier to Ask Forgiveness) - 更 Pythonic ✅
try:
    value = dictionary[key]
    process(value)
except KeyError:
    pass  # or handle missing key
```

**When to prefer LBYL:**
- Check is cheap and failure is common
- Side effects on failure are problematic

## Exception Chaining

```python
# ✅ Preserve original context with `from`
try:
    data = json.loads(raw_data)
except json.JSONDecodeError as e:
    raise ValidationError(f"Invalid JSON in config") from e

# ✅ Suppress chain when irrelevant (rare)
try:
    value = cache[key]
except KeyError:
    raise ConfigNotFoundError(key) from None

# ❌ Never lose context
try:
    do_something()
except SomeError:
    raise OtherError("failed")  # Original traceback lost!
```

## Batch vs Pipeline Error Handling

### Batch: Isolate Per-Item Failures

```python
results = []
errors = []

for item in items:
    try:
        result = process(item)
        results.append(result)
    except ProcessingError as e:
        errors.append((item, e))
        logger.warning("Failed to process {}: {}", item.id, e)

if errors:
    logger.error("Failed {} of {} items", len(errors), len(items))
```

### Pipeline: Fail Fast

```python
def pipeline(data: RawData) -> FinalResult:
    # Each step can fail, stops pipeline
    validated = validate(data)      # May raise ValidationError
    transformed = transform(validated)  # May raise TransformError
    enriched = enrich(transformed)  # May raise EnrichmentError
    return finalize(enriched)
```

## Exception Design

### Module-Specific Exceptions

```python
# app/core/exceptions.py
class AppError(Exception):
    """Base for all app exceptions."""

class NotFoundError(AppError):
    """Resource not found."""
    def __init__(self, resource: str, id: Any):
        self.resource = resource
        self.id = id
        super().__init__(f"{resource} {id} not found")

class ValidationError(AppError):
    """Input validation failed."""
    def __init__(self, field: str, message: str):
        self.field = field
        self.message = message
        super().__init__(f"{field}: {message}")

# app/users/exceptions.py
from app.core.exceptions import NotFoundError

class UserNotFoundError(NotFoundError):
    def __init__(self, user_id: int):
        super().__init__("User", user_id)
        self.user_id = user_id
```

### Exception with Structured Data

```python
from dataclasses import dataclass

@dataclass
class ErrorDetail:
    code: str
    message: str
    field: str | None = None

class APIError(Exception):
    def __init__(self, status_code: int, details: list[ErrorDetail]):
        self.status_code = status_code
        self.details = details
        super().__init__(f"API Error {status_code}: {details}")

# Usage
raise APIError(400, [
    ErrorDetail("invalid_email", "Email format is invalid", "email"),
    ErrorDetail("required", "This field is required", "name"),
])
```

## Context Managers for Cleanup

```python
from contextlib import contextmanager

@contextmanager
def managed_resource(name: str):
    resource = acquire_resource(name)
    try:
        yield resource
    except Exception:
        logger.error("Error while using {}", name)
        raise  # Re-raise after logging
    finally:
        release_resource(resource)  # Always cleanup

# Usage
with managed_resource("db_connection") as conn:
    conn.execute(query)
```

## Async Exception Handling

```python
async def fetch_with_retry(url: str, retries: int = 3) -> dict:
    last_error: Exception | None = None

    for attempt in range(retries):
        try:
            async with asyncio.timeout(10):
                return await fetch(url)
        except TimeoutError:
            last_error = TimeoutError(f"Timeout fetching {url}")
            logger.warning("Attempt {} timed out", attempt + 1)
        except aiohttp.ClientError as e:
            last_error = e
            logger.warning("Attempt {} failed: {}", attempt + 1, e)

        if attempt < retries - 1:
            await asyncio.sleep(2 ** attempt)  # Exponential backoff

    raise FetchError(f"Failed after {retries} attempts") from last_error
```

## Anti-Patterns

```python
# ❌ Bare except
try:
    do_something()
except:  # Catches SystemExit, KeyboardInterrupt too!
    pass

# ❌ Catching too broad
try:
    value = int(user_input)
except Exception:  # Hides bugs
    value = 0

# ❌ Silent failures
try:
    important_operation()
except SomeError:
    pass  # Error swallowed, debugging nightmare

# ❌ Redundant exception handling
try:
    do_something()
except SomeError as e:
    raise e  # Just let it propagate naturally

# ✅ Correct patterns
try:
    value = int(user_input)
except ValueError:
    logger.warning("Invalid input: {}", user_input)
    value = default_value
```
