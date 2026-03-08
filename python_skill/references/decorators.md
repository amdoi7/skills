# Decorators (装饰器)

> [!info] Migration Notice  
> This file is being merged into [[Vibe_Coding/skill/python_skill/references/meta-and-context.md]].  
> Prefer the merged guide for new usage.

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
    @functools.wraps(func)  # 保留原函数元数据
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        # 前置逻辑
        result = func(*args, **kwargs)
        # 后置逻辑
        return result
    return wrapper

@decorator
def my_function(): ...
```

### 带参数装饰器（三层嵌套）

```python
def retry(max_attempts: int = 3, delay: float = 1.0):
    """带参数的重试装饰器"""
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
from dataclasses import dataclass

def singleton(cls):
    """单例装饰器"""
    instances = {}

    @functools.wraps(cls)
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]

    return get_instance

@singleton
class Database:
    def __init__(self, url: str): ...
```

## 实用装饰器

### 计时装饰器

```python
import time
from loguru import logger

def timed(func: Callable[P, R]) -> Callable[P, R]:
    """同步函数计时"""
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
    """异步函数计时"""
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

# 无界缓存 (Python 3.9+)
@cache
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

# 有界 LRU 缓存
@lru_cache(maxsize=128)
def get_user(user_id: int) -> User:
    return db.query(User).get(user_id)

# 清除缓存
get_user.cache_clear()
```

### 验证装饰器

```python
def validate_positive(func: Callable[P, R]) -> Callable[P, R]:
    """验证所有数值参数为正数"""
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

@validate_positive
def calculate_area(width: float, height: float) -> float:
    return width * height
```

### 限流装饰器

```python
import asyncio
import time

def rate_limit(calls: int, period: float):
    """速率限制装饰器"""
    semaphore = asyncio.Semaphore(calls)
    last_reset = time.monotonic()

    def decorator(func: Callable[P, R]) -> Callable[P, R]:
        @functools.wraps(func)
        async def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            nonlocal last_reset

            now = time.monotonic()
            if now - last_reset >= period:
                # 重置信号量
                for _ in range(calls - semaphore._value):
                    semaphore.release()
                last_reset = now

            async with semaphore:
                return await func(*args, **kwargs)

        return wrapper
    return decorator

@rate_limit(calls=10, period=1.0)  # 每秒最多 10 次
async def call_api(endpoint: str) -> dict: ...
```

## 描述符协议与装饰器

```python
class CachedProperty:
    """延迟计算的缓存属性"""

    def __init__(self, func):
        self.func = func
        self.__doc__ = func.__doc__

    def __set_name__(self, owner, name):
        self.name = name

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        value = self.func(obj)
        setattr(obj, self.name, value)  # 替换描述符
        return value

class User:
    @CachedProperty
    def profile(self) -> dict:
        """只计算一次"""
        return expensive_load_profile()
```

## @contextmanager - 生成器变上下文

将生成器转化为上下文管理器，比类实现更简洁：

```python
from contextlib import contextmanager
import time

@contextmanager
def timed_operation(name: str):
    """计时上下文管理器"""
    start = time.perf_counter()
    try:
        yield  # __enter__ 结束点，__exit__ 开始点
    finally:
        elapsed = time.perf_counter() - start
        logger.info("{} took {:.3f}s", name, elapsed)

# 使用
with timed_operation("database_query"):
    db.execute(sql)
```

### 带返回值的上下文

```python
@contextmanager
def open_db_transaction(conn):
    """数据库事务管理"""
    tx = conn.begin()
    try:
        yield tx  # 返回给 as 变量
        tx.commit()
    except Exception:
        tx.rollback()
        raise

with open_db_transaction(conn) as tx:
    tx.execute(...)
```

### ContextDecorator - 双重身份

同时支持 `with` 和 `@` 两种用法：

```python
from contextlib import ContextDecorator

class track_time(ContextDecorator):
    def __enter__(self):
        self.start = time.perf_counter()
        return self

    def __exit__(self, *args):
        logger.info("Elapsed: {:.3f}s", time.perf_counter() - self.start)
        return False

# 作为装饰器
@track_time()
def slow_function(): ...

# 作为上下文管理器
with track_time():
    slow_function()
```

## 装饰器组合

```python
# 装饰器从下往上应用
@timed           # 3. 最外层
@retry(3)        # 2. 中间层
@validate_input  # 1. 最内层
async def process(data: dict) -> Result:
    ...

# 等价于:
# process = timed(retry(3)(validate_input(process)))
```

## 常见陷阱

| 陷阱 | 问题 | 解决方案 |
|------|------|----------|
| 丢失元数据 | `__name__`, `__doc__` 被覆盖 | 使用 `@functools.wraps` |
| 同步/异步混用 | 同步装饰器用于异步函数 | 分别实现 sync/async 版本 |
| 闭包变量绑定 | 循环中的 lambda 捕获问题 | 使用默认参数或 `functools.partial` |
| 类方法装饰 | `self` 参数处理不当 | 正确处理描述符协议 |

## 与 FastAPI 集成

```python
from fastapi import Depends, HTTPException

def require_admin(func):
    """要求管理员权限"""
    @functools.wraps(func)
    async def wrapper(*args, current_user: User = Depends(get_current_user), **kwargs):
        if not current_user.is_admin:
            raise HTTPException(status_code=403, detail="Admin required")
        return await func(*args, current_user=current_user, **kwargs)
    return wrapper

@router.delete("/users/{user_id}")
@require_admin
async def delete_user(user_id: int, current_user: User) -> dict:
    ...
```
