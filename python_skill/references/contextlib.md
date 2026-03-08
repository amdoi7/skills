# Contextlib Utilities (上下文管理工具)

> [!info] Migration Notice  
> This file is being merged into [[Vibe_Coding/skill/python_skill/references/meta-and-context.md]].  
> Prefer the merged guide for new usage.

> "Small tools for big problems: suppressing errors, closing streams, and handling optional contexts."

## Quick Reference

| 工具 | 作用 | 等价代码 |
|------|------|----------|
| `closing(obj)` | 自动调用 `.close()` | `try: yield obj finally: obj.close()` |
| `suppress(*ex)` | 忽略特定异常 | `try: ... except ex: pass` |
| `nullcontext(val)` | 空操作占位符 | `yield val` |
| `redirect_stdout(f)` | 重定向输出 | 临时替换 `sys.stdout` |
| `ExitStack` | 动态管理 N 个资源 | LIFO 清理栈 |

## suppress - 优雅忽略异常

```python
from contextlib import suppress
import os

# ✅ 简洁：忽略文件不存在
with suppress(FileNotFoundError):
    os.remove('temp.tmp')

# ✅ 多个异常类型
with suppress(FileNotFoundError, PermissionError):
    os.remove('temp.tmp')

# ❌ 传统写法：冗长
try:
    os.remove('temp.tmp')
except FileNotFoundError:
    pass
```

**注意**: 只忽略你 *明确预期* 的异常，类型要尽可能具体。

## closing - 驯服旧接口

用于有 `close()` 但没实现 `__enter__/__exit__` 的对象：

```python
from contextlib import closing
from urllib.request import urlopen

# 旧接口适配
with closing(urlopen('https://api.example.com')) as response:
    data = response.read()
# 自动调用 response.close()
```

## nullcontext - 条件性上下文

处理"可能有资源，也可能没有"的场景：

```python
from contextlib import nullcontext

def process(file_path: str | None = None):
    """统一的代码路径，无需 if-else 分支"""
    cm = open(file_path) if file_path else nullcontext("default data")

    with cm as source:
        if isinstance(source, str):
            return handle_string(source)
        return handle_file(source)

# 两种调用方式，内部逻辑一致
process("input.txt")  # 从文件读取
process()             # 使用默认数据
```

**适用场景**:
- 条件性资源获取
- 测试 Mock 替换
- 保持 API 一致性

## redirect_stdout - 捕获输出

```python
from contextlib import redirect_stdout
from io import StringIO

buffer = StringIO()
with redirect_stdout(buffer):
    print("Hello, World!")
    help(str)  # 捕获 help 输出

captured = buffer.getvalue()
```

## ExitStack - 动态资源管理

解决 `with` 语句无法处理 **数量不确定** 资源的问题。

### 动态打开 N 个文件

```python
from contextlib import ExitStack

def process_files(paths: list[str]):
    with ExitStack() as stack:
        # 动态打开所有文件
        files = [stack.enter_context(open(p)) for p in paths]

        # 同时处理所有文件
        for f in files:
            process(f)

    # 离开 with 块，所有文件按 LIFO 顺序关闭
```

### 模拟 Go 的 defer

```python
with ExitStack() as stack:
    stack.callback(print, "3. Cleanup")
    stack.callback(print, "2. Release")
    print("1. Work")

# Output:
# 1. Work
# 2. Release
# 3. Cleanup
```

### 异常安全的资源转移

```python
def open_resources() -> ExitStack:
    """打开资源，成功则转移所有权给调用者"""
    stack = ExitStack()
    try:
        stack.enter_context(open("a.txt"))
        stack.enter_context(open("b.txt"))
    except Exception:
        stack.close()  # 失败时清理已打开的资源
        raise
    return stack.pop_all()  # 成功，转移所有权

# 调用者负责关闭
with open_resources() as stack:
    # 使用资源...
    pass
```

### 核心方法

| 方法 | 功能 |
|------|------|
| `enter_context(cm)` | 进入上下文管理器并压栈 |
| `callback(func, *args)` | 注册清理回调 |
| `pop_all()` | 转移所有权到新 Stack |
| `close()` | 手动触发清理 |

## AsyncExitStack - 异步版本

```python
from contextlib import AsyncExitStack

async def process():
    async with AsyncExitStack() as stack:
        # 异步上下文管理器
        conn = await stack.enter_async_context(get_db_connection())

        # 同步上下文管理器也可以
        stack.enter_context(open("log.txt"))

        # 异步回调
        stack.push_async_callback(async_cleanup)

        # 同步回调
        stack.callback(sync_cleanup)

        await do_work(conn)
```

## 选择指南

| 场景 | 推荐工具 |
|------|----------|
| 忽略特定异常 | `suppress` |
| 关闭没有 `with` 支持的对象 | `closing` |
| 条件性资源 / 可选上下文 | `nullcontext` |
| 捕获 print 输出 | `redirect_stdout` |
| 动态数量的资源 | `ExitStack` |
| 模拟 defer 模式 | `ExitStack.callback` |
| 异步资源管理 | `AsyncExitStack` |
