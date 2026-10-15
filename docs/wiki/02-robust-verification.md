# 02. Robust Verification

> **핵심 질문:** 후보를 많이 생성할 수 있게 되었을 때, 그중 무엇이 정말 맞는 답인지 어떻게 검증할 것인가?

## 1. Problem

Test-Time Compute Scaling은 다음 구조를 만든다.

```text
Prompt
  ↓
Generate N candidates
  ↓
???
  ↓
Select best answer
```

이번 파트의 핵심은 이 `???`를 다루는 것이다.

```text
Generation
= 답을 만들어내는 능력

Verification
= 만들어진 답이 맞는지 판별하는 능력
```

지난 파트의 `Discovery ≠ Selection`을 한 단계 확장하면 다음과 같다.

> **Selection quality is bounded by verifier quality.**

후보를 많이 생성해도 verifier가 잘못된 후보를 고르면 Test-Time Compute의 이득을 회수할 수 없다.

---

## 2. Prerequisites

이 파트에서 필요한 선수 개념은 많지 않다.

- repeated sampling / Best-of-N
- candidate generation vs candidate selection
- 확률적 판단과 결정적 규칙의 차이
- confusion matrix의 기본 구조

DHAA 관점에서는 다음 구분을 계속 유지한다.

```text
Candidate space  = possible
Judgment         = probable
Authority guard  = permitted
```

---

## 3. Paper Map

Stanford CS329A Robust Verification 파트의 핵심 읽기 흐름은 다음과 같이 볼 수 있다.

| Paper | 역할 | 학습 우선순위 |
|---|---|---|
| Training Verifiers to Solve Math Word Problems | 별도 verifier를 통한 candidate ranking | 필독 개념 |
| Let's Verify Step by Step | Outcome supervision → Process supervision | 필독 |
| Math-Shepherd | 자동 process supervision | 핵심 부분 |
| Shrinking the Generation-Verification Gap with Weak Verifiers (Weaver) | 여러 weak verifier를 결합해 stronger verification 구성 | 필독 |

연구 흐름을 압축하면:

```text
Final-answer verification
        ↓
Outcome Reward Model
        ↓
Process Reward Model
        ↓
Automatic Process Supervision
        ↓
Weak Verifier Aggregation
        ↓
Generation–Verification Gap 축소
```

---

## 4. Core Idea

### 4.1 Generator와 Verifier는 다른 역할이다

```text
Generator
→ 후보를 만든다.

Verifier
→ 후보가 맞는지 판단한다.
```

같은 모델을 사용하더라도 역할은 분리해서 봐야 한다.

문제를 직접 푸는 것과 이미 나온 풀이를 검사하는 것은 동일한 문제가 아니다.

---

### 4.2 Outcome Verification

Outcome verifier는 주로 최종 결과만 본다.

```text
Reasoning trajectory
        ↓
Final answer
        ↓
Correct / Incorrect
```

문제는 중간 reasoning이 틀렸는데 우연히 최종 답이 맞을 수도 있고, 거의 모든 과정이 맞았는데 마지막 계산 실수로 최종 답만 틀릴 수도 있다는 점이다.

즉 outcome supervision만으로는 다음 질문에 답하기 어렵다.

> **어디서 틀렸는가?**

---

### 4.3 Process Verification

Process supervision은 reasoning trajectory의 중간 단계를 평가한다.

```text
Step 1 → PASS
Step 2 → PASS
Step 3 → FAIL
Step 4 → ...
```

이 구조는 long-horizon agent에서도 동일하게 확장할 수 있다.

```text
Plan             → PASS
Tool selection   → PASS
Search           → PASS
Edit             → FAIL
Test             → FAIL
```

따라서 Process Verification의 실무적 가치는 단순 성공/실패 판정이 아니라 **failure localization**에 있다.

---

## 5. Process Verification과 Agent Workflow

DHAA의 Macro-workflow는 Process Verification과 구조적으로 닮아 있다.

```text
PRD
 ↓
SPEC
 ↓
Implementation
 ↓
Lint / Test / Build
 ↓
Docs Sync
 ↓
Cleanup
```

정확히는 Macro-workflow 자체가 Process Verification인 것이 아니라, **각 phase 사이에 verification gate가 배치된 구조**라고 보는 편이 더 정확하다.

```text
Development phase
       ↓
Phase-local verifier
       ↓
Pass?
 ├─ Yes → next phase
 └─ No  → repair / retry
```

따라서 DHAA Macro-workflow는 단순 Process Verification보다 다음 조합에 가깝다.

> **Process Verification + Stage Gates + Repair Loop**

그리고 이 구조의 핵심 효과는 “품질을 직접 계속 올린다”기보다 **오류가 다음 단계로 전파되는 것을 조기에 차단한다**는 데 있다.

---

## 6. Weak Verifier와 Hard Verifier

### 6.1 Weak verifier

Weak verifier는 **불완전한 verifier**다.

예:

- LLM-as-a-Judge
- reward model
- small classifier
- heuristic checker
- Jev와 같은 probabilistic judgment provider

핵심은 “확률적이냐” 자체가 아니라 **oracle처럼 항상 정확하지 않다**는 점이다.

따라서:

```text
Weak verifier
≠ probabilistic verifier 그 자체
```

확률적 verifier는 weak verifier가 될 가능성이 높지만, deterministic rule도 나쁜 규칙이면 weak할 수 있다.

---

### 6.2 Hard verifier

Hard verifier는 검증할 명제가 명확하고, 좁은 범위에서 기계적으로 판정 가능한 verifier에 가깝다.

예:

- compiler
- type checker
- JSON Schema validator
- unit test
- formal proof checker
- deterministic policy rule

하지만 hard verifier도 oracle은 아니다.

```text
Type checker
→ 타입 오류에는 강함

Type checker
→ 사용자의 제품 요구사항 충족 여부는 모름
```

따라서:

> **Hard verifier도 자기 scope 밖에서는 weak하다.**

---

## 7. Verification Stack

실제 agent system에서는 하나의 verifier보다 서로 다른 실패 모드를 가진 verifier들을 겹쳐 사용하는 편이 자연스럽다.

```text
LLM reviewer
      ↓
Type checker
      ↓
Compiler
      ↓
Unit test
      ↓
Integration test
      ↓
Security scan
      ↓
Authority gate
```

각 verifier가 묻는 질문은 다르다.

```text
LLM reviewer
→ 논리적으로 그럴듯한가?

Compiler
→ 실행 가능한가?

Unit test
→ 기대 behavior를 만족하는가?

Security scanner
→ 위험한 패턴이 있는가?

Authority guard
→ 이 변경을 자동 실행해도 되는가?
```

즉:

> **Verification은 단일 기능이 아니라 verification stack이다.**

---

## 8. Weaver — Weak Verifier Aggregation

Weaver는 verifier의 종류가 아니라, **여러 weak verifier를 결합해 더 강한 unified verifier를 만드는 framework**다.

```text
Weak verifier 1 ─┐
Weak verifier 2 ─┼─→ Weaver ─→ Stronger unified verifier
Weak verifier 3 ─┘
```

단순 majority vote는 verifier마다 reliability가 다를 때 비효율적일 수 있다.

개념적으로 candidate (x)의 score를 다음과 같이 생각할 수 있다.

```math
Score(x)=w_1V_1(x)+w_2V_2(x)+\cdots+w_nV_n(x)
```

- $V_i(x)$: verifier $i$가 candidate $x$에 준 score
- $w_i$: verifier $i$의 reliability를 반영하는 scalar weight

즉 핵심은 verifier들을 동일하게 취급하지 않고 **상대적인 reliability를 추정해서 결합**하는 것이다.

### Weaver와 DHAA의 차이

```text
Weaver
= ensemble verification
= 여러 verifier가 같은 candidate를 병렬 평가하고 합침

DHAA Macro-workflow
= layered verification
= 서로 다른 phase에 verifier를 직렬 배치
```

---

> ⚠️ **주의: MML/선형대수와 연관된 내용**
>
> weighted verifier aggregation은 MML에서 다루는 **linear combination / scalar coefficient / dot product**와 직접 연결된다.
> 수학적 연결을 깊게 이해하려면 관련 선형대수 선수지식이 도움이 된다.

벡터로 쓰면:

```math
\mathbf{w}=
\begin{bmatrix}
w_1\\
w_2\\
\vdots\\
w_n
\end{bmatrix},
\qquad
\mathbf{v}(x)=
\begin{bmatrix}
V_1(x)\\
V_2(x)\\
\vdots\\
V_n(x)
\end{bmatrix}
```

따라서 weighted aggregation은:

```math
Score(x)=\mathbf{w}^{T}\mathbf{v}(x)
```

로 쓸 수 있다.

```text
Linear Algebra
      ↓
Linear Combination
      ↓
Weighted Ensemble
      ↓
Weak Verifier Aggregation
      ↓
Robust Verification
```

이 연결은 이 저장소가 지향하는 **Connecting the dots**의 대표적인 예다.

---

## 9. Verification과 Authority는 다르다

Robust Verification에서 가장 중요한 구분 중 하나다.

```text
Generation asks:
"What can I produce?"

Verification asks:
"What should I believe?"

Authority asks:
"What am I allowed to decide?"
```

DHAA에 대응하면:

```text
Candidate FSM
= possible

Jev / DecisionProvider
= probable

Decision-Authority-Guard
= permitted
```

Jev가 어떤 후보를 높은 confidence로 선택해도 해당 결정이 founder authority를 요구한다면 최종 route는 FOUNDER여야 한다.

즉:

```math
P(\text{correct}) \neq \text{authority}
```

그래서 confidence는 permission input이 아니라 **observability / calibration signal**로 취급하는 것이 더 안전하다.

---

## 10. DHAA Negative Experiment와 Process Verification

DHAA의 Jev negative experiment를 outcome만 보면:

```text
Founder interruption reduction
= failed
```

이다.

하지만 process-level로 분해하면:

```text
Evidence available?
        ↓
Authority classification?
        ↓
Candidate reachable?
        ↓
Provider offered which classes?
        ↓
Provider selected what?
        ↓
Guard routed where?
```

가 된다.

실제 핵심은:

> **Jev가 AUTO를 선택하지 못한 것이 아니라, provider에게 AUTO-capable class 자체가 제공되지 않았다.**

따라서:

```text
Evidence missing at replay time
        ↓
Candidate FSM closes search space
        ↓
AUTO-capable class unreachable
        ↓
Provider has nothing useful to select
        ↓
No AUTO route
```

이 예시는 다음 원칙을 잘 보여준다.

> **System outcome failure does not imply component failure.**

Process Verification은 최종 FAIL보다 **어디서 FAIL이 발생했는지**를 찾는 데 강하다.

---

## 11. Confusion Matrix와 Risk

Verifier 오류는 대표적으로 다음 네 가지로 볼 수 있다.

이산수학 관점에서는 먼저 실제 상태와 예측 상태를 비교하면 쉽다.

```math
\mathrm{True} \iff \mathrm{Actual} = \mathrm{Prediction}
```

```math
\mathrm{False} \iff \mathrm{Actual} \neq \mathrm{Prediction}
```

그 다음 Positive / Negative는 **시스템 예측값**을 따른다.

| Actual | Prediction | Same? | Result |
|---|---|---:|---|
| P | P | True | TP |
| N | N | True | TN |
| N | P | False | FP |
| P | N | False | FN |

비트로 쓰면:

```text
Actual  Prediction

1       1   → TP
0       0   → TN
0       1   → FP
1       0   → FN
```

True / False 판정은 사실상 equality 또는 XNOR 관점으로 볼 수 있다.

```math
\mathrm{True} = \neg(A \oplus P)
```

DHAA에서 Positive를 `AUTO 승인`으로 정의하면:

```text
실제로 AUTO 가능 + AUTO 승인
= TP

실제로 Founder 필요 + Founder/Verify
= TN

실제로 Founder 필요 + AUTO 승인
= FP  ← 위험

실제로 AUTO 가능 + Founder/Verify
= FN  ← 자동화율 손실
```

따라서 authority-sensitive system에서는 보통:

```math
\mathrm{Cost}(FP) \gg \mathrm{Cost}(FN)
```

으로 보는 것이 자연스럽다.

즉 단순 accuracy보다 **어떤 종류의 오류가 더 위험한가**를 봐야 한다.

---

## 12. Calibration

Verifier confidence가 0.95라면 동일한 조건의 예측들 중 실제 correctness도 약 95% 수준이어야 calibrated되어 있다고 볼 수 있다.

하지만:

```text
confidence = 0.95
actual accuracy = 0.72
```

라면 overconfidence가 존재한다.

따라서:

```text
Confidence
≠ Correctness

Well-calibrated confidence
≈ empirical reliability estimate
```

DHAA에서는 confidence를 authority input으로 사용하는 대신 replay eval과 logging에 활용하는 편이 더 자연스럽다.

---

## 13. Freshness Check — 2026

2026 시점의 흐름은 verifier도 더 많은 test-time compute를 사용한다는 방향으로 확장되고 있다.

```text
Generator thinks more
        +
Verifier thinks more
```

그러나 Process Verification과 LLM-as-a-Judge가 해결된 문제는 아니다.

특히 long-horizon trajectory에서는:

```text
local correctness
≠ global correctness
```

가 될 수 있다.

중간 step이 겉으로 이상해 보여도 이후 단계에서 정당화될 수 있고, 각 step이 그럴듯해도 trajectory 전체 방향은 틀릴 수 있다.

따라서 verifier 역시 별도의 evaluation 대상이어야 한다.

---

## 14. System Mapping — DHAA

현재 DHAA 기준으로 다음처럼 정리할 수 있다.

```text
Jev
= probabilistic judgment provider
= weak-verifier-like role

Compiler / Tests
= deterministic hard verification of correctness

Decision-Authority-Guard
= deterministic hard verification of permission
```

다만 이들은 동일한 종류의 verifier가 아니다.

특히 Authority Guard는:

```text
"이 답이 사실인가?"
```

를 검증하는 것이 아니라:

```text
"이 route가 policy상 허용되는가?"
```

를 검증한다.

따라서 correctness와 permission을 반드시 분리해서 본다.

---

## 15. Apply or Archive

### APPLY NOW

현재 추가 구현보다 **기존 DHAA 설계 원칙 유지**가 중요하다.

- confidence ≠ authority
- probability ≠ permission
- deterministic guardrail 유지
- phase별 verification gate 유지
- process-level trace를 failure localization에 활용

### KEEP IN MIND

- multiple weak verifier aggregation
- verifier calibration
- process-level trace 강화
- verifier ensemble
- verifier-side test-time compute

### SKIP FOR NOW

- Jev 반복 sampling만으로 문제를 해결하려는 접근
- verifier score를 authority permission으로 직접 연결하는 설계

현재 DHAA negative experiment의 주요 병목은 verifier strength보다:

```text
evidence availability
candidate reachability
replay timing
evaluation design
```

에 더 가까웠다.

---

## 16. Takeaways

1. **Generation과 Verification은 다르다.**

   ```text
   Can I generate it?
   ≠
   Can I verify it?
   ```

2. **Outcome Verification과 Process Verification은 다르다.**

   ```text
   Did it succeed?
   vs
   Where did it fail?
   ```

3. **Verifier 하나를 신뢰하기보다 verification stack을 설계한다.**

4. **Weak verifier aggregation은 weighted ensemble 문제로 볼 수 있다.**

5. **Correctness와 Authority는 별개의 축이다.**

   ```text
   Likely correct
   ≠
   Allowed to act
   ```

6. **DHAA의 핵심 abstraction은 계속 유지된다.**

   ```text
   possible
      ↓
   probable
      ↓
   permitted
   ```

---

## 17. Q&A — 학습 중 개인 질문

이 섹션은 학습 당시 실제로 막혔던 지점과 연결점을 보존한다. 나중에 다시 읽었을 때 당시의 사고 과정을 회상하기 위한 기록이다.

### Q1. Process Verification은 DHAA의 Macro-workflow와 비슷한가?

**질문 맥락**

DHAA는 대략 다음 Macro-workflow를 가진다.

```text
PRD
→ SPEC
→ 구현
→ 테스트
→ 문서 sync
→ dead-code cleanup
```

각 단계에서 lint나 정합성 검사를 수행하기 때문에 Process Verification과 닮아 보였다.

**답**

맞다. 다만 Macro-workflow 자체가 Process Verification이라기보다 **각 phase 사이에 verification gate가 배치된 layered workflow**라고 보는 것이 더 정확하다.

그리고 실패 시 `moai loop / fix`처럼 repair가 수행되기 때문에:

> **Process Verification + Stage Gates + Repair Loop**

에 더 가깝다.

---

### Q2. Macro-workflow의 verifier들은 전부 weak verifier인가?

전부 weak라고 하면 너무 거칠다.

- 문서 정합성 판단, LLM review → weak / soft 성격
- compiler, type checker, schema validator, unit test → hard / deterministic 성격

다만 hard verifier도 scope 밖에서는 강하지 않다.

따라서 DHAA는 하나의 verifier보다 **서로 다른 verification scope를 가진 verifier들을 계층적으로 배치한 구조**로 보는 편이 좋다.

---

### Q3. Weaver와 Weak verifier는 정확히 무슨 관계인가?

`Weak verifier`는 **불완전한 개별 verifier**다.

`Weaver`는 여러 weak verifier를 결합해서 stronger unified verifier를 만드는 **framework**다.

```text
Weak verifier 1 ─┐
Weak verifier 2 ─┼─→ Weaver ─→ Stronger verifier
Weak verifier 3 ─┘
```

즉 Weaver 자체가 verifier 종류인 것은 아니다.

---

### Q4. Weaver의 weighted aggregation은 MML의 linear combination과 같은 구조인가?

그렇다.

```math
Score(x)=w_1V_1(x)+w_2V_2(x)+w_3V_3(x)
```

에서 $w_i$는 scalar coefficient에 해당한다.

벡터 표현으로 바꾸면:

```math
Score(x)=\mathbf{w}^{T}\mathbf{v}(x)
```

가 되어 dot product로도 연결된다.

따라서:

```text
MML / Linear Algebra
→ Linear Combination
→ Scalar Coefficient
→ Dot Product
→ Weighted Ensemble
→ Weak Verifier Aggregation
```

으로 연결해서 이해할 수 있다.

---

### Q5. Weak verifier = probabilistic verification, Hard verifier = deterministic verification인가?

방향은 비슷하지만 완전히 동일하지는 않다.

```text
Probabilistic
≠ automatically weak

Deterministic
≠ automatically hard
```

나쁜 deterministic heuristic도 weak verifier가 될 수 있다.

더 정확한 구분은:

- **Weak verifier:** 불완전하고 오류 가능성이 큰 verifier
- **Hard verifier:** 좁은 명제에 대해 명확한 규칙이나 실행 결과로 강하게 판정할 수 있는 verifier

DHAA에 대입하면:

```text
Jev
→ probabilistic judgment provider
→ weak-verifier-like

Decision-Authority-Guard
→ deterministic verification of permission
→ hard-verifier-like
```

단, 둘은 서로 다른 명제를 검증한다.

---

### Q6. TP / TN / FP / FN이 계속 헷갈린다. 이산수학처럼 보면 안 되나?

그렇게 보는 것이 오히려 더 직관적이다.

먼저:

```math
\mathrm{True} \iff \mathrm{Actual} = \mathrm{Prediction}
```

```math
\mathrm{False} \iff \mathrm{Actual} \neq \mathrm{Prediction}
```

그리고 뒤의 Positive / Negative는 **Prediction 값**이다.

따라서:

```text
Actual == Prediction ?
    Yes → T
    No  → F

Prediction == P ?
    Yes → Positive
    No  → Negative
```

결과:

```text
P P → TP
N N → TN
N P → FP
P N → FN
```

즉 True / False는 equality, 혹은 XNOR 관점으로 이해할 수 있다.

이 방식은 자연어 암기보다 논리적으로 재구성하기 쉽다.

---

## 18. References

- Cobbe et al., *Training Verifiers to Solve Math Word Problems*
- Lightman et al., *Let's Verify Step by Step*
- Wang et al., *Math-Shepherd: Verify and Reinforce LLMs Step-by-step without Human Annotations*
- *Shrinking the Generation-Verification Gap with Weak Verifiers* — Weaver
- Stanford CS329A — *Self-Improving AI Agents*
