# Async & Concurrency

## When to Use What

| Workload Type | Recommended Approach |
|---------------|---------------------|
| I/O-bound (network, files) | `asyncio` (preferred) or `threading` |
| CPU-bound (computation) | `ProcessPoolExecutor` |
| Mixed | Combine: async for I/O, executor for CPU |

## Asyncio Fundamentals

### Event Loop & Cooperative Scheduling

```python
import asyncio

# Coroutines yield control at await points
async def fetch_data(url: str) -> dict:
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()  # Yields here

# NEVER block the event loop
async def bad_example():
    time.sleep(5)  # WRONG: blocks entire loop
    requests.get(url)  # WRONG: blocking I/O

async def good_example():
    await asyncio.sleep(5)  # OK: yields control
    await aiohttp.get(url)  # OK: async I/O
```

### Structured Concurrency (TaskGroup)

```python
async def fetch_all(urls: list[str]) -> list[dict]:
    results = []

    async with asyncio.TaskGroup() as tg:
        for url in urls:
            tg.create_task(fetch_and_append(url, results))

    return results

async def fetch_and_append(url: str, results: list):
    data = await fetch_data(url)
    results.append(data)
```

**TaskGroup benefits:**
- All tasks complete or all fail together
- Automatic cleanup on exception
- Clear scope boundaries

### Cancellation Handling

```python
async def worker(queue: asyncio.Queue):
    try:
        while True:
            item = await queue.get()
            await process(item)
            queue.task_done()
    except asyncio.CancelledError:
        # Cleanup resources
        logger.info("Worker cancelled, cleaning up")
        raise  # MUST re-raise to propagate cancellation

# Protect atomic operations
async def critical_operation():
    async with asyncio.shield(save_to_database()):
        pass  # Won't be cancelled even if parent is
```

### Timeout Management

```python
# Modern timeout context manager (3.11+)
async def fetch_with_timeout(url: str) -> dict:
    async with asyncio.timeout(10):
        return await fetch_data(url)

# Handle timeout
async def safe_fetch(url: str) -> dict | None:
    try:
        async with asyncio.timeout(10):
            return await fetch_data(url)
    except TimeoutError:
        logger.warning("Fetch timed out for {}", url)
        return None
```

### Backpressure Control

```python
# Limit concurrent operations
async def process_urls(urls: list[str], max_concurrent: int = 10):
    semaphore = asyncio.Semaphore(max_concurrent)

    async def bounded_fetch(url: str) -> dict:
        async with semaphore:
            return await fetch_data(url)

    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(bounded_fetch(url)) for url in urls]

    return [t.result() for t in tasks]

# Queue-based backpressure
async def producer_consumer():
    queue: asyncio.Queue[str] = asyncio.Queue(maxsize=100)

    async def producer():
        for item in items:
            await queue.put(item)  # Blocks if queue full

    async def consumer():
        while True:
            item = await queue.get()
            await process(item)
            queue.task_done()
```

## Threading (for I/O when async not available)

```python
from concurrent.futures import ThreadPoolExecutor
import asyncio

# Run blocking I/O in thread pool
async def fetch_legacy_api(url: str) -> dict:
    loop = asyncio.get_running_loop()
    return await loop.run_in_executor(
        None,  # Default executor
        requests.get,  # Blocking function
        url
    )

# Custom thread pool
executor = ThreadPoolExecutor(max_workers=4)

async def process_files(paths: list[Path]) -> list[str]:
    loop = asyncio.get_running_loop()
    tasks = [
        loop.run_in_executor(executor, read_file, path)
        for path in paths
    ]
    return await asyncio.gather(*tasks)
```

## Multiprocessing (for CPU-bound)

```python
from concurrent.futures import ProcessPoolExecutor
import asyncio

def cpu_intensive(data: bytes) -> bytes:
    """Pure function for CPU work - must be picklable."""
    return heavy_computation(data)

async def process_batch(items: list[bytes]) -> list[bytes]:
    loop = asyncio.get_running_loop()

    with ProcessPoolExecutor(max_workers=4) as executor:
        tasks = [
            loop.run_in_executor(executor, cpu_intensive, item)
            for item in items
        ]
        return await asyncio.gather(*tasks)
```

**ProcessPoolExecutor rules:**
- Functions must be picklable (top-level, no closures)
- Prefer `ProcessPoolExecutor` over raw `multiprocessing`
- Share data via queues or shared memory, not global state

## Time Functions

| 函数 | 用途 | 精度 | 不受系统时钟影响 |
|------|------|------|------------------|
| `time.monotonic()` | TTL / 超时 / deadline | 中高 | ✅ |
| `time.perf_counter()` | benchmark / profiling | 最高 | ✅ |
| `time.time()` | 需要真实时间戳 | 中 | ❌ 会被 NTP 调整 |

## Caching

```python
from functools import cache, lru_cache
import time  # Use monotonic() for TTL, perf_counter() for profiling

# Unbounded cache (Python 3.9+)
@cache
def get_config(key: str) -> str:
    return expensive_lookup(key)

# Bounded LRU cache
@lru_cache(maxsize=128)
def get_user_settings(user_id: int) -> dict:
    return load_settings(user_id)

# Async TTL cache with monotonic time
class AsyncTTLCache[K, V]:
    """Thread-safe async cache with TTL using monotonic time."""

    def __init__(self, ttl: float = 300):
        self.ttl = ttl
        self._cache: dict[K, tuple[float, V]] = {}
        self._lock = asyncio.Lock()

    async def get(self, key: K, factory: Callable[[K], Awaitable[V]]) -> V:
        now = time.monotonic()

        async with self._lock:
            if key in self._cache:
                cached_time, data = self._cache[key]
                if now - cached_time < self.ttl:
                    return data

            data = await factory(key)
            self._cache[key] = (now, data)
            return data

# Usage
_cache = AsyncTTLCache[str, dict](ttl=300)

async def cached_fetch(url: str) -> dict:
    return await _cache.get(url, fetch_data)
```

## Common Patterns

### Graceful Shutdown

```python
async def main():
    # Setup
    queue = asyncio.Queue()
    workers = [asyncio.create_task(worker(queue)) for _ in range(4)]

    try:
        await run_server()
    finally:
        # Signal workers to stop
        for _ in workers:
            await queue.put(None)

        # Wait for completion with timeout
        await asyncio.wait_for(
            asyncio.gather(*workers, return_exceptions=True),
            timeout=30
        )
```

### Rate Limiting

```python
class RateLimiter:
    def __init__(self, rate: float, per: float = 1.0):
        self.rate = rate
        self.per = per
        self.tokens = rate
        self.last_update = time.monotonic()
        self.lock = asyncio.Lock()

    async def acquire(self):
        async with self.lock:
            now = time.monotonic()
            elapsed = now - self.last_update
            self.tokens = min(self.rate, self.tokens + elapsed * (self.rate / self.per))
            self.last_update = now

            if self.tokens < 1:
                wait_time = (1 - self.tokens) * (self.per / self.rate)
                await asyncio.sleep(wait_time)
                self.tokens = 0
            else:
                self.tokens -= 1
```

## Designing Sync and Async Facades

- Keep one conceptual API, not two unrelated products.
- Match names, argument order, return semantics, and error meaning where possible.
- Only diverge when Python semantics force it, such as `await`, async iteration, or context-manager protocol differences.
- Centralize core behavior in one implementation path instead of letting sync and async logic drift apart.
- Mirror tests across both surfaces and run both in CI if both are public.

```python
user = fetch_user(user_id)
user = await fetch_user_async(user_id)
```

## Pitfalls (常见陷阱)

### 1. 忘记 await

```python
# ❌ 错误：忘记 await，只创建了协程对象
result = fetch_data()  # <coroutine object ...>

# ✅ 正确
result = await fetch_data()
```

**检测**: 启用 `RuntimeWarning` 警告

```python
import warnings
warnings.filterwarnings('error', category=RuntimeWarning, message='.*was never awaited')
```

### 2. 阻塞事件循环

```python
# ❌ 阻塞操作 - 冻结整个事件循环
time.sleep(1)
requests.get(url)
open(file).read()

# ✅ 异步替代方案
await asyncio.sleep(1)
async with aiohttp.ClientSession() as s: await s.get(url)
async with aiofiles.open(file) as f: await f.read()

# ✅ 对于不可避免的阻塞，使用 executor
loop = asyncio.get_running_loop()
await loop.run_in_executor(None, blocking_function)
```

### 3. 错误的异常处理

```python
# ❌ 错误：捕获所有异常（包括 CancelledError）
try:
    await some_task()
except:  # 或 except Exception:
    pass

# ✅ 正确：显式处理 CancelledError
try:
    await some_task()
except asyncio.CancelledError:
    # 清理资源
    raise  # 必须重新抛出！
except Exception as e:
    logger.error("Task failed: {}", e)
```

### 4. 过度并发

```python
# ❌ 危险：创建过多并发任务
tasks = [fetch(url) for url in urls]  # 10000 个任务
await asyncio.gather(*tasks)  # 耗尽资源

# ✅ 使用信号量限制并发
semaphore = asyncio.Semaphore(20)

async def limited_fetch(url):
    async with semaphore:
        return await fetch(url)

tasks = [limited_fetch(url) for url in urls]
await asyncio.gather(*tasks)
```

### 5. gather 异常处理

```python
# ❌ 一个失败导致全部取消
results = await asyncio.gather(*tasks)  # 抛出第一个异常

# ✅ 收集所有结果（包括异常）
results = await asyncio.gather(*tasks, return_exceptions=True)
for result in results:
    if isinstance(result, Exception):
        logger.error("Task failed: {}", result)
```

### 6. 共享状态竞态

```python
# ❌ 不安全：协程间共享可变状态
counter = 0
async def increment():
    global counter
    temp = counter
    await asyncio.sleep(0.001)
    counter = temp + 1  # 竞态条件！

# ✅ 使用 asyncio.Lock
lock = asyncio.Lock()
async def safe_increment():
    global counter
    async with lock:
        counter += 1
```

## Pitfall Checklist

| 类别 | 检查项 |
|------|--------|
| **await** | 所有 async 调用都有 await |
| **阻塞** | 无 `time.sleep`, `requests`, 同步 I/O |
| **并发** | 使用 Semaphore 限制并发数 |
| **异常** | `CancelledError` 重新抛出 |
| **gather** | 使用 `return_exceptions=True` |
| **状态** | 共享状态用 `asyncio.Lock` 保护 |
