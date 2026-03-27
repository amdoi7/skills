# Contextlib Utilities (上下文管理工具)

> Detailed context management reference for this skill.

> "Small tools for big problems: suppressing errors, closing streams, and handling optional contexts."

## Quick Reference

| 工具 | 作用 | 等价代码 |
|------|------|----------|
| `closing(obj)` | 自动调用 `.close()` | `try: yield obj finally: obj.close()` |
| `suppress(*ex)` | 忽略特定异常 | `try: ... except ex: pass` |
| `nullcontext(val)` | 空操作占位符 | `yield val` |
| `redirect_stdout(f)` | 重定向输出 | 临时替换 `sys.stdout` |
| `ExitStack` | 动态管理 N 个资源 | LIFO 清理栈 |

## suppress

```python
import os
from contextlib import suppress

with suppress(FileNotFoundError):
    os.remove("temp.tmp")
```

只忽略你**明确预期**的异常，类型尽量窄。

## closing

```python
from contextlib import closing
from urllib.request import urlopen

with closing(urlopen("https://api.example.com")) as response:
    data = response.read()
```

## nullcontext

```python
from contextlib import nullcontext

def process(file_path: str | None = None):
    cm = open(file_path) if file_path else nullcontext("default data")

    with cm as source:
        if isinstance(source, str):
            return handle_string(source)
        return handle_file(source)
```

## redirect_stdout

```python
from contextlib import redirect_stdout
from io import StringIO

buffer = StringIO()
with redirect_stdout(buffer):
    print("Hello, World!")

captured = buffer.getvalue()
```

## ExitStack

```python
from contextlib import ExitStack

def process_files(paths: list[str]):
    with ExitStack() as stack:
        files = [stack.enter_context(open(p)) for p in paths]
        for f in files:
            process(f)
```

### 模拟 defer

```python
from contextlib import ExitStack

with ExitStack() as stack:
    stack.callback(print, "3. Cleanup")
    stack.callback(print, "2. Release")
    print("1. Work")
```

### 转移资源所有权

```python
from contextlib import ExitStack

def open_resources() -> ExitStack:
    stack = ExitStack()
    try:
        stack.enter_context(open("a.txt"))
        stack.enter_context(open("b.txt"))
    except Exception:
        stack.close()
        raise
    return stack.pop_all()
```

## AsyncExitStack

```python
from contextlib import AsyncExitStack

async def process():
    async with AsyncExitStack() as stack:
        conn = await stack.enter_async_context(get_db_connection())
        stack.enter_context(open("log.txt"))
        stack.push_async_callback(async_cleanup)
        stack.callback(sync_cleanup)
        await do_work(conn)
```

## 选择指南

| 场景 | 推荐工具 |
|------|----------|
| 忽略特定异常 | `suppress` |
| 关闭没有 `with` 支持的对象 | `closing` |
| 条件性资源 / 可选上下文 | `nullcontext` |
| 捕获 `print` 输出 | `redirect_stdout` |
| 动态数量的资源 | `ExitStack` |
| 模拟 defer 模式 | `ExitStack.callback` |
| 异步资源管理 | `AsyncExitStack` |
