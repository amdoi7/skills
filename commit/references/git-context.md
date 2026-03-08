# Git Context

把当前任务先归一化成统一上下文，再进入 commit / PR 流程。

## Task Context

始终维护以下字段：

- `mode`：`commit` 或 `pr`
- `scope_paths`：允许纳入本次动作的路径集合
- `base_branch`：PR 模式的基线分支；未知时先推断再确认
- `current_branch`：当前分支名
- `leftover_changes`：完成后故意保留的改动

## Shared Preflight

对 `scope_paths` 先运行最小公共检查：

- `git status --short`
- `git diff --staged -- <scope_paths>`
- `git diff -- <scope_paths>`
- `git log --oneline -10`

如果用户没有给 `scope_paths`，先从提示词与工作区状态推断；仍不清晰时再询问。

## Extra Context for `mode=pr`

只有在 `mode=pr` 时，额外收集：

- `git branch --show-current`
- `git diff <base_branch>...HEAD`
- `git log --oneline <base_branch>..HEAD`

如果前一步已经完成 commit，且 `git status --short` 显示工作区干净，就不要重复读取 `git diff --staged` / `git diff` 全量工作树信息。
