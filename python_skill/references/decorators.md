# Decorators (装饰器)

> Detailed decorator reference for this skill.

> "Decorators are wrappers that enhance functions without modifying their code."

装饰器是 Python 元编程的基础，本质是**高阶函数 + 闭包**。

## 基础模式

### 函数装饰器

```python
import functools
from typing import Callable, TypeVar, ParamSpec

P = ParamSpec("P")
R = TypeVar("R")

def decorator(func: Callable[P, R]) -> Callable[P, R]:
    @functools.wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        result = func(*args, **kwargs)
        return result
    return wrapper

@decorator
def my_function(): ...
```

### 带参数装饰器

```python
import asyncio
import functools

def retry(max_attempts: int = 3, delay: float = 1.0):
    def decorator(func: Callable[P, R]) -> Callable[P, R]:
        @functools.wraps(func)
        async def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            last_error: Exception | None = None
            for attempt in range(max_attempts):
                try:
                    return await func(*args, **kwargs)
                except Exception as e:
                    last_error = e
                    if attempt < max_attempts - 1:
                        await asyncio.sleep(delay * (2 ** attempt))
            raise last_error
        return wrapper
    return decorator

@retry(max_attempts=3, delay=0.5)
async def fetch_data(url: str) -> dict: ...
```

### 类装饰器

```python
import functools

def singleton(cls):
    instances = {}

    @functools.wraps(cls)
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]

    return get_instance
```

## 实用装饰器

### 计时装饰器

```python
import time
from loguru import logger

def timed(func: Callable[P, R]) -> Callable[P, R]:
    @functools.wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        start = time.perf_counter()
        try:
            return func(*args, **kwargs)
        finally:
            elapsed = time.perf_counter() - start
            logger.debug("{} took {:.3f}s", func.__name__, elapsed)
    return wrapper

def async_timed(func: Callable[P, R]) -> Callable[P, R]:
    @functools.wraps(func)
    async def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        start = time.perf_counter()
        try:
            return await func(*args, **kwargs)
        finally:
            elapsed = time.perf_counter() - start
            logger.debug("{} took {:.3f}s", func.__name__, elapsed)
    return wrapper
```

### 缓存装饰器

```python
from functools import cache, lru_cache

@cache
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

@lru_cache(maxsize=128)
def get_user(user_id: int) -> User:
    return db.query(User).get(user_id)
```

### 验证装饰器

```python
def validate_positive(func: Callable[P, R]) -> Callable[P, R]:
    @functools.wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        for arg in args:
            if isinstance(arg, (int, float)) and arg <= 0:
                raise ValueError(f"Expected positive number, got {arg}")
        for key, value in kwargs.items():
            if isinstance(value, (int, float)) and value <= 0:
                raise ValueError(f"{key} must be positive, got {value}")
        return func(*args, **kwargs)
    return wrapper
```

### 限流装饰器

```python
import asyncio
import time

def rate_limit(calls: int, period: float):
    semaphore = asyncio.Semaphore(calls)
    last_reset = time.monotonic()

    def decorator(func: Callable[P, R]) -> Callable[P, R]:
        @functools.wraps(func)
        async def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            nonlocal last_reset

            now = time.monotonic()
            if now - last_reset >= period:
                for _ in range(calls - semaphore._value):
                    semaphore.release()
                last_reset = now

            async with semaphore:
                return await func(*args, **kwargs)

        return wrapper
    return decorator
```

## @contextmanager - 生成器变上下文

```python
import time
from contextlib import contextmanager
from loguru import logger

@contextmanager
def timed_operation(name: str):
    start = time.perf_counter()
    try:
        yield
    finally:
        elapsed = time.perf_counter() - start
        logger.info("{} took {:.3f}s", name, elapsed)
```

## 装饰器组合

```python
@timed
@retry(3)
@validate_input
async def process(data: dict) -> Result:
    ...
```

## 常见陷阱

| 陷阱 | 问题 | 解决方案 |
|------|------|----------|
| 丢失元数据 | `__name__`, `__doc__` 被覆盖 | 使用 `@functools.wraps` |
| 同步/异步混用 | 同步装饰器用于异步函数 | 分别实现 sync/async 版本 |
| 闭包变量绑定 | 循环中的 lambda 捕获问题 | 使用默认参数或 `functools.partial` |
| 装饰器里藏业务逻辑 | 控制流变隐式，难排查 | 仅放横切关注点，业务规则留在显式函数里 |

## 与 FastAPI 集成

```python
import functools
from fastapi import Depends, HTTPException

def require_admin(func):
    @functools.wraps(func)
    async def wrapper(*args, current_user: User = Depends(get_current_user), **kwargs):
        if not current_user.is_admin:
            raise HTTPException(status_code=403, detail="Admin required")
        return await func(*args, current_user=current_user, **kwargs)
    return wrapper
```
