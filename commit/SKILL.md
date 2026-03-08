---
name: commit
description: 安全地分批暂存、起草 Conventional Commits 风格提交信息、拆分混杂改动，并在需要时创建或更新 Pull Request。Use when asked to git commit, split changes into multiple commits, draft commit messages, stage only selected files, or create/edit a PR.
---

# Commit Skill

产出**安全、可 review、边界清晰**的 git commit；必要时继续创建或更新 PR。

默认顺序：**先归一化任务上下文，再收敛 diff，最后执行最小 git/gh 操作**。

## 决策顺序

1. **边界正确**：先确定本次动作只覆盖哪些路径。
2. **安全优先**：先应用 git 安全约束，再考虑速度。
3. **可审查性**：混杂改动先拆分，再提交或发 PR。
4. **消息准确**：提交主题行与 PR 标题都描述意图与收益。
5. **最小操作面**：只运行当前模式真正需要的 git/gh 命令。

## Task Context

先把任务归一化为以下字段，并在 references 中复用同一套命名：

- `mode`：`commit` 或 `pr`
- `scope_paths`：本次动作允许包含的文件、目录或 glob
- `base_branch`：PR 模式下用于 `...HEAD` 对比的基线分支
- `current_branch`：当前工作分支
- `leftover_changes`：本次动作完成后有意保留在工作区的改动

## Reference Matrix

| 场景 | 参考文件 |
| --- | --- |
| 安全边界 | `references/git-safety.md` |
| 收集共享 git 上下文 | `references/git-context.md` |
| 起草并创建 commit | `references/commit-workflow.md` |
| 创建或更新 PR | `references/pr-workflow.md` |
| 大 diff / 混杂改动的精简 review | `references/simplify-review.md` |
| 最终回报格式 | `references/output-contract.md` |

只加载当前模式需要的 reference，不要一次性全读。

## Workflow Router

### `mode=commit`

1. 读取 `references/git-safety.md`。
2. 按 `references/git-context.md` 收集 `scope_paths` 内的 git 上下文。
3. 按 `references/commit-workflow.md` 完成拆分、暂存、起草消息与提交。
4. 按 `references/output-contract.md` 回报结果。

### `mode=pr`

1. 先确认是否还需要补 commit；如果需要，先走 `mode=commit`。
2. 读取 `references/git-context.md`，补齐 `base_branch` 与 `current_branch`。
3. 按 `references/pr-workflow.md` 创建或更新 PR。
4. 按 `references/output-contract.md` 回报结果，并返回 PR URL。

### `mode=simplify-before-commit`

当 diff 较大、改动混杂，或用户明确要求“先清理/先 review”时，再读取 `references/simplify-review.md`。
