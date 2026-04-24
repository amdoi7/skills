# DevOps & Production

## Dependency Management with uv

```bash
# Create new project
uv init my-project
cd my-project

# Add dependencies
uv add fastapi uvicorn

# Add dev dependencies (using dependency groups)
uv add --group dev pytest pytest-asyncio ruff

# Add other groups
uv add --group test pytest-cov hypothesis
uv add --group docs mkdocs mkdocs-material

# Lock dependencies
uv lock

# Sync environment (all groups)
uv sync

# Sync without dev group
uv sync --no-group dev

# Build package
uv build

# Run scripts
uv run python main.py
uv run pytest
```

```toml
# pyproject.toml
[project]
name = "my-project"
version = "0.1.0"
requires-python = ">=3.13"
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.32.0",
]

# PEP 735: Dependency Groups (replaces optional-dependencies for dev)
[dependency-groups]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.24.0",
    "ruff>=0.8.0",
]
test = [
    "pytest-cov>=6.0.0",
    "hypothesis>=6.0.0",
]
docs = [
    "mkdocs>=1.6.0",
]
```

## Ruff Configuration

```toml
# pyproject.toml
[tool.ruff]
target-version = "py313"
line-length = 88

[tool.ruff.lint] # 选择合适的 而不是全选 select = ["E", "F", "I", "UP", "B", "SIM"]
select = [
    "E",    # pycodestyle errors
    "F",    # pyflakes
    "I",    # isort
    "UP",   # pyupgrade
    "B",    # flake8-bugbear
    "SIM",  # flake8-simplify
    "S",    # flake8-bandit (security)
    "A",    # flake8-builtins
    "C4",   # flake8-comprehensions
    "DTZ",  # flake8-datetimez
    "T10",  # flake8-debugger
    "ICN",  # flake8-import-conventions
    "PIE",  # flake8-pie
    "PT",   # flake8-pytest-style
    "RSE",  # flake8-raise
    "RET",  # flake8-return
    "ARG",  # flake8-unused-arguments
    "PL",   # pylint
    "RUF",  # ruff-specific
]
ignore = [
    "S101",  # assert usage (ok in tests)
]

[tool.ruff.lint.per-file-ignores]
"tests/**/*.py" = ["S101", "ARG001"]

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
```

```bash
# Format code
ruff format .

# Check and fix
ruff check --fix .

# Check only (CI)
ruff check .
ruff format --check .
```

## Pre-commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.2.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: local
    hooks:
      - id: pytest
        name: pytest
        entry: uv run pytest tests/unit -x
        language: system
        pass_filenames: false
        always_run: true
```

```bash
# Install hooks
pre-commit install

# Run manually
pre-commit run --all-files
```

## Docker Multi-Stage Build

```dockerfile
# Dockerfile
FROM python:3.13-slim AS builder

# Install uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

WORKDIR /app

# Copy dependency files
COPY pyproject.toml uv.lock ./

# Install dependencies (exclude dev group)
RUN uv sync --frozen --no-group dev

# Production stage
FROM python:3.13-slim

WORKDIR /app

# Copy virtual environment from builder
COPY --from=builder /app/.venv /app/.venv

# Copy application code
COPY ./app ./app

# Set PATH to use virtual environment
ENV PATH="/app/.venv/bin:$PATH"

# Run as non-root user
RUN useradd --create-home appuser
USER appuser

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports:
      - "8000:8000"
    environment:
      - APP_DATABASE_URL=postgresql://user:pass@db:5432/app
      - APP_SECRET_KEY=${SECRET_KEY}
    depends_on:
      - db
      - redis

  db:
    image: postgres:16-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=app

  redis:
    image: redis:7-alpine

volumes:
  postgres_data:
```

## Deployment with Uvicorn

```bash
# Development
uvicorn app.main:app --reload --port 8000

# Production (single process)
uvicorn app.main:app --host 0.0.0.0 --port 8000

# Production (multiple workers)
uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4
```

## Release Gate and Compatibility Matrix

- Define a smallest release gate: the few commands or flows that would block a release if they broke.
- Keep the release gate smaller than the full suite. It should be fast enough to trust and stable enough to keep.
- Separate fast CI baseline from slower or environment-heavy validation.
- Build the compatibility matrix from actual support promises, not aspirational combinations.
- If sync and async APIs are both public, run both in CI before claiming parity.

```yaml
strategy:
  matrix:
    python-version: ["3.11", "3.12"]  # replace with supported versions
    os: [ubuntu-latest]
```

## Observability

### Logging Conventions

Prefer `loguru` for application logging unless the repository has already standardized another logger. Keep sinks, formats, and field names consistent across services and jobs.

```python
from loguru import logger
import os
import sys


def setup_logging() -> None:
    logger.remove()

    if os.getenv("ENV") == "production":
        logger.add(sys.stdout, format="{message}", serialize=True)
    else:
        logger.add(sys.stderr)

logger.info("service started")
```

### Health Checks

```python
from fastapi import FastAPI, Response

app = FastAPI()

@app.get("/health")
async def health_check():
    return {"status": "healthy"}

@app.get("/health/ready")
async def readiness_check(db: AsyncSession = Depends(get_db)):
    try:
        await db.execute(text("SELECT 1"))
        return {"status": "ready"}
    except Exception:
        return Response(status_code=503, content='{"status": "not ready"}')
```

### OpenTelemetry (Basic Setup)

```python
from opentelemetry import trace
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

# Instrument FastAPI
FastAPIInstrumentor.instrument_app(app)

# Manual spans
tracer = trace.get_tracer(__name__)

async def process_order(order_id: int):
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order_id", order_id)
        # ... processing logic
```

## Security Checklist

- [ ] API input validated via Pydantic
- [ ] Ruff + bandit security checks in CI
- [ ] Dependencies audited (`uv pip check`, `pip-audit`)
- [ ] Secrets in environment variables, not code
- [ ] CORS configured appropriately
- [ ] Rate limiting enabled
- [ ] JWT tokens with proper expiry
- [ ] SQL queries parameterized (no string formatting)
