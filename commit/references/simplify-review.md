# Simplify Review

在 diff 较大、改动混杂，或用户明确要求“先清理 / 先 review”时，再执行本流程。

## 1. 缩小 review 范围

优先只 review `scope_paths` 内的 diff：

- 有 staged 改动时：`git diff HEAD -- <scope_paths>`
- 否则：`git diff -- <scope_paths>`
- 若目标文件未跟踪，用 `git status --short -- <scope_paths>` 补充确认

如果仓库里有大量无关改动，只 review 当前准备提交的路径。

## 2. 用三种 lens 过一遍

把 review 拆成三种视角；可串行，也可并行，但**不是默认必须三 agent 并发**。

- **Reuse**：是否已有现成 helper、模板、参考结构可复用
- **Quality**：是否有重复规则、泄漏实现细节、边界不清或结构过重
- **Efficiency**：是否存在重复检查、过宽读取、重复步骤或不必要的流程负担

## 3. 复用现有 checklist

当 `scope_paths` 指向特定类型内容时，优先复用相邻 skill 的 checklist：

- Python 代码：`modern-python/references/review-checklist.md`
- REST/API 设计：`restful-api-design/references/review-checklist.md`
- Go 设计与代码健康：`idiomatic-go/references/philosophy.md`

当前 skill 自己更关注**提交边界、文档结构、流程轻重**，不要重新发明一整套语言级 review 规则。

## 4. 默认先报告，再决定是否修

- 默认输出 findings 与建议。
- 只有在任务本身明确包含 cleanup / refactor，或当前仍处于提交前整理阶段且修复风险很低时，才直接改动。
- 只修 `scope_paths` 内的问题，不顺手扩散到外围无关文件。
