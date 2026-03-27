# Review / Refactor Intake Template

## 1. Task Framing

- Task type: design review / service split / module refactor
- Business goal:
- Current system / module under review:
- Current public entrypoint(s):
- External systems or dependencies:
- Operational constraints / rollout constraints:
- Why this slice is the right place to start:

## 2. Current vs Desired Behavior

### Current behavior summary

- 今天系统如何完成这件事：
- 当前最痛的地方：
- 当前哪些结果仍然必须保留：

### Desired behavior summary

- 希望变成什么样：
- 想消除哪些边界问题：
- 哪些外部契约暂时不能改：

## 3. Critical Scenarios

### Current-state Scenario 1: [标题]

Given [前置条件]
When [动作]
Then [当前可观察结果]

### Target-state Scenario 1: [标题]

Given [前置条件]
When [动作]
Then [目标可观察结果]

### Current-state Scenario 2: [标题]

Given [前置条件]
When [动作]
Then [当前可观察结果]

### Target-state Scenario 2: [标题]

Given [前置条件]
When [动作]
Then [目标可观察结果]

## 4. Current Boundary Symptoms

- Change amplification hotspots:
- Cognitive load hotspots:
- Pass-through layers / shallow modules:
- Duplicated rules / information leakage:
- Suspected wrong context ownership:
- Safest area to change first:

## 5. Testing and Migration Signals

- Existing scenario / API entrypoints:
- Existing domain tests:
- Existing repository / integration tests:
- Where mocks feel unnatural today:
- Characterization-test candidates:
- Rollback or migration constraints:

## 6. What Must Stay Stable

- Business language that should be preserved:
- Existing contracts that cannot break:
- Data / rollout constraints:
- Monitoring or operational guardrails:
