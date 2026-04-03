# Formatting

格式层面的 AI 写作模式。核心问题：用格式装饰替代内容组织。

## 破折号上瘾（Em-Dash Addiction）

强迫性地过度使用破折号做戏剧停顿、插入语和转折。人类作者一篇文章可能用 2-3 个（自然地）。AI 会用 20 个以上。

坏例子：
- "The problem -- and this is the part nobody talks about -- is systemic."
- "Not recklessly, not completely -- but enough -- enough to matter."

修正方式：
- 插入语用括号或逗号
- 转折用句号分成两句
- 一篇文章中破折号不超过 3 个
- 如果你发现自己在写破折号，问："用逗号或句号行不行？"通常可以。

## 加粗开头列表（Bold-First Bullets）

每个列表项都以加粗短语开头。Claude 和 ChatGPT 的 Markdown 输出中极其常见。几乎没有人手写列表时会这样格式化。

坏例子：
```markdown
- **Security**: Environment-based configuration with...
- **Performance**: Lazy loading of expensive resources...
- **Scalability**: Horizontal scaling through message queues...
```

修正方式：
- 如果列表项够短，不需要加粗任何东西
- 如果需要标题，用子标题（###）而不是加粗开头
- 如果确实需要 key-value 结构，用表格

例外：技术文档中的参数说明、API 字段说明等天然的 key-value 结构可以保留。

## Unicode 装饰

使用标准键盘不容易打出的 Unicode 字符：箭头（→）、花括号引号（"..."）、特殊符号。真人在文本编辑器里打字会产出直引号和 -> 或 =>。

坏例子：
- "Input → Processing → Output"
- ""Smart quotes" instead of straight quotes"

修正方式：
- 用 `->` 或 `=>` 替代 `→`
- 用直引号替代花括号引号
- 在技术文档和 ASCII 图中保持纯 ASCII（ASCII 图 skill 有专门的字符集规则，那是例外）

## 格式的正确用法

格式是工具，不是装饰。使用标准：

| 格式 | 用在 | 不用在 |
| --- | --- | --- |
| **加粗** | 读者可能跳读时的关键术语（每段最多 1 处） | 每个列表项开头、每个段落的第一句 |
| *斜体* | 外来词、书名、首次引入的术语 | 强调（用句子结构来强调） |
| `代码` | 代码、命令、文件名 | 普通词汇的强调 |
| 列表 | 并列的 3 个以上同类项 | 本应是段落的论证 |
| 标题 | 话题切换 | 每 2-3 段就加一个 |

## 自查规则

1. 数破折号——超过 3 个就逐个检查能否替换
2. 检查列表项是否都以加粗开头——如果是，重新评估格式选择
3. 搜索 → 和花括号引号——替换成键盘能直接打出的字符
4. 检查加粗使用频率——如果超过每 300 字 1 处，过多了
