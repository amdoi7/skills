# 用 BDD 驱动故事风暴（渐进式版）

## 核心主张

故事风暴的最小工作单元不是“事件便签”，而是一个 BDD 场景：

```text
Scenario: [标题]
Given ...
When ...
Then ...
```

先把场景写清楚，再从场景里提取命令、规则、边界和测试 seam，能更稳定地把业务语言、可观察结果和后续实现串起来。

## 先过 readiness gate

如果当前输入还说不清下面几件事，就先回到 `Pre-DDD readiness`，不要硬做聚合设计：

- 这次到底在建模哪个能力切片
- 主 actor 是谁
- 关键成功 / 失败结果是什么
- 相邻能力是否已经基本区分
- 至少能不能写出 1–2 个关键场景

## 模式选择

| 模式 | 适用场景 | 最小工作量 | 常见产物 |
| --- | --- | --- | --- |
| Explore | slice 还在收敛，但已经能写出少量关键场景 | 写少量场景 + 轻量提取 | Explore note |
| Lean delivery | MVP / 小模块 / 小切片 | 写主路径 + 关键失败路径 + 最小边界判断 | Lean delivery design |
| Full design | 核心域 / 高风险跨边界协作 | 完整场景矩阵、上下文图、测试策略 | Full design doc |
| Review / refactor | 既有系统评审或重构 | current-state + target-state scenarios + gap | Review / refactor report |
| Quick boundary cut | 只问一个聚焦边界问题 | 1 个核心场景 + 共享不变式判断 | Quick boundary cut note |

## 提取深度：先轻后重

### Explore depth

先提取最小判断面：

- Command
- Invariant
- Boundary Guess
- Test Seam
- Open Question

适合用来判断“这是不是一个稳定 slice”“这条规则更像谁的责任”。

### Lean depth

在 Explore 的基础上，补这些信息：

- Actor
- Observable Result
- External Boundaries
- Minimal implementation slice

适合 MVP、小模块、小功能，不要求一次把所有 DDD 名词补齐。

### Full depth

只有在边界复杂度真的上来时，再补完整提取：

- Actor
- Command
- Observable Result
- Candidate Domain Event(s)
- Invariants
- Policies / Follow-ups
- External Boundaries
- Open Questions
- Context map / ACL notes
- Test strategy implications

## Forward delivery flow

### 1. 先切场景

- 先写 happy path。
- 再写关键异常、边界条件、失败路径。
- 每个场景只描述一个清晰意图。

### 2. 从 `When` 提取命令

- 命令应该表达业务意图，而不是技术动作。
- `提交订单` 比 `调用下单接口` 更好。

### 3. 从 `Then` 提取结果

- 先写用户或外部系统能观察到的结果。
- 再判断它是否需要成为领域事件。
- 不是每个 `Then` 都必须变成事件。

### 4. 从 `Given` 提取约束

- 哪些前置条件属于状态合法性？
- 哪些只是策略或配置？
- 哪些前置条件其实来自另一个边界？

### 5. 用场景聚边界

- 共享同一不变式和同一公开接口的场景，应优先归到同一边界。
- 需要不同外部依赖、不同 fixture、不同词汇的场景，通常应拆到不同边界。
- 先得到 context 候选，再在 context 内切 aggregate。

### 6. 用 TDD 验证边界

- 先为场景选一个最自然的公开测试入口。
- 如果测试需要 mock 很多内部层，边界通常太碎。
- 如果一个测试要初始化半个系统才能验证一条规则，边界通常太大。

## Reverse refactor flow

### 1. 先写 current-state scenario(s)

不要先看类图或包图，先写：

- 今天系统在什么入口处理这个行为
- 现在的可观察结果是什么
- 痛点在哪里暴露
- 哪些地方让改动放大

### 2. 再写 target-state scenario(s)

目标不是“换个更漂亮的架构”，而是明确：

- 哪些业务结果必须保留
- 哪些边界问题必须消失
- 哪些协作方式要改
- 哪些外部契约必须暂时保持不变

### 3. 对比 gap

至少回答四件事：

- 当前规则归属错在哪
- 目标归属应该怎么变
- 中间需要什么翻译层或过渡层
- 最小安全迁移切片是什么

### 4. 先找 smallest safe migration slice

优先选择：

- 可以用 characterization test 护住的切口
- 能减少 change amplification 的切口
- 不会一次牵动多个上下文的切口

## Upgrade triggers

出现下面信号时，从 Explore / Lean 升级到更重模式：

| Trigger | 说明 | 升级方向 |
| --- | --- | --- |
| Repeated vocabulary conflicts | 同一个词在不同场景里语义不一致 | 补统一语言与 context map |
| One rule crosses boundary guesses | 同一条规则在多个边界猜测里来回出现 | 升级到 Full design 或 Review / refactor |
| Non-natural test seam | 找不到自然公开测试入口，只能 mock 一串内部层 | 重做边界判断，补 TDD 边界划分 |
| Consistency-sensitive `Then` | `Then` 暗含立即可见性、顺序、唯一性或延迟容忍 | 显式加一致性说明，必要时升级到 Full design |
| Migration risk is unclear | 不知道第一刀该切哪、回滚怎么做 | 走 Review / refactor 路径 |

## Open-question management

- 每个场景都允许保留 `Open Question`，不要把不确定性藏进模糊措辞。
- 开放问题要尽量贴场景写，不要集中堆在文末。
- 如果问题会阻断规则归属判断，要把它写进 `Stop Conditions / Do not proceed until`。
- 如果只是细节问题，不要因为它们阻塞 lean 方案。

## 一致性 / 可见性提示（重点看 `Then`）

`Then` 里如果隐含下面约束，要显式写出来：

- **Immediate visibility**：结果是不是要立即可见，还是允许异步到账
- **Ordering**：多个动作是否有先后保证
- **Uniqueness**：是不是必须防重、去重、全局唯一
- **Eventual consistency tolerance**：用户或下游系统能否接受短暂不一致

这些约束会直接影响：

- 边界要不要放在一起
- 需要什么测试 seam
- 能不能拆成异步协作
- review / refactor 时第一刀能切在哪

## 输出检查

- 每个场景是否都有清晰的业务结果？
- `When` 是否能映射到一个清晰命令？
- 当前模式是不是已经足够，还是还在机械补全？
- 关键 `Then` 是否写明了一致性 / 可见性要求？
- review / refactor 时，是否同时写了 current-state 与 target-state？
- 是否已经显式记录开放问题、下一跳和停止条件？

## 常见误区

- 先列对象、表和接口，再硬补场景。
- 把所有 `Then` 都翻译成领域事件。
- 用技术层而不是场景与不变式来切边界。
- 把 BDD 场景写成实现步骤清单。
- 明明只是 MVP 小切片，却默认展开完整 DDD 文档。
- 明明是 legacy 重构，却跳过 current-state / target-state 对比，直接给“理想新架构”。
