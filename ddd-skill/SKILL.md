---
name: ddd-skill
description: >
  Shape stable business slices and messy module or service boundaries into BDD-first
  DDD artifacts. Use when the main question is domain ownership, bounded contexts,
  aggregates, invariants, scenario clustering, or legacy boundary redesign. Start
  with a readiness check or explore output when the slice is still fuzzy. Route pure
  REST or OpenAPI contract work to restful-api-design, language-specific code shaping
  to idiomatic-go or modern-python, and packaging or PR flow to commit.
---

# BDD-First Domain Modeling

Use this skill to move from `Given / When / Then` scenarios to boundaries, domain language, and implementation slices without drifting into table-first or layer-first design.

## Routing

### Use this skill when

- 输入已经是一个稳定或基本稳定的能力切片，或一个明确的现有模块 / service / 边界评审对象。
- 主要问题是：规则归属、边界划分、上下文协作、聚合一致性、场景聚类、重构方向。
- 需要把业务语言和测试入口一起拉齐，而不是只产出技术层拆分意见。

### Stop early or route upstream

- 如果输入仍是大而宽的 PRD、多个能力簇混在一起、关键场景缺失，不要硬产出聚合或上下文图。
- 先产出 `Pre-DDD readiness check`，明确缺什么、下一刀切哪块、该回到哪个上游阶段。
- 如果只是模糊地“想建模一下”，但还说不清主要 actor、关键结果和范围，先停在 `Explore mode` 或 readiness 输出。

### Hand off sideways or downstream

- 纯 REST / OpenAPI / URI / Problem JSON / 版本策略 → [[01_Area/Vibe_Coding/skills/restful-api-design/SKILL.md|restful-api-design]]
- 语言与代码结构已进入落地阶段，主要是 Go 包边界 / Python 模块边界 / 实现风格 → [[01_Area/Vibe_Coding/skills/go_skill/SKILL.md|idiomatic-go]] 或 [[01_Area/Vibe_Coding/skills/python_skill/SKILL.md|modern-python]]
- 已完成设计并进入 commit / PR 打包 → [[01_Area/Vibe_Coding/skills/commit/SKILL.md|commit]]

## Artifact Contract

| Item | Meaning |
| --- | --- |
| Expects In | `stable slice brief`, `scenario set`, 或 `existing module/service slice + entrypoints + current pains` |
| Produces Out | `Pre-DDD readiness check`, `Explore note`, `Lean delivery design`, `Full design doc`, `Review / refactor boundary report`, `Quick boundary cut note` |
| Next Hop | API / use-case design, code shaping, implementation slicing, commit / PR flow |

## Core Stance

- `BDD` 是故事风暴的最小工作单元，不是最后补上的验收附件。
- `TDD` 是边界探针，不只是写测试的节奏。
- `DDD` 负责把稳定业务语言、规则归属和系统边界组织起来。
- 默认选最轻量但足够的输出：能用轻量模式解决，就不要机械展开全套 DDD 名词。

## Forward Path

适用于 greenfield / MVP / 新切片建模。

1. 先做 readiness 判断：这是不是一个可建模的 slice？
2. 如果 slice 还在收敛，但已经能写出几个关键场景，先走 `Explore mode`。
3. 如果是窄范围 MVP / 小模块 / 小功能，优先走 `Lean delivery mode`。
4. 如果是核心域、跨边界协作复杂、或一致性风险高，再升级到 `Full design mode`。
5. 产出后明确 handoff：是去 API contract、代码结构、实现切片，还是先回上游补输入。

## Reverse Path

适用于 legacy / review / refactor / service split。

1. 先写 **current-state scenario(s)**：今天系统如何工作、痛点在哪、哪里最难改。
2. 再写 **target-state scenario(s)**：希望留下什么行为、修掉什么边界问题。
3. 用 current vs target gap 判断要保留、重塑、拆分还是合并。
4. 先找 **smallest safe migration slice**，不要直接给“大重构蓝图”。
5. 产出后明确 handoff：进入语言特定重构、API 契约调整，或继续做更聚焦的 boundary cut。

## Before You Model

- 如果输入仍是大而宽的 PRD、跨多个能力簇、边界明显未稳，不要直接产出聚合和上下文。
- 先判断当前输入是否已经满足最小建模前提：
  - scope 是否已收敛到一个稳定能力切片
  - 关键场景是否覆盖主路径和关键失败路径
  - 相邻能力是否已经基本区分
  - 是否存在明显的 completeness / orthogonality / minimality 风险
- 如果还不满足，先输出 `Pre-DDD readiness check`，使用 [templates/pre-ddd-readiness-template.md](templates/pre-ddd-readiness-template.md)，明确：
  - 建议先拆哪几个能力边界
  - 哪些场景仍缺失
  - 哪些词汇 / 规则还在混用
  - 推荐回到哪个上游阶段或工件

## Language Policy

- 默认跟随用户当前使用的语言。
- 如果输入是双语的，优先跟随用户最近一次明确使用的语言，同时保留必要的领域术语原文。
- 不要为了形式统一而强行翻译稳定领域词；先定义，再保持一致。

## Decision Order

1. 先澄清业务结果、角色和范围。
2. 先判断是 `forward path` 还是 `reverse path`。
3. 先选最轻量但足够的模式，再写场景。
4. 先写 BDD 场景，再做故事风暴提取。
5. 从每个场景中提取 command、observable result、invariant、boundary guess、test seam；只有在需要时再升级到更重提取。
6. 按共享语言、共享不变式和共享失败语义，把场景聚成边界候选。
7. 用 TDD 判断边界：同一公开接口和同一不变式应尽量留在一个边界里；需要不同 fixture、不同外部依赖、不同词汇的行为应考虑拆开。
8. 最后再落到限界上下文、聚合、值对象、领域服务和仓储。
9. 输出时显式写明当前阶段、产物类型、主要决定、下一跳、开放问题和停止条件。

## Intake Rules

If the user has not provided enough detail, first ask for the minimum modeling inputs:

- source stage / artifact source
- business goal
- primary actor
- key scenarios
- observable success / failure outcomes
- scope / non-goals
- external systems or dependencies
- whether the task is greenfield / MVP, review, or refactor
- current public entrypoints / code artifacts (if reviewing existing code)
- upstream split status: stable slice / partially split / still fuzzy

Do not jump straight to aggregates, repositories, or layers when the scenario is still unclear.

## Modes

Always choose the lightest mode that preserves the decision.

| Stage / Mode | Use when | Main artifact | Template / subset |
| --- | --- | --- | --- |
| Pre-DDD readiness gate | 输入还不够稳定，无法可靠判断边界 | `Pre-DDD readiness check` | [templates/pre-ddd-readiness-template.md](templates/pre-ddd-readiness-template.md) |
| Explore mode | slice 还在收敛，但已能写出少量关键场景 | `Explore note` | 用 [templates/user-story-template.md](templates/user-story-template.md) 作为工作画布；输出使用 [templates/output-template.md](templates/output-template.md) 的最小子集：`0, 1, 2, 3, 11, 12` |
| Lean delivery mode | MVP / 小模块 / 窄范围 greenfield，重点是先做对一小刀 | `Lean delivery design` | 使用 [templates/output-template.md](templates/output-template.md) 的轻量子集：`0, 1, 2, 3, 6, 7, 9, 11, 12` |
| Full design mode | 核心域、高风险跨边界协作、需要完整上下文图与测试策略 | `Full design doc` | 完整使用 [templates/output-template.md](templates/output-template.md) |
| Review / refactor mode | 既有系统评审、service split、模块重塑、legacy redesign | `Review / refactor boundary report` | 输入用 [templates/review-input-template.md](templates/review-input-template.md)，输出用 [templates/review-output-template.md](templates/review-output-template.md) |
| Quick boundary cut mode | 聚焦问题，例如“X 要不要单独成聚合 / 上下文？” | `Quick boundary cut note` | [templates/quick-boundary-cut-template.md](templates/quick-boundary-cut-template.md) |

Every final output should end with the same closing fields:
- `Current Stage`
- `Artifact Type`
- `Primary Decision`
- `Recommended Next Skill / Next Stage`
- `Open Questions`
- `Stop Conditions / Do not proceed until`

## References by Task

Load only what the current task needs.

- BDD 场景驱动、mode depth、upgrade trigger、current vs target scenario 写法：读 [references/story-storming.md](references/story-storming.md)
- 深模块、信息隐藏、命名与模块评审：读 [references/design-philosophy.md](references/design-philosophy.md)
- 用 TDD 划边界、测试策略、mock 与数据库策略：读 [references/testing-strategy.md](references/testing-strategy.md)
- 执行切片与实现落地：读 [references/implementation-guide.md](references/implementation-guide.md)

## Modeling Guardrails

- 先场景，后对象。
- 先行为和结果，后层和表。
- 同一个 `When` 要尽量对应一个清晰命令。
- `Then` 先写可观察结果，再判断是否需要内化为领域事件。
- 不要把 `bounded context = microservice`。
- 共享同一不变式和事务一致性要求的行为，优先留在同一边界里。
- Application Service 负责编排，不负责拥有业务规则。
- 避免 `router -> service -> repository` 这种只转发不增量抽象的薄层链路。
- 如果能力切片本身还不稳定，先停在上游拆分，不要硬产出聚合图。
- 如果问题本质上已经变成 API contract、语言实现细节或 commit 打包，就切给相邻 skill，不要留在这里硬答。

## Testing Guardrails

- 测试行为和业务结果，不测试内部调用路径。
- Mock 系统边界，不 mock 内部层级。
- Repository 优先用内存库、临时真实库或 Testcontainers 验证真实行为。
- 如果一个测试为了隔离某个规则必须 mock 一串内部层，通常说明边界划分有问题。
- 如果内部重构会迫使场景测试大改，说明公开接口泄漏了实现细节。
- `Then` 如果暗含立即可见性、唯一性、顺序保证或 eventual consistency 容忍度，要显式写出来，再决定边界和测试。

## Anti-Patterns To Avoid

- 不要先给表结构，再倒推领域边界。
- 不要先画分层，再硬塞业务语义进去。
- 不要把 application service 当成业务规则容器。
- 不要为了形式完整机械产出所有 DDD 名词。
- 不要把每个 `Then` 都翻译成领域事件。
- 不要把 review / refactor 问题强行扩写成完整新系统设计。
- 不要把上游能力拆分问题，伪装成聚合命名问题。
- 不要用完整 DDD 文档去回答一个本质上只是 API、代码风格或 commit 流程的问题。

## Task Routing

| Request type | Route | Load first | Default mode | Notes |
| --- | --- | --- | --- | --- |
| Broad / fuzzy PRD | 先做 readiness gate | `story-storming` | Pre-DDD readiness gate | 先判断是否回到上游 capability / spec split |
| Fuzzy but local slice | 先探索，再决定是否升级 | `story-storming` | Explore mode | 目标是收敛问题，不是立刻画完整上下文 |
| Narrow MVP feature | 先做够用的边界与测试 seam | `story-storming`, `testing-strategy` | Lean delivery | 只保留最小但可交付的设计面 |
| Greenfield core domain | 需要完整上下文图、协作关系和一致性判断 | `story-storming`, `testing-strategy` | Full design | 先场景，再抽边界 |
| Design review / module redesign | 先找边界问题和 pass-through 层 | `design-philosophy`, `testing-strategy` | Review / refactor | 用 current vs target scenarios 对比 |
| Module or service split | 想拆 service、拆模块、拆上下文 | `testing-strategy`, `design-philosophy` | Review / refactor | 用公开测试入口和不变式回推 |
| Focused boundary question | “X 要不要单独成聚合 / 上下文？” | `story-storming`, `testing-strategy` | Quick boundary cut | 只回答必要边界和权衡 |
| Pure API contract design | 路由到 sibling skill | `restful-api-design` | n/a | 这里最多提供 handoff，不展开 HTTP 细节 |
| Go / Python code shaping | 路由到 sibling skill | `idiomatic-go` / `modern-python` | n/a | 先继承本 skill 的 boundary 结论，再做代码结构 |
| Commit / PR packaging | 路由到 sibling skill | `commit` | n/a | 这里不处理 git 打包 |

## Templates and References

- Lean / default intake template: [templates/user-story-template.md](templates/user-story-template.md)
- Review intake template: [templates/review-input-template.md](templates/review-input-template.md)
- Input example: [templates/input-example.md](templates/input-example.md)
- Pre-DDD readiness template: [templates/pre-ddd-readiness-template.md](templates/pre-ddd-readiness-template.md)
- Full / lean / explore output template: [templates/output-template.md](templates/output-template.md)
- Review output template: [templates/review-output-template.md](templates/review-output-template.md)
- Quick boundary cut template: [templates/quick-boundary-cut-template.md](templates/quick-boundary-cut-template.md)
- Full design example: [templates/output-example.md](templates/output-example.md)
- Story storming reference: [references/story-storming.md](references/story-storming.md)
- Design philosophy reference: [references/design-philosophy.md](references/design-philosophy.md)
- Testing strategy reference: [references/testing-strategy.md](references/testing-strategy.md)
- Execution guide: [references/implementation-guide.md](references/implementation-guide.md)
