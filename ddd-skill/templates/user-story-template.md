# Lean / Default BDD-Domain Intake Template

> 默认走轻量输入。只有当 slice 真的变复杂时，再升级到更重提取。

## 0. 建模前提

- Source stage: broad PRD / domain tree slice / existing code review
- Task type: explore / lean delivery / full design / review / quick boundary cut
- Upstream split status: stable slice / partially split / still fuzzy
- Existing public entrypoint(s) if any:
- Existing modules / services / documents if any:

## 1. Intake

- Business goal:
- Primary actor:
- Observable success outcome:
- Observable failure outcome:
- Scope:
- Non-goals:
- External systems or dependencies:
- Why this is the right slice to model now:

## 2. BDD 场景切片

### Scenario 1: [标题]

Given [前置条件]
And [补充前置条件]
When [触发动作]
Then [可观察结果]
And [补充结果]

#### Lean extraction

- Command:
- Invariant:
- Boundary Guess:
- Test Seam:
- Open Question:

### Scenario 2: [标题]

Given [前置条件]
When [触发动作]
Then [可观察结果]

#### Lean extraction

- Command:
- Invariant:
- Boundary Guess:
- Test Seam:
- Open Question:

## 3. 统一语言草案

| 术语 | 当前含义 | 备注 / 边界猜测 |
| --- | --- | --- |
| [术语] | [明确含义] | [是否有歧义] |

## 4. Upgrade only when needed

只有出现词汇冲突、跨边界规则、非自然测试 seam、或一致性风险时，再补下面这些字段。

### Optional deeper extraction per scenario

- Actor:
- Observable Result:
- Candidate Domain Event(s):
- Policies / Follow-ups:
- External Boundaries:

### Optional example mapping

#### Rules

- [规则]

#### Examples

- [具体业务例子]

#### Questions

- [待澄清问题]

### Optional TDD boundary prompts

- 哪些场景共享同一公开接口？
- 哪些场景共享同一不变式或事务边界？
- 哪些行为一旦测试就需要不同的 fixture 或外部系统？
- 哪些步骤如果必须 mock 内部层，说明边界可能切错了？

## 5. Existing System Signals（可选）

- Current boundary pain:
- Change amplification hotspots:
- Pass-through layers / duplicated rules:
- Existing test entrypoints:
- Refactoring constraints:
