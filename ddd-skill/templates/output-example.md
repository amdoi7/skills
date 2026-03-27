# 示例输出：BDD 驱动的领域建模与交付设计文档

## 0. 上游边界与建模前提

- 输入来源：订单子域的 Domain Tree 切片。
- 本次建模切片：顾客提交订单。
- 已明确不纳入的相邻能力：支付、发货、售后、营销推荐。
- 上游拆分假设：Ordering、Inventory、Promotion 已经是可区分的能力边界。
- Completeness / Orthogonality / Minimality 风险：通知副作用尚未建模，但不影响下单最小闭环。
- 若这些前提不成立，先回到上游规格拆分。

## 1. 范围与业务目标

- 本次范围：顾客提交订单。
- 非目标：支付、发货、售后。
- 核心业务结果：在库存与优惠规则满足时创建订单，否则返回明确失败原因。

## 2. BDD 场景矩阵

| Scenario | Given | When | Then | 备注 |
| --- | --- | --- | --- | --- |
| 库存充足时成功下单 | 购物车有可售商品且优惠券有效 | 顾客提交订单 | 系统创建订单并返回订单编号 | 主路径 |
| 库存不足时拒绝下单 | 商品数量超过可售库存 | 顾客提交订单 | 系统返回库存不足且不创建订单 | 关键异常 |

## 3. 场景到故事风暴提取

| Scenario | Actor | Command | Observable Result | Candidate Domain Event(s) | Invariants / Policies | External Boundaries | Open Questions |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 库存充足时成功下单 | 顾客 | 提交订单 | 创建成功并返回订单编号 | 订单已提交、库存已预留 | 订单至少一个行项；折扣不能超过原价 | Inventory、Promotion | 库存预留在前还是在后 |
| 库存不足时拒绝下单 | 顾客 | 提交订单 | 创建失败并返回库存不足 | 订单创建失败、库存预留被拒绝 | 库存不足时不能创建订单 | Inventory | 失败原因要细到什么程度 |

## 4. 统一语言

| 术语 | 定义 | 所属上下文 | 备注 |
| --- | --- | --- | --- |
| 订单 | 顾客一次购买请求及其生命周期 | Ordering | 不等同于支付单 |
| 可售库存 | 可用于新订单的库存数量 | Inventory | 不是总库存 |
| 优惠券 | 对订单金额生效的折扣凭证 | Promotion | 只暴露可用性与折扣结果 |

## 5. 限界上下文地图与协作方式

| Context | Core Language | Responsibilities | Relationship Pattern | Translation / ACL | Notes |
| --- | --- | --- | --- | --- | --- |
| Ordering | 订单、提交订单、下单失败 | 接受下单意图并维护订单生命周期 | Customer-Supplier | 通过 `InventoryGateway` / `PromotionGateway` 接收外部结果，不泄漏上游内部模型 | Ordering 保留自己的失败语义 |
| Inventory | 可售库存、库存预留 | 判断可售库存并执行预留 | OHS | 暴露库存预留契约，不向 Ordering 暴露库存内部对象 | 失败结果需翻译成下单语义 |
| Promotion | 优惠券、折扣结果 | 校验优惠券并返回折扣结果 | ACL | Ordering 只消费折扣结果，不直接引入 Promotion 对象模型 | 避免把促销规则塞回 Order 聚合 |

## 6. TDD 边界划分

| 行为 / 场景簇 | 最自然的公开测试入口 | 共享不变式 / 一致性 | 建议边界 | 划分理由 |
| --- | --- | --- | --- | --- |
| 提交订单主路径与失败路径 | `PlaceOrderUseCase.execute()` | 下单成功 / 失败语义、购物车保留规则 | Ordering 入口边界 | 两个场景共享同一业务入口和结果语义 |
| 订单金额与折扣约束 | `Order.place()` | 总金额大于零、折扣不能超过原价 | Order 聚合 | 同一组不变式应在一个公开领域入口内测试 |
| 库存预留与拒绝 | `InventoryGateway.reserve()` 合同 + 库存上下文行为测试 | 可售库存与预留数量 | Inventory 边界 | 词汇、fixture 和失败语义都属于库存域 |

## 7. 限界上下文与聚合

### Ordering

- 职责：接受下单意图并维护订单生命周期。
- 与其他上下文关系：调用 Inventory 和 Promotion。
- 为什么是一个独立边界：它承载“订单”语言和下单结果语义。

### Order

- 职责：守住订单创建与金额语义。
- 边界理由：订单项、金额和状态变化必须一起保持一致。
- 不变式：至少一个订单项；总金额大于零；折扣不能超过原价。
- 处理命令：提交订单、拒绝订单创建。
- 产生事件：订单已提交、订单创建失败。
- 实体：OrderItem。
- 值对象：OrderId、Money、CouponApplication。
- 不放进聚合的内容：库存规则、优惠券复杂策略。

## 8. 服务、仓储与外部边界

| 类型 | 名称 | 职责 | 为什么在这里 |
| --- | --- | --- | --- |
| Application Service | PlaceOrderUseCase | 编排下单流程和跨上下文调用 | 它组织顺序，但不拥有规则 |
| Repository | OrderRepository | 保存和加载订单聚合 | 订单持久化属于聚合边界 |
| Gateway | InventoryGateway | 调用库存上下文 | 库存是系统边界 |
| Gateway | PromotionGateway | 调用促销上下文并翻译折扣结果 | 防止 Promotion 语言泄漏进 Ordering |

## 9. TDD / 测试策略

- 场景测试：通过 `PlaceOrderUseCase` 或 HTTP 覆盖成功、库存不足、优惠券失效。
- 领域行为测试：直接测试 `Order.place()` 的金额与状态语义。
- 持久化测试：使用内存数据库验证 `OrderRepository` 的映射和约束。
- 需要 mock 的系统边界：库存服务、优惠券服务、通知服务。
- 不建议 mock 的内部层：不要按 `router/service/repository` 链路逐层 mock。

## 10. 复杂度与设计复盘

- 深模块机会：让 `Order` 聚合吸收金额计算和失败语义。
- 信息泄漏风险：库存不足规则不要散落在 API、应用服务和仓储。
- 需要避免的 pass-through 层：只负责转发参数的 service 包装层。
- 命名 / 注释提醒：用“提交订单”“可售库存”这类业务词。
- 不该做的事情：不要把支付、发货和营销逻辑提前塞进下单聚合。

## 11. 实现切片

1. 先实现“库存充足时成功下单”的场景和 `Order.place()` 领域测试。
2. 再实现“库存不足时失败”的场景与错误语义。
3. 最后补优惠券策略和通知副作用。

## 12. Handoff / Next Step

- 这次产出的主要工件：Ordering 上下文设计、Order 聚合边界、Inventory / Promotion 协作契约。
- 推荐下一个阶段：API / use-case 设计与实现切片落地。
- 下游消费者：实现任务卡、接口设计、测试计划。
- 进入代码前仍需澄清的问题：库存预留确认时机、优惠券失效错误语义、通知是否同步触发。
