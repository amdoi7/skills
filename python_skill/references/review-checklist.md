# Code Review Checklist

## Quick Review Questions

Ask these questions when reviewing Python code:

### Design & Structure

- [ ] Does naming reflect domain intent?
- [ ] Is the module split driven by bounded context or capability rather than framework layer names?
- [ ] Are there unnecessary cross-context or cross-layer dependencies?
- [ ] Is there a clearer data structure choice?
- [ ] Is there over-abstraction or hidden magic?
- [ ] Is the entry point obvious for users?
- [ ] Are changes localized and reversible?
- [ ] Are schemas/DTOs used only at boundaries instead of mirroring the domain everywhere?
- [ ] Are invariants kept in the model while changeable policies stay outside?

### Error Handling

- [ ] Does error handling preserve context?
- [ ] Are only specific exceptions caught (no bare `except:`)?
- [ ] Are exceptions re-raised with `from` to preserve chain?
- [ ] Are invariant violations, policy rejections, and infrastructure failures distinguished clearly?

### Code Quality

- [ ] Do new features require edits in too many places?
- [ ] Is there code duplication that should be extracted?
- [ ] Are functions/methods under 50 lines?
- [ ] Is nesting depth ≤ 3 levels?

### Python-Specific

- [ ] Are type hints present and correct?
- [ ] Is `is` vs `==` used correctly?
- [ ] No mutable default arguments?
- [ ] F-strings avoided in logging?
- [ ] Using stdlib before external deps?

## Code Smells & Refactoring

### Growing If-Else Chain

**Problem:**
```python
def get_news(source: str | None = None) -> Iterable[News]:
    if source is None:
        return chain(HNSource().iter_news(), V2Source().iter_news())
    if source == "HN":
        return HNSource().iter_news()
    elif source == "V2":
        return V2Source().iter_news()
    elif source == "Reddit":
        return RedditSource().iter_news()
    else:
        raise ValueError(f"Unknown source: {source}")
```

**Solution: Registry pattern**
```python
SOURCES: dict[str, type[NewsSource]] = {
    "HN": HNSource,
    "V2": V2Source,
    "Reddit": RedditSource,
}

def get_news(source: str | None = None) -> Iterable[News]:
    if source is None:
        return chain(*(s().iter_news() for s in SOURCES.values()))
    if source not in SOURCES:
        raise ValueError(f"Unknown source: {source}")
    return SOURCES[source]().iter_news()
```

### Constructor Mode Switches

**Problem:**
```python
class DataLoader:
    def __init__(self, source: str, **kwargs):
        if source == "file":
            self.path = kwargs["path"]
            self.data = self._load_file()
        elif source == "db":
            self.conn = kwargs["connection"]
            self.data = self._load_db()
        elif source == "api":
            self.url = kwargs["url"]
            self.data = self._load_api()
```

**Solution: Factory + separate classes**
```python
class DataLoader(Protocol):
    def load(self) -> Data: ...

class FileLoader:
    def __init__(self, path: Path):
        self.path = path

    def load(self) -> Data:
        return self._load_file()

class DatabaseLoader:
    def __init__(self, connection: Connection):
        self.connection = connection

    def load(self) -> Data:
        return self._load_db()

def create_loader(source: str, **kwargs) -> DataLoader:
    loaders = {"file": FileLoader, "db": DatabaseLoader, "api": ApiLoader}
    return loaders[source](**kwargs)
```

### Deep Nesting

**Problem:**
```python
def process(data):
    if data:
        if data.is_valid():
            if data.has_items():
                for item in data.items:
                    if item.is_active:
                        process_item(item)
```

**Solution: Early returns and extraction**
```python
def process(data: Data | None) -> None:
    if not data or not data.is_valid() or not data.has_items():
        return

    for item in data.items:
        process_if_active(item)

def process_if_active(item: Item) -> None:
    if not item.is_active:
        return
    process_item(item)
```

### God Class / Fat Service

**Problem:** Single class with 500+ lines handling multiple concerns.

**Solution:**
1. Identify distinct responsibilities
2. Separate domain rules, orchestration, and adapters
3. Compose via dependency injection or explicit wiring

### Stringly-Typed Code

**Problem:**
```python
def set_status(status: str): ...
set_status("actve")  # Typo goes unnoticed
```

**Solution:**
```python
from enum import StrEnum

class Status(StrEnum):
    ACTIVE = "active"
    INACTIVE = "inactive"
    PENDING = "pending"

def set_status(status: Status): ...
set_status(Status.ACTIVE)  # Type-safe, also works as string
```

## Review Response Template

```markdown
## Summary
[Brief overview of what the PR does]

## Strengths
- [What's done well]

## Suggestions
- [ ] [Actionable suggestion with line reference]
- [ ] [Another suggestion]

## Questions
- [Clarifying questions about design decisions]

## Nitpicks (Optional)
- [Minor style suggestions]
```
