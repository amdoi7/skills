# Commit Workflow

按下面顺序完成提交；以 `references/git-context.md` 中的公共上下文为前提。

## 1. 判断是否需要拆分

满足任一情况时，优先拆成多个 commit：

- 同时包含功能改动、重构、文档或格式化
- 同一文件里有明显独立的逻辑块
- 改动难以用一句主题行准确概括
- 有些文件超出 `scope_paths`

拆分目标：每个 commit 都能独立说明“为什么值得存在”。

## 2. 起草提交信息

主题行格式：

```text
<type>(<scope>): <summary>
```

选择规则：

- `feat`：新增能力
- `fix`：修复 bug 或错误行为
- `refactor`：重构但不改变外部行为
- `docs`：文档改动
- `test`：测试改动
- `perf`：性能优化
- `chore`：杂项维护

写法要求：

- 用祈使句
- 控制在 72 字符内
- 不带句号
- 聚焦“为什么做这次提交”

`body` 仅在以下情况添加：

- 需要解释取舍或限制
- 需要说明拆分策略
- 需要提示后续验证方式

## 3. 暂存并提交

优先显式列出文件：

```bash
git add path/to/file1 path/to/file2
```

默认优先：

```bash
git commit -m "<subject>" -m "<body>"
```

仅在消息较长、需要稳定保留换行，或调用环境要求时，才改用 heredoc。

## 4. 记录结果

记录并传递给 `references/output-contract.md`：

- 实际提交的 `scope_paths`
- 最终主题行
- `leftover_changes`
