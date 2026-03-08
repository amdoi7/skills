# Git Safety

在执行任何 `git add`、`git commit`、`git push`、`gh pr` 之前，先应用这些约束。

## 必守规则

- 只在用户明确要求时提交或创建 PR。
- 不要修改 git config。
- 不要执行破坏性命令：`reset --hard`、`checkout .`、`restore .`、`clean -f`、`branch -D`、`push --force`。
- 不要跳过 hooks：禁止 `--no-verify`、`--no-gpg-sign` 等，除非用户明确要求。
- 不要默认使用 `git add -A` 或 `git add .`；优先显式列出 `scope_paths` 内的目标文件。
- 不要提交疑似敏感文件，如 `.env`、私钥、token、凭据导出文件。
- 如果 hook 失败，说明 commit 没成功；修复后重新创建**新 commit**，不要 `--amend`。

## 范围检查

1. 先按 `references/git-context.md` 建立 `scope_paths`。
2. 若仓库里存在无关改动，只把 `scope_paths` 内文件加入暂存区。
3. 若 `scope_paths` 仍然含糊，先问清楚，不要擅自提交全部变更。
