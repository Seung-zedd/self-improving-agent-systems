# Chapter 02 — ADaPT: As-Needed Decomposition

> **Part:** 04 — Multi-Step Reasoning & Planning  
> **Status:** Study checkpoint — 본문 및 첫 Q&A 완료, 복습 가능

## 1. Problem — 왜 처음부터 모든 task를 분해해야 할까?

LATS가 **어느 trajectory를 탐색할 것인가**를 다룬다면, ADaPT는 **언제 task를 더 작게 분해해야 하는가**를 다룬다.

고정된 plan-and-execute에서는 사전에 decomposition depth를 정해야 한다. 너무 얕으면 executor가 해결하지 못하고, 너무 깊으면 불필요한 planning compute를 소비한다.

ADaPT의 핵심은 다음과 같다.

> **일단 실행하고, 현재 executor가 수행하지 못하면 필요한 부분만 재귀적으로 분해한다.**

> **용어 캡션 — Decomposition:** 복잡한 task를 더 작고 실행 가능한 subtask로 나누는 과정.

## 2. Execute First → Fail → Decompose

```text
Task T
   ↓
Executor
   ↓
Success?
   ├─ YES → Done
   └─ NO  → Decompose(T)
                ├─ T1 → Execute
                └─ T2 → Execute
```

분해한 T1도 해결되지 않으면 T1만 다시 분해한다.

```text
T
├─ T1 → FAIL
│   ├─ T1.1 → SUCCESS
│   └─ T1.2 → SUCCESS
└─ T2 → SUCCESS
```

> **용어 캡션 — Recursive Decomposition:** subtask에도 같은 분해 절차를 반복 적용하는 것.

## 3. Capability-Aware Decomposition

같은 task라도 executor의 능력에 따라 필요한 분해 깊이가 달라질 수 있다.

```text
Stronger Executor
Goal → Task A / Task B → Done

Weaker Executor
Goal → Task A → FAIL → A1 / A2 → Done
```

개념적 추상화:

```math
D = f(\text{Task Complexity},\ \text{Executor Capability})
```

이 식은 논문에서 그대로 인용한 공식이 아니라, 학습 과정에서 정리한 직관이다.

> **용어 캡션 — Atomic Task:** 현재 executor가 추가 분해 없이 수행할 수 있을 정도로 작은 작업. 그 경계는 executor capability에 따라 달라질 수 있다.

## 4. Connecting the dots — Divide and Conquer와 차이

```text
Conventional Divide and Conquer
Divide → Solve subproblems → Combine

ADaPT
Try Solve → Failure → Divide only as needed
```

전통적 divide-and-conquer와 재귀적 분해 구조는 닮았지만, ADaPT에서는 **실행 실패가 추가 분해의 trigger**가 된다.

Part 01의 adaptive compute allocation과도 연결된다.

```text
Part 01: Which candidates deserve more compute?
LATS:    Which trajectories deserve more search?
ADaPT:   Which tasks need deeper decomposition?
```

## 5. LATS vs ADaPT

| | LATS | ADaPT |
|---|---|---|
| Tree 의미 | 가능한 action/trajectory의 탐색 | task/subtask의 분해 |
| 핵심 질문 | 어느 branch를 탐색할까? | 언제 더 분해할까? |
| 적응 대상 | search allocation | decomposition depth |
| 주요 신호 | value, visits, environment feedback | executor의 수행 결과 |

**두 기법의 결합은 별도의 system-level 아이디어**이며, ADaPT 논문 자체의 제안으로 오해하면 안 된다.

---

## 6. ❓ Q&A — ADaPT는 실패를 어떻게 판단하나?

핵심 질문:

> **“실패했으면 분해한다”가 성립하려면, 누가 무엇을 근거로 실패라고 판정하는가?**

실행 환경이 명확한 결과를 반환하는 경우에는 feedback을 사용할 수 있다.

```text
Action → Environment → Observation / Reward / Result
```

예: 문이 잠겨 행동이 거절됨, 테스트가 실패함. 단, **test failure가 반드시 task가 너무 복잡하다는 뜻은 아니다.**

현실적인 열린 과제에서는 판정이 더 어렵다. LLM judge, rubric, human review, external test 등 verifier가 필요할 수 있다.

> **용어 캡션 — Failure Detector / Verifier:** 결과가 요구된 성공 조건을 충족했는지 평가하는 구성요소. 모든 verifier가 동일한 신뢰도를 가지는 것은 아니다.

### False Positive / False Negative

이 Q&A에서는 **Positive = 실패로 판정**이라고 정의한다.

| | 실제 실패 | 실제 성공 |
|---|---|---|
| FAIL 판정 | TP | FP |
| SUCCESS 판정 | FN | TN |

- **FP:** 실제 성공했는데 실패로 판단 → 불필요한 decomposition과 compute 낭비
- **FN:** 실제 실패했는데 성공으로 판단 → 잘못된 결과 채택

```text
Adaptive Decomposition
       depends on
Reliable Failure Detection
```

이는 Part 02 Robust Verification과 연결되는 **시스템 수준의 해석**이다.

### 더 중요한 질문 — 실패하면 무조건 분해해야 하나?

**아니다.**

```text
Can't Solve ≠ Too Complex
```

실패 원인이 무엇인지에 따라 대응이 달라질 수 있다.

```text
FAIL
 │
 ├─ Too Complex       → Decompose
 ├─ Missing Evidence  → Retrieve
 ├─ Wrong Plan        → Replan
 ├─ Invalid Output    → Repair
 └─ Authority Issue   → Stop / Escalate
```

따라서 일반적인 agent architecture에서는 다음이 더 안전한 추상화 후보가 된다.

```text
Execution
    ↓
Verification
    ↓
Failure
    ↓
Cause Classification
    ↓
Appropriate Repair Strategy
```

**중요:** 이 failure taxonomy와 repair router는 ADaPT 논문의 직접 제안이 아니라, 학습 중 Part 02와 DHAA의 authority boundary를 연결해 도출한 engineering interpretation이다. ADaPT 자체의 as-needed decomposition과 구분해야 한다.

### Human-First Playbook 연결

요구사항 자체가 불명확할 때는 subtask를 더 작게 쪼개도 누락된 사실이 생겨나지 않는다. 이때는 evidence 확인이나 인간의 결정이 필요할 수 있다.

> **Research flag 🚩:** Failure → Cause Classification → Appropriate Repair. Part 04 종료 후 Gist impact를 검토하되, 지금은 Playbook에 반영하지 않는다.

---

## 7. Takeaways

1. ADaPT의 핵심은 **Execute First → Fail → Decompose**.
2. Decomposition depth는 task complexity와 executor capability에 적응한다.
3. 실패 판정이 부정확하면 불필요한 분해 또는 실패의 조기 승인으로 이어질 수 있다.
4. **Can't Solve ≠ Too Complex**. 실패 원인 분류와 적절한 복구 전략은 ADaPT를 실제 agent architecture에 적용할 때 고려할 확장 관점이다.

## Glossary

| Term | Meaning |
|---|---|
| **ADaPT** | **As-Needed Decomposition and Planning with Language Models.** 필요한 경우에만 작업을 분해하는 접근 |
| **LM / LLM** | Language Model / Large Language Model. 언어 모델 / 대규모 언어 모델 |
| **Executor** | 현재 task를 직접 수행하려는 구성요소 |
| **Decomposition** | task를 작은 subtask들로 나누는 것 |
| **Recursive Decomposition** | 실패한 subtask에 동일한 분해 절차를 반복 적용 |
| **Atomic Task** | 현재 executor가 추가 분해 없이 수행할 수 있는 수준의 작업 |
| **Verifier** | 결과가 성공 조건을 충족했는지 평가하는 요소 |
| **False Positive / False Negative** | 실제 상태와 verifier의 분류가 어긋나는 오류. Positive의 정의를 먼저 확인해야 함 |
| **Adaptive Compute Allocation** | 문제나 상황에 따라 계산 비용을 선택적으로 배분하는 전략 |

## Research → System Mapping (Deferred)

- **KEEP:** executor capability에 따라 decomposition granularity를 조정하는 발상
- **KEEP:** failure detection reliability와 decomposition decision을 분리
- **POST-READING CANDIDATE:** failure classification → repair strategy routing
- **NO DHAA WRITE:** 전체 논문 리딩 종료 전에는 DHAA 코드를 변경하지 않는다.
- **GIST FLAG ONLY:** Part 04 종합 시 승격 여부 판정

## References

- [ADaPT — As-Needed Decomposition and Planning with Language Models](https://arxiv.org/abs/2311.05772)
- [Stanford CS329A — Self-Improving AI Agents](https://cs329a.stanford.edu/)
