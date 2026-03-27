# Review / Refactor Boundary Report

## 1. Scope

- Reviewed slice:
- Business goal:
- Non-goals:
- Why this slice now:

## 2. Current-State Scenarios

| Scenario | Current Entry | Current Result | Pain Today |
| --- | --- | --- | --- |
| [场景] | [入口] | [当前结果] | [问题] |

## 3. Target-State Scenarios

| Scenario | Target Entry / Boundary | Target Result | Why Better |
| --- | --- | --- | --- |
| [场景] | [目标入口 / 边界] | [目标结果] | [原因] |

## 4. Gap Analysis

| Gap | Current State | Target State | Why It Matters | Suggested Move |
| --- | --- | --- | --- | --- |
| [差距] | [当前] | [目标] | [影响] | [动作] |

## 5. Current Boundary Problems

| Symptom | Where It Shows Up | Why It Hurts | Severity |
| --- | --- | --- | --- |
| [问题] | [位置] | [影响] | [高/中/低] |

## 6. Context Map Impact

| Context / Module | Keep / Reshape / Split / Merge | Relationship Pattern | ACL / Translation Need | Notes |
| --- | --- | --- | --- | --- |
| [上下文] | [动作] | [关系] | [需要什么翻译层] | [备注] |

## 7. Boundary Recommendations

- What should stay together:
- What should move out:
- What should become a gateway / adapter:
- What should not become a separate aggregate or context:
- Safest area to change first:

## 8. Testing Strategy Feedback

- Stable scenario entrypoints:
- Domain behaviors that need direct tests:
- System boundaries that should be mocked:
- Internal layers that should stop being mocked:
- Characterization tests to add before moving anything:

## 9. Smallest Safe Migration Slice

1. First slice:
2. Second slice:
3. Third slice:

## 10. Closing Contract

- Downstream consumer(s):
- Must-preserve assumptions:
- Risks to monitor during implementation:
- Current Stage:
- Artifact Type:
- Primary Decision:
- Recommended Next Skill / Next Stage:
- Open Questions:
- Stop Conditions / Do not proceed until:
