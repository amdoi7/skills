# Pull Request Workflow

只有在 `mode=pr` 且用户明确要求创建或更新 PR 时才执行本流程。

## 1. 确认分支上下文

以前置的 `references/git-context.md` 为准，重点使用：

- `current_branch`
- `base_branch`
- `git diff <base_branch>...HEAD`
- `git log --oneline <base_branch>..HEAD`

如果工作区仍是 dirty tree，再补看 `git status --short` 的残留项；否则不要重复全量工作树 diff。

## 2. 准备分支

- 如果当前就在默认分支上，先创建新分支。
- 如果当前分支尚未推送，使用 `git push -u origin <current_branch>`。
- 不要强推默认分支。

## 3. 起草 PR 标题与正文

标题要求：

- 小于 70 字符
- 直接概括用户收益或变更目的

正文模板：

```markdown
## Summary
- <1-3 条关键变化>

## Test Plan
- [ ] <待验证项>
```

如果已有 PR，就更新标题和正文，使其匹配当前 branch diff。

## 4. 执行并回传

- 新建 PR：`gh pr create`
- 更新 PR：`gh pr edit`
- 把 PR URL 传给 `references/output-contract.md`
