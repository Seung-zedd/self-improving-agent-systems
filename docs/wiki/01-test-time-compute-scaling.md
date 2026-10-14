# 01. Test-Time Compute Scaling

> Stanford CS329A의 Test-Time Compute Scaling 파트를 바탕으로, 추론 시점의 추가 계산이 모델 성능을 어떻게 끌어올리는지와 그 한계를 정리한다.

## 1. Problem

기존의 LLM scaling은 주로 학습 시점에 더 많은 파라미터, 데이터, compute를 투입하는 방식이었다.

Test-Time Compute Scaling은 질문을 바꾼다.

> **모델을 다시 학습하지 않고, 추론 시점에 더 많은 계산을 사용하면 성능을 얼마나 끌어올릴 수 있는가?**

기본 구조는 다음과 같다.

```text
Prompt
  ↓
Generate multiple candidates
  ↓
Score / Verify / Vote / Search
  ↓
Select a better answer
```

핵심은 모델 weight를 바꾸지 않고도 inference budget을 늘려 더 나은 결과를 얻는 것이다.

---

## 2. Prerequisites

이 파트에서 필요한 선수 개념은 다음 정도다.

- Sampling
- Probability
- Best-of-N
- Ranking / Reranking
- Reward Model
- Verifier
- Search space
- Compute budget

가장 중요한 추상화는 다음이다.

```text
Candidate Generation
        ↓
Candidate Scoring
        ↓
Selection
```

이 구조는 RAG의 다음 흐름과 유사하다.

```text
Retrieve
  ↓
Rerank
  ↓
Select
```

차이는 RAG는 기존 문서를 후보로 가져오고, Test-Time Scaling은 모델이 새로운 응답 후보를 생성한다는 점이다.

---

## 3. Paper Map

Stanford CS329A Schedule의 이 파트는 다음 네 논문을 읽기 자료로 둔다.

### 3.1 Large Language Monkeys: Scaling Inference Compute with Repeated Sampling

- **판정: 필독 개념**
- repeated sampling과 inference-time scaling law의 출발점
- 여러 번 샘플링했을 때 정답 후보를 한 번이라도 생성할 확률이 어떻게 증가하는지 보여준다
- 특히 자동 검증이 가능한 coding / formal proof 영역에서 효과가 직접 성능 향상으로 이어진다

### 3.2 Archon: An Architecture Search Framework for Inference-Time Techniques

- **판정: 필독 개념**
- inference-time technique들을 하나의 system architecture로 조합한다
- repeated sampling, ranking, fusion, critique, verification 등을 search space로 취급한다
- 단일 모델의 성능보다 **inference architecture 자체를 최적화**하는 관점이 중요하다

### 3.3 Scaling LLM Test-Time Compute Optimally can be More Effective than Scaling Model Parameters

- **판정: 최우선 필독**
- 무작정 sample 수를 늘리는 대신 prompt 난이도에 따라 compute를 adaptive하게 배분해야 한다는 점을 보여준다
- compute-optimal scaling이 Best-of-N baseline보다 더 효율적일 수 있음을 보여준다

### 3.4 How Do Large Language Monkeys Get Their Power (Laws)?

- **판정: 핵심 부분만**
- 개별 문제의 성공률은 exponential하게 개선될 수 있는데, 왜 전체 benchmark에서는 power-law scaling처럼 보이는지 설명한다
- 핵심 원인은 문제별 single-attempt success probability의 heavy-tailed distribution이다

---

## 4. Core Idea

### 4.1 Repeated Sampling

한 번의 시도에서 성공 확률을 `p`라고 하자.

한 번 실패할 확률은:

```text
1 - p
```

N번 모두 실패할 확률은:

```text
(1 - p)^N
```

따라서 N번 중 최소 한 번 성공할 확률은:

```text
1 - (1 - p)^N
```

예를 들어 `p = 0.2`라면:

- 1회 시도: 20%
- 5회 안에 최소 한 번 성공: 약 67%
- 10회 안에 최소 한 번 성공: 약 89%

즉 모델 자체를 바꾸지 않아도 sampling budget을 늘리면 **정답을 발견할 확률**은 크게 올라갈 수 있다.

### 4.2 Discovery와 Selection은 다르다

하지만 정답 후보를 만들었다고 해서 시스템이 자동으로 정답을 고르는 것은 아니다.

```text
Discovery
≠
Selection
```

예를 들어:

```text
A1 ❌
A2 ❌
A3 ✅
A4 ❌
A5 ✅
```

정답 후보가 존재하더라도 verifier가 A1을 가장 높게 평가하면 최종 시스템은 실패한다.

따라서 Test-Time Compute Scaling의 병목은 최소 두 개다.

```text
Can the generator discover a correct answer?
                 +
Can the verifier select the correct answer?
```

이 구분은 다음 파트인 **Robust Verification**으로 직접 이어진다.

---

## 5. Algorithm / Architecture

### 5.1 Best-of-N

가장 단순한 구조는 다음과 같다.

```text
Problem
  ↓
Generate N candidates
  ↓
Score each candidate
  ↓
Select best candidate
```

여기서 N을 늘리면 discovery coverage는 증가할 수 있지만, verifier가 약하면 실제 최종 성능은 plateau할 수 있다.

### 5.2 Archon 관점

Archon은 문제를 더 크게 본다.

```text
Generate
  ↓
Rank
  ↓
Critique
  ↓
Regenerate
  ↓
Fuse
  ↓
Verify
```

즉 추론 시점의 여러 technique을 조합한 **inference architecture** 자체를 최적화한다.

이를 최적화 문제로 쓰면 대략 다음과 같다.

```text
maximize Quality(Architecture)

subject to

Compute(Architecture) <= Budget
```

이 관점에서 중요한 것은:

> 좋은 모델 하나를 고르는 문제에서, 좋은 inference system을 설계하는 문제로 확장된다.

---

## 6. Paper Comparison

네 논문은 하나의 연구 흐름으로 보면 자연스럽다.

```text
Repeated Sampling
      ↓
정답 발견 확률 증가
      ↓
Selection 병목 발생
      ↓
Ranking / Verification / Composition
      ↓
Inference Architecture Search
      ↓
Prompt별 Adaptive Compute Allocation
      ↓
Aggregate Scaling Law 해석
```

핵심 차이는 다음과 같다.

- **Large Language Monkeys**: sample 수를 늘리면 얼마나 coverage가 늘어나는가?
- **Archon**: 어떤 inference-time technique 조합이 좋은 system을 만드는가?
- **Compute-Optimal Scaling**: 문제 난이도에 따라 compute를 어떻게 다르게 배분할 것인가?
- **Power Laws**: 왜 aggregate benchmark에서 scaling law가 특정 형태로 나타나는가?

---

## 7. Freshness Check — 2026 관점

이 파트의 핵심 아이디어는 여전히 유효하다.

다만 2026년 관점에서 질문은 더 세밀해졌다.

과거:

```text
More inference compute?
```

현재:

```text
Which problem?
Which stage?
Which branch?
Which inference strategy?
How much compute?
```

즉 최신 관점의 핵심은 단순한 compute 증가가 아니라 **adaptive allocation**이다.

또한 reasoning을 더 길게 한다고 항상 좋아지는 것은 아니다.

- 쉬운 문제에 과한 compute를 쓰면 낭비
- 이미 정답에 도달한 뒤 더 생각하면 overthinking 가능
- 매우 어려운 문제는 추가 sampling의 ROI가 낮을 수 있음

따라서 실제 agent system에는 일종의 **compute scheduler**가 필요하다고 볼 수 있다.

---

## 8. System Mapping — DHAA

현재 DHAA의 decision layer를 단순화하면:

```text
Evidence / Context
      ↓
Authority Classification
      ↓
Candidate Space Restriction
      ↓
Judgment Provider
      ↓
Deterministic Authority Guard
      ↓
Action / Verify / Escalate
```

Test-Time Scaling과 연결할 때 가장 중요한 점은:

> **추론 compute를 늘리는 것과 authority를 확장하는 것은 서로 다른 문제다.**

예를 들어 Candidate FSM이:

```text
{FOUNDER, EVIDENCE-NEEDED}
```

만 허용한 상황에서는 Jev를 1번 호출하든 1000번 호출하든:

```text
ENGINEERING-RESOLVED
EVIDENCE-RESOLVED
```

같은 후보를 만들어낼 수 없다.

provider에게 애초에 선택지로 주어지지 않았기 때문이다.

이를 한 문장으로 요약하면:

> **Search cannot recover options outside the allowed search space.**

또는:

```text
Test-Time Scaling
≠
Constraint Bypass
```

DHAA에서는 오히려 이것이 의도된 safety property다.

### Candidate FSM을 search space로 보면

```text
Candidate FSM
= Search-space boundary

Jev
= Judgment / selection mechanism

Decision-Authority-Guard
= Permission boundary

Test-Time Scaling
= Allowed space 내부에서 inference budget을 늘리는 전략
```

따라서 test-time scaling과 authority guard는 경쟁 관계가 아니라 **orthogonal layer**다.

---

## 9. Apply or Archive

### KEEP IN MIND — Adaptive Jev Sampling

이론적으로는 다음 구조를 생각할 수 있다.

```text
Allowed candidates
      ↓
Jev judgment × N
      ↓
Vote / Rank / Aggregate
      ↓
Decision-Authority-Guard
```

하지만 모든 상황에서 N을 늘릴 필요는 없다.

예:

```text
candidate 1개
→ provider 호출 불필요

AUTO-capable 후보 없음
→ repeated sampling ROI 낮음

AUTO-capable 후보 여러 개
→ additional sampling 고려 가능
```

즉 candidate-set 상태나 uncertainty에 따라 compute budget을 다르게 주는 방식이 더 자연스럽다.

현재 판정:

> **KEEP IN MIND**

이유:

- 연구 아이디어는 DHAA에 직접 연결된다
- 하지만 현재 negative experiment의 병목은 provider sampling 부족보다 candidate reachability / evidence timing 쪽에 더 가깝다
- 따라서 Best-of-N을 지금 바로 추가할 근거는 부족하다

### APPLY NOW

현재는 없음.

이 파트의 목적은 구현 변경보다 **search / compute / authority를 서로 다른 축으로 분리해 이해하는 것**이다.

---

## 10. Takeaways

### Takeaway 1

> **Test-Time Compute Scaling은 모델을 재학습하지 않고 inference budget을 늘려 성능을 높이는 전략이다.**

### Takeaway 2

> **Discovery와 Selection은 별개의 병목이다.**

```text
Can the system generate a correct candidate?
                    ↓
Can the system identify that candidate?
```

정답을 생성할 수 있는 능력과 정답을 골라낼 수 있는 능력은 동일하지 않다.

### Takeaway 3

> **더 많은 compute는 authority를 확장하지 않는다.**

DHAA 관점에서는:

```text
Inference-Time Scaling
= search harder inside the allowed space

Authority Guard
= define what space is allowed
```

즉 search quality와 decision permission은 분리되어야 한다.

---

## 이해 확인

### Q1

Candidate FSM이 `{FOUNDER, EVIDENCE-NEEDED}`만 허용한 상태에서 Jev를 100번 sampling하면 `ENGINEERING-RESOLVED`를 찾을 수 있는가?

**A. 찾을 수 없다.**

해당 class가 provider의 choice set에 포함되어 있지 않기 때문이다.

Repeated sampling은 허용된 search space 내부의 탐색을 강화할 뿐, search-space constraint 자체를 우회하지 않는다.

### Q2

Best-of-N에서 sample 수를 계속 늘려도 최종 성능이 plateau할 수 있는 이유는?

**A. Discovery가 개선되어도 Selection이 약할 수 있기 때문이다.**

정답 후보가 candidate set 안에 들어와도 verifier / reward model / ranker가 잘못 선택하면 최종 답은 틀릴 수 있다.

---

## References

Stanford CS329A Schedule:
- https://cs329a.stanford.edu/

Papers:
- Bradley Brown et al., *Large Language Monkeys: Scaling Inference Compute with Repeated Sampling* — https://arxiv.org/abs/2407.21787
- Jon Saad-Falcon et al., *Archon: An Architecture Search Framework for Inference-Time Techniques* — https://arxiv.org/abs/2409.15254
- Charlie Snell et al., *Scaling LLM Test-Time Compute Optimally can be More Effective than Scaling Model Parameters* — https://arxiv.org/abs/2408.03314
- Rylan Schaeffer et al., *How Do Large Language Monkeys Get Their Power (Laws)?* — https://arxiv.org/abs/2502.17578

Related implementation:
- DHAA — https://github.com/Seung-zedd/dhaa
