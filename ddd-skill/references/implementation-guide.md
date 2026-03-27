# 实现指南

## 目录组织建议

优先按限界上下文或业务能力组织，再在上下文内部放领域、应用和基础设施细节。

```text
src/
  ordering/
    domain/
      order.py
      order_item.py
      value_objects.py
      events.py
    application/
      place_order.py
    infrastructure/
      order_repository.py
      inventory_gateway.py
      promotion_gateway.py
    interfaces/
      http/
        place_order_endpoint.py

tests/
  scenarios/
    test_place_order_api.py
  domain/
    test_order_behavior.py
  integration/
    test_order_repository_sqlite.py
```

## 场景到边界的实施顺序

1. 先写一个 `Given / When / Then` 场景。
2. 为这个场景选一个最自然的公开测试入口。
3. 写第一个失败测试。
4. 如果测试需要大量无关状态或开始 mock 内部层，就回头重切边界。
5. 用聚合和值对象把共享不变式收回模型内部。
6. 让应用服务只做编排，把系统耦合压到 gateway / adapter。

## 测试默认值

- 场景测试：测业务结果与错误语义。
- 聚合和值对象：直接测公开行为和领域事件。
- Repository：优先内存数据库或临时真实数据库。
- 外部系统：只 mock 支付、短信、邮件、对象存储等真正的系统边界。

## 设计校验问题

1. 这个场景是否能通过一个稳定公开接口自然表达？
2. 哪些规则在多个场景中重复出现，说明它们应该内聚在同一边界？
3. 这一层是否真的提供了新的抽象，还是只是 pass-through？
4. 如果内部重构，现有场景测试是否大体不需要改？
