# Output Contract

完成 commit 或 PR 后，统一按下面格式回报，避免在多个 reference 中各写一套。

## `mode=commit`

- **Included**：本次提交实际包含的 `scope_paths`
- **Message**：最终 commit subject；如有必要再补 1 句 body 摘要
- **Leftover Changes**：有意未提交的改动；没有则写明 clean
- **Checks**：hooks / 测试是否运行，若未运行则明确说明
- **Next Step**：如 `push`、`create PR`、`done`

## `mode=pr`

- **Branch**：`current_branch` 与 `base_branch`
- **PR**：标题与 URL
- **Included**：PR 覆盖的核心变化
- **Leftover Changes**：dirty tree 残留；没有则写明 clean
- **Checks**：已运行 / 未运行的验证
- **Next Step**：如 `request review`、`merge after checks`
