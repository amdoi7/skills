# 示例输入：BDD / 领域建模素材

## 0. 建模前提

- Source stage: domain tree slice
- Upstream split status: stable slice
- Existing public entrypoint(s) if any: `PlaceOrder`
- Existing modules / services / documents if any: 订单域 PRD 子域说明、库存服务契约、优惠券服务契约

## 1. Intake

- Task type: greenfield modeling
- Business goal: 让顾客能够从购物车提交订单，并得到明确成功或失败结果。
- Primary actor: 顾客
- Observable success outcome: 创建订单并返回订单编号。
- Observable failure outcome: 返回明确失败原因且不创建订单。
- Scope: 本次只覆盖下单，不覆盖支付、发货和售后。
- Non-goals: 推荐、营销投放、客服流程。
- External systems or dependencies: 库存服务、优惠券服务、通知服务。

## 1.1 Boundary Risks

- Suspected overlap with adjacent capabilities: 支付、发货、营销不在本次建模切片内。
- Suspected completeness gaps: 订单成功后的通知副作用仍需后续补充。
- Suspected orthogonality / ownership conflicts: Promotion 只提供折扣结果，不拥有 Ordering 的订单语义。
- Why this is the right slice to model now: “提交订单”已经具备清晰入口、主路径和关键失败路径。

## 2. BDD 场景切片

### Scenario 1: 库存充足时成功下单

Given 顾客购物车中有 2 件商品且库存充足  
And 顾客使用一张有效优惠券  
When 顾客提交订单  
Then 系统创建订单并返回订单编号  
And 订单金额等于原价减去优惠金额

#### 场景提取

- Actor：顾客
- Command：提交订单
- Observable Result：订单创建成功并返回订单编号
- Candidate Domain Event(s)：订单已提交、库存已预留、优惠券已应用
- Invariants：订单至少一个行项；折扣不能超过原价；库存不能为负
- Policies / Follow-ups：订单成功后可异步发送通知
- External Boundaries：库存服务、优惠券服务
- Open Questions：库存预留是在订单创建前还是创建后确认？

### Scenario 2: 库存不足时拒绝下单

Given 顾客购物车中某商品数量超过可售库存  
When 顾客提交订单  
Then 系统拒绝创建订单并返回库存不足原因  
And 购物车内容保持不变

#### 场景提取

- Actor：顾客
- Command：提交订单
- Observable Result：下单失败且返回库存不足
- Candidate Domain Event(s)：订单创建失败、库存预留被拒绝
- Invariants：库存不足时不能创建订单
- Policies / Follow-ups：保留购物车，允许顾客调整后重试
- External Boundaries：库存服务
- Open Questions：库存不足需要多细粒度地暴露给前端吗？

## 3. 统一语言草案

| 术语 | 定义 | 备注 / 所属边界 |
| --- | --- | --- |
| 订单 | 顾客一次购买请求及其生命周期 | Ordering |
| 可售库存 | 可用于新订单的库存数量 | Inventory |
| 优惠券 | 可作用于订单金额的折扣凭证 | Promotion |

## 4. Example Mapping

### Rules

- 库存不足时不能创建订单。
- 优惠券折扣不能超过订单原价。

### Examples

- 2 件商品库存充足，订单成功。
- 5 件商品库存只剩 3 件，订单失败。

### Questions

- Promotion 与 Ordering 之间传“优惠券对象”还是只传“折扣结果”？

## 5. TDD 边界提示

- “提交订单”的主路径和失败路径共享同一公开入口：`PlaceOrder`。
- 订单金额与折扣约束应由同一领域边界守住。
- 库存规则需要不同 fixture 和库存词汇，倾向拆到独立边界。

## 6. Existing System Signals（可选，用于 review / refactor）

- Current boundary pain: 暂无，当前为 greenfield 建模。
- Change amplification hotspots: 暂无。
- Pass-through layers / duplicated rules: 需要避免在 API、应用服务和仓储重复校验库存不足。
- Existing test entrypoints: `PlaceOrderUseCase.execute()`。
- Refactoring constraints: 先完成下单最小闭环，再扩到后续流程。
