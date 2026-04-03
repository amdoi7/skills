# Composition

篇章层面的 AI 写作模式。核心问题：用结构模板替代有机的思考推进。

## 循环总结（Fractal Summaries）

"我要告诉你什么；我正在告诉你什么；我刚才告诉了你什么"——在文档的每一层都重复这个模式。每个小节有总结，每个章节有总结，整篇文章还有总结。

坏例子：
- "In this section, we'll explore... [3000 字后] ...as we've seen in this section."
- 结尾段重述了前文已经说过的每一个要点
- "And so we return to where we began."

修正方式：
- 删掉所有"在本节中我们将..."和"正如我们在本节看到的..."
- 结尾应该推进到新的位置（下一步行动、开放问题、具体建议），而不是回到起点
- 如果读者需要回顾，他们会自己往回翻

## 死比喻（The Dead Metaphor）

抓住一个比喻，在整篇文章里反复使用到死。人类作者会引入比喻、用一次、然后继续。AI 会在同一篇文章里把同一个比喻重复 5-10 次。

坏例子：
- "walls and doors" 在同一篇文章里出现 30 多次
- 每段都想办法再说一次 "primitives"
- "The ecosystem needs ecosystems to build ecosystem value."

修正方式：一个比喻用一次就放下。如果你发现自己在第二段又回到了同一个比喻，换一个表达方式或者直接用字面描述。

## 历史类比堆砌

快速列举历史上的公司或技术革命来建立虚假的权威感。在技术写作中尤其常见。

坏例子：
- "Apple didn't build Uber. Facebook didn't build Spotify. Stripe didn't build Shopify."
- "Every major technological shift -- the web, mobile, social, cloud -- followed the same pattern."
- "Take Spotify... Or consider Uber... Airbnb followed a similar path... Shopify is another example..."

问题：(1) 这些类比通常过于简化 (2) 读者见过太多次了 (3) 列举案例不等于论证。

修正方式：最多用一个历史案例，深入分析它和你的论点之间的具体关联。不要做案例点名册。

## 一点稀释（One-Point Dilution）

一个论点用 10 种方式重述，横跨几千字。模型把简单的主题句膨胀成"全面"的文章，方法是用不同的比喻、例子和框架重新表述同一个想法。800 字的论点变成 4000 字的循环重复。

修正方式：
- 写完后问："这篇文章有几个独立论点？"如果答案是 1，文章不应该超过 1000 字
- 每段结束时检查：这段说了什么上一段没说的？如果答案是"没有"，合并或删除

## 内容重复

在同一篇文章里逐字重复整段或整节。模型在长文中失去对已写内容的追踪。

修正方式：写完后搜索关键短语，确认没有重复段落。

## 路标式结尾（The Signposted Conclusion）

用"总之"、"综上所述"、"In conclusion"来宣布结尾。好的写作不需要告诉读者它正在结尾。读者能感受到。

坏例子：
- "In conclusion, the future of AI depends on..."
- "To sum up, we've explored three key themes..."
- "In summary, the evidence suggests..."

修正方式：直接写你的最终论点或行动建议。如果你的结尾需要"总之"来提醒读者这是结尾，说明结尾本身写得不够有力。

## "Despite Its Challenges..." 僵化公式

AI 承认问题只是为了立刻否定它们。永远遵循同一节奏："Despite its [正面词], [主语] faces challenges..." 然后以 "Despite these challenges, [乐观结论]" 结尾。

坏例子：
- "Despite these challenges, the initiative continues to thrive."
- "Despite its industrial prosperity, Korattur faces challenges typical of urban areas."

修正方式：如果挑战是真的，给它们应有的篇幅和分析。如果挑战不重要，不要提它们。"虽然有问题但还是很好"不是分析，是敷衍。

## 自查规则

1. 检查开头和结尾是否说了同一句话的不同版本——如果是，重写结尾
2. 搜索核心比喻的关键词——如果出现超过 3 次，替换多余的
3. 数一数文章里提到了多少个"历史案例"——超过 2 个就削减
4. 对每段问"这段推进了什么新内容？"——如果答不上来，合并或删除
5. 搜索"总之"、"综上"、"In conclusion"、"In summary"——删掉，直接写结论
6. 搜索"despite"——检查是否在用公式化的"虽然有问题但很好"
