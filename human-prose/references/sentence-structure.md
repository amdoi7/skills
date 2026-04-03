# Sentence Structure

句子层面的 AI 写作模式。核心问题：反复使用同一种句式制造虚假的修辞效果。

## 否定并列 "It's not X -- it's Y"

最常被识别的 AI 写作特征。用"不是 X，而是 Y"制造虚假的惊人反转。

一篇文章里用一次可以。用十次就是侮辱读者。

变体：
- "not because X, but because Y"（因果反转版）
- "X -- not Y"（破折号否定版）
- "The question isn't X. The question is Y."（跨句反转版）

坏例子：
- "It's not bold. It's backwards."
- "Half the bugs you chase aren't in your code. They're in your head."

修正方式：直接说你要说的 Y，不需要先否定 X。如果 X 和 Y 的对比确实重要，用正常的转折连接词（但是、不过、然而）而不是戏剧化的句式。

## "Not X. Not Y. Just Z."

戏剧倒计时：连续否定两三个东西，再揭示真正的答案。

坏例子：
- "Not a bug. Not a feature. A fundamental design flaw."
- "Not ten. Not fifty. Five hundred and twenty-three."

修正方式：直接说"523 个 lint 违规"。数字本身就有冲击力，不需要倒计时铺垫。

## "The X? A Y." 自问自答

自己提问、自己回答。没人在问这个问题，模型假装有人在问。

坏例子：
- "The result? Devastating."
- "The worst part? Nobody saw it coming."
- "The scary part? This attack vector is perfect for developers."

修正方式：把问答合并成陈述句。"结果很严重"比"结果？毁灭性的。"更诚实。

## 排比滥用（Anaphora）

连续多个句子用相同开头。

坏例子：
- "They could expose... They could offer... They could provide... They could create..."
- "They have built engines, but not vehicles. They have built power, but not leverage."

修正方式：合并。"他们可以开放 API、提供文档、创建示例"是一句话的事。

## 三段论滥用（Tricolon）

过度使用三件套排列。单个三段论是优雅的；连续三个三段论是模式识别失败。

坏例子：
- "Products impress people; platforms empower them. Products solve problems; platforms create worlds."
- "workflows, decisions, and interactions"

修正方式：一篇文章里最多用一次三段论。如果你发现自己在写第二组三连，停下来重新组织。

## "It's Worth Noting"

空洞的过渡短语，不建立任何逻辑连接。

包括：It bears mentioning, Importantly, Interestingly, Notably, It should be noted that。

修正方式：删掉过渡语，直接说下一个论点。如果论点和上文确实有关系，用具体的逻辑连接词（因为、所以、但是、例如）。

## 浅层分析尾巴

在句尾拖一个 -ing 短语假装在做分析。

坏例子：
- "contributing to the region's rich cultural heritage"
- "highlighting its enduring legacy"
- "underscoring its role as a dynamic hub"

修正方式：如果这个分析值得说，给它一个完整的句子，写清楚为什么和怎么样。如果不值得，删掉。

## 虚假范围 "from X to Y"

用"从 X 到 Y"暗示一个并不存在的连续谱。

坏例子：
- "From innovation to cultural transformation."（中间是什么？）
- "From problem-solving to artistic expression."

修正方式：如果 X 和 Y 确实在一个连续谱上（从 1 到 100、从初级到高级），保留。如果只是两个松散相关的东西，用"和"连接或分开说。

## 自查规则

1. 搜索 "not...but"、"isn't...is"，确认每个实例是否真的需要对比结构
2. 搜索问号，检查是否有自问自答
3. 检查连续 3 个以上句子是否以相同词开头
4. 检查是否有连续多组三件套
5. 删掉所有 "It's worth noting" / "Importantly" / "Interestingly"
6. 搜索句尾的 -ing 从句，评估是否有实质内容
