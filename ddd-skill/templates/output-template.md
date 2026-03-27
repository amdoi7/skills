# 领域建模输出模板

> `Full design mode` 使用完整模板。`Explore mode` 与 `Lean delivery mode` 只使用文中指定子集，不要为了形式保留空章节。

## Mode subsets

- `Explore mode`: `0, 1, 2, 3, 11, 12`
- `Lean delivery mode`: `0, 1, 2, 3, 6, 7, 9, 11, 12`
- `Full design mode`: 使用全部章节

## 0. 上游边界与建模前提

- 输入来源：
- 本次建模切片：
- 已明确不纳入的相邻能力：
- 上游拆分假设：
- Completeness / Orthogonality / Minimality 风险：
- 若这些前提不成立，先回到上游规格拆分

## 1. 范围与业务目标

- 本次范围：
- 非目标：
- 核心业务结果：
- 为什么现在值得建模这块：

## 2. BDD 场景矩阵

| Scenario | Given | When | Then | 备注 |
| --- | --- | --- | --- | --- |
| [标题] | [前置] | [动作] | [结果] | [备注] |

## 3. 场景到故事风暴提取

| Scenario | Actor | Command | Observable Result | Candidate Domain Event(s) | Invariants / Policies | External Boundaries | Open Questions |
| --- | --- | --- | --- | --- | --- | --- | --- |
| [标题] | [角色] | [命令] | [结果] | [事件] | [规则] | [边界] | [问题] |

## 4. 统一语言

| 术语 | 定义 | 所属上下文 | 备注 |
| --- | --- | --- | --- |
| [术语] | [定义] | [上下文] | [备注] |

## 5. 限界上下文地图与协作方式

| Context | Core Language | Responsibilities | Relationship Pattern | Translation / ACL | Notes |
| --- | --- | --- | --- | --- | --- |
| [上下文] | [核心术语] | [职责] | [ACL / Customer-Supplier / OHS / Conformist ...] | [翻译层 / published language] | [备注] |

## 6. TDD 边界划分

| 行为 / 场景簇 | 最自然的公开测试入口 | 共享不变式 / 一致性 | 建议边界 | 划分理由 |
| --- | --- | --- | --- | --- |
| [行为] | [接口] | [规则] | [上下文 / 聚合 / gateway] | [原因] |

## 7. 限界上下文与聚合

### [上下文名]

- 职责：
- 与其他上下文关系：
- 为什么是一个独立边界：

### [聚合根名称]

- 职责：
- 边界理由：
- 不变式：
- 处理命令：
- 产生事件：
- 实体：
- 值对象：
- 不放进聚合的内容：

## 8. 服务、仓储与外部边界

| 类型 | 名称 | 职责 | 为什么在这里 |
| --- | --- | --- | --- |
| Application Service / Domain Service / Repository / Gateway | [名称] | [职责] | [边界理由] |

## 9. TDD / 测试策略

- 场景测试：
- 领域行为测试：
- 持久化测试：
- 需要 mock 的系统边界：
- 不建议 mock 的内部层：
- 一致性 / 可见性提醒：

## 10. 复杂度与设计复盘

- 深模块机会：
- 信息泄漏风险：
- 需要避免的 pass-through 层：
- 命名 / 注释提醒：
- 不该做的事情：

## 11. 实现 / 交付切片

1. 最小安全第一刀：
2. 第二个切片：
3. 第三个切片：

## 12. Closing Contract

- Downstream consumer(s):
- Must-preserve assumptions:
- Current Stage:
- Artifact Type:
- Primary Decision:
- Recommended Next Skill / Next Stage:
- Open Questions:
- Stop Conditions / Do not proceed until:
