---
name: human-prose
description: 写出自然、具体、没有 AI 腔的散文/文档/笔记。审查或改写文本时自动检测常见 AI 写作模式并修正。Use when writing, editing, or reviewing prose — blog posts, notes, documentation, essays — or when output reads like generic AI slop.
---

# Human Prose

写出读起来像人写的文字：具体、有变化、不装腔。

## 何时使用

- 写散文、笔记、博客、文档、README、邮件、评论
- 审查已有文本是否有 AI 腔调
- 改写 AI 生成文本使其自然化
- 用户说"写得像人话"、"别那么 AI"、"去掉套话"

## 不适用

- 纯代码生成（用语言 skill）
- commit message / PR description（用 commit skill）
- 需要刻意使用修辞手法的创意写作（用户明确要求时除外）

## 决策顺序

1. **内容正确**：先把要说的事实、论点、结论搞对
2. **结构自然**：段落组织像人在思考，不像模板在填空
3. **句子有变化**：长短交替、开头各异、不重复同一句式
4. **用词精确**：选最准确的普通词，不选最花哨的同义词
5. **格式克制**：只在真正有用时才用列表/加粗/emoji

## 核心原则

- **具体胜过抽象**：给数字、给名字、给场景，不给"丰富的XX"
- **普通词优先**：用"是"不用"充当"，用"重要"不用"至关重要"
- **一种句式用一次**：同一篇里不重复使用同一个修辞结构
- **段落有重量**：每段至少推进一个论点，不做纯过渡/纯总结
- **删到不能删**：如果去掉一句话意思不变，就去掉

## Reference Matrix

| 检测到的问题 | 参考文件 |
| --- | --- |
| 词汇浮夸、副词堆砌、"delve/leverage/robust" | [references/word-choice.md](references/word-choice.md) |
| 句式重复、反问自答、否定并列、排比滥用 | [references/sentence-structure.md](references/sentence-structure.md) |
| 碎片段落、变相列表、缺乏推进 | [references/paragraph-structure.md](references/paragraph-structure.md) |
| 居高临下、虚假悬念、夸大其词、模糊引用 | [references/tone.md](references/tone.md) |
| 破折号过多、加粗列表、Unicode 装饰 | [references/formatting.md](references/formatting.md) |
| 循环论证、死比喻、历史类比堆砌、重复总结 | [references/composition.md](references/composition.md) |

只加载当前写作/审查需要的 reference，不要一次性全读。

## Workflow

### 写作模式

1. 明确写作目的和读者。
2. 先写内容骨架（论点序列），不写修辞。
3. 逐段填充，每段只推进一件事。
4. 通读一遍，按 Quick Checklist 自查。
5. 如果某类问题频繁出现，加载对应 reference 做针对性修正。

### 审查模式

1. 通读全文，标记可疑段落。
2. 按 Quick Checklist 分类问题。
3. 加载命中最多的 1-2 个 reference。
4. 逐处修正，给出修改前后对照。

## Quick Checklist

写完或审查时逐条过一遍：

- [ ] 有没有同一个句式连用两次以上？（否定并列、排比、反问自答）
- [ ] 有没有一个词/短语在 500 字内出现三次以上？
- [ ] 有没有段落只做过渡或总结、不推进论点？
- [ ] 有没有破折号超过 3 个？
- [ ] 有没有"delve/leverage/robust/tapestry/landscape"等 AI 高频词？
- [ ] 有没有自问自答（"The result? Devastating."）？
- [ ] 有没有虚假悬念（"Here's the thing..."）？
- [ ] 有没有模糊引用（"experts say..."）？
- [ ] 开头和结尾是否说了同一句话的不同版本？
- [ ] 去掉任意一段后文章是否仍然完整？如果是，去掉它。
