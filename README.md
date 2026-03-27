# Skills Portfolio Map

## Lifecycle view

```text
upstream idea / spec split
        ↓
ddd-skill
        ↓
restful-api-design
        ↓
idiomatic-go / modern-python
        ↓
commit
```

这份 README 的目标不是列全量 skill，而是先把当前这条交付链讲清楚，让 `ddd-skill` 在 portfolio 里有明确位置。

## What `ddd-skill` is for

[[01_Area/Vibe_Coding/skills/ddd-skill/SKILL.md|ddd-skill]] 负责把稳定业务切片或既有模块边界问题，转成可交接的 DDD 工作流工件。

它最适合回答：
- 这个规则应该归谁？
- 这是不是一个单独的 bounded context / aggregate？
- 这个 legacy service 该怎么拆第一刀？
- 这个 MVP slice 的最小边界和 test seam 是什么？

它不应该吞掉：
- 仍然很宽、还没切稳的上游 spec 问题
- 纯 REST / OpenAPI 合同设计
- 纯 Go / Python 代码风格与包模块重塑
- commit / PR 打包流程

## Upstream stages

在进入 `ddd-skill` 前，最好至少具备下面之一：
- 一个相对稳定的 capability / domain slice
- 一组关键 BDD 场景
- 一个明确要评审的模块 / service / boundary

如果这些还没有，`ddd-skill` 应先产出 readiness check，而不是直接进入 full design。

## Downstream / adjacent skills

### [[01_Area/Vibe_Coding/skills/restful-api-design/SKILL.md|restful-api-design]]

当领域边界和语言已经基本稳定，接下来要把它落成：
- endpoint inventory
- OpenAPI contract
- error model
- versioning / pagination / security rules

这时应从 `ddd-skill` handoff 到 API contract 设计，而不是继续在 DDD 文档里写 HTTP 细节。

### [[01_Area/Vibe_Coding/skills/go_skill/SKILL.md|idiomatic-go]] / [[01_Area/Vibe_Coding/skills/python_skill/SKILL.md|modern-python]]

当主要问题变成：
- 包 / 模块怎么落
- 类型和接口怎么建
- 语言层的边界如何保持清晰
- 并发 / 异常 / 框架边界怎么处理

就应该继承 `ddd-skill` 的边界结论，转到语言特定 skill。

### [[01_Area/Vibe_Coding/skills/commit/SKILL.md|commit]]

当设计和实现都已经完成，需要：
- 收敛 diff
- 拆 commit
- 起草提交信息
- 建 PR

就切到 `commit`，不要继续把 git 工作留在 `ddd-skill` 里。

## Simple routing map

| If the user mainly needs... | Use this skill |
| --- | --- |
| 上游 spec 仍然很宽、切片还不稳定 | upstream split / readiness first |
| 业务边界、规则归属、上下文协作、legacy boundary review | [[01_Area/Vibe_Coding/skills/ddd-skill/SKILL.md|ddd-skill]] |
| REST endpoints / OpenAPI / Problem JSON / HTTP semantics | [[01_Area/Vibe_Coding/skills/restful-api-design/SKILL.md|restful-api-design]] |
| Go 代码结构、包边界、并发与接口设计 | [[01_Area/Vibe_Coding/skills/go_skill/SKILL.md|idiomatic-go]] |
| Python 代码结构、模块边界、异常与 async 设计 | [[01_Area/Vibe_Coding/skills/python_skill/SKILL.md|modern-python]] |
| Commit / PR packaging | [[01_Area/Vibe_Coding/skills/commit/SKILL.md|commit]] |

## Productization notes

这一版先做到三件事：
- 让 `ddd-skill` 更像 workflow node，而不只是“DDD 提示词”
- 让 handoff 字段、stage / artifact vocabulary 更显式
- 让 portfolio routing 更清楚，减少和 sibling skills 打架

暂时不做完整 shared-policy 抽离；目前只轻量保持这些概念的一致性：
- language-following convention
- lightest-viable-output rule
- shared handoff contract fields
- stage / artifact vocabulary
