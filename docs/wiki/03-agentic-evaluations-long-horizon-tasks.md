# 03. Agentic Evaluations & Long-Horizon Tasks

> **Status:** In Progress  
> **핵심 질문:** 에이전트가 짧은 benchmark 문제를 잘 푸는 것을 넘어서, 사람이 몇 시간·며칠씩 수행하는 긴 작업을 얼마나 신뢰성 있게 완주할 수 있는가?

## 1. Problem

기존 benchmark accuracy만으로는 agent의 실제 장기 작업 능력을 설명하기 어렵다.

예를 들어 어떤 모델이 코딩 benchmark에서 높은 점수를 받더라도 다음 질문에는 답하지 못한다.

```text
5분짜리 bug fix를 잘하는가?
30분짜리 feature 구현도 가능한가?
2시간짜리 migration은?
하루짜리 프로젝트는?
```

long-horizon task에서는 작업 길이가 늘어날수록 유지해야 할 state, 의존관계, 과거 결정, tool output이 함께 증가한다.

```text
Task length ↑
→ State to maintain ↑
→ Dependencies ↑
→ Opportunities for error ↑
→ Error propagation ↑
```

단순화해서 각 step의 성공률이 99%이고 100개 step이 모두 성공해야 한다면:

```math
P(\text{all 100 steps succeed}) = 0.99^{100} \approx 0.366
```

따라서 **짧은 문제에서 강한 모델과 긴 autonomous task를 안정적으로 완주하는 agent는 같은 능력이 아니다.**

### ❓ Q&A — 작업이 길어질수록 인간처럼 attention budget이 줄어드는 것인가?

정확히는 **attention budget 자체가 인간의 피로처럼 소진된다고 보는 것은 부정확하다.**

long-horizon failure는 다음 요인들이 겹친 결과에 가깝다.

```text
finite context
+ salience dilution
+ state drift
+ tool noise
+ planning inconsistency
+ compounding error
```

즉 핵심은 **attention decay라기보다 context degradation / state drift / error propagation**이다.

사람의 경우에는 working memory와 fatigue가 문제라면, LLM agent는 이전 결정과 tool output이 계속 context에 누적되면서 중요한 정보가 상대적으로 묻히거나 잘못된 state가 이후 step의 전제로 재사용될 수 있다.

---

## 2. Measuring AI Ability to Complete Long Software Tasks

이 논문은 agent의 능력을 단순 benchmark score 대신 **time horizon**으로 측정한다.

### 2.1 50% Task-Completion Time Horizon

핵심 질문은 다음과 같다.

> **AI가 성공 확률 50%로 수행할 수 있는 task를 숙련된 인간이 수행하면 보통 얼마나 걸리는가?**

예:

| Human expert task time | AI success rate |
|---:|---:|
| 5 min | 95% |
| 15 min | 85% |
| 30 min | 70% |
| 1 hour | 50% |
| 2 hours | 25% |
| 8 hours | 5% |

이 경우:

```math
T_{50} = 1\text{ hour}
```

이다.

중요한 점은 이 수치가 **AI가 실제로 1시간 동안 실행된다는 뜻이 아니라, 인간 기준으로 1시간 규모인 task를 AI가 50% 확률로 완료한다는 뜻**이라는 것이다.

```text
Human expert task length
        ↓
AI success probability
        ↓
50% completion point
        ↓
Capability time horizon
```

이 metric은 다음 질문을 직접적으로 만든다.

> **How long a coherent task can the agent reliably carry?**

---

## 3. Long-Horizon Failure와 Transformer Context

long-horizon agent에서는 이전 단계의 결과가 다음 단계의 input으로 계속 들어가는 경우가 많다.

```text
Plan
Tool output
Code diff
Error log
Intermediate conclusion
Evidence
    ↓
Next context
    ↓
Next action
```

잘못된 중간 가정이 들어가면 이후 context에서 사실처럼 재사용될 수 있다.

```text
Step 1: 잘못된 가정
        ↓
Step 2: 그 가정을 사실처럼 재사용
        ↓
Step 3: 구현
        ↓
Step 4: 구현 결과까지 다시 context에 포함
```

따라서 long-horizon hallucination은 단순한 1회성 생성 오류보다:

> **error propagation + self-conditioning**

으로 보는 것이 더 정확하다.

### ❓ Q&A — Transformer 기반이라 context가 누적되면서 hallucination이 커지는가? KV cache도 관련 있는가?

방향은 맞지만 역할을 구분해야 한다.

Transformer attention은 일반적으로:

```math
\mathrm{Attention}(Q,K,V)
=
\mathrm{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V
```

처럼 Query가 context의 Key들과 어떤 관련성을 갖는지 계산하고 Value를 가중합한다.

context가 길어진다고 해서 그 안의 모든 정보가 동일하게 잘 활용되는 것은 아니다.

```text
Available in context
≠
Reliably retrieved / used
```

반면 **KV cache는 hallucination의 직접 원인이 아니다.**

KV cache는 이미 처리한 token들의 Key / Value를 저장해 매 decoding step마다 다시 계산하지 않도록 하는 **inference optimization**이다.

```text
KV Cache
= reuse previously computed K/V
= speed / memory optimization

KV Cache
≠ truth / memory quality guarantee
```

다만 context가 길어지면서 KV cache 비용이 커지고, 실제 시스템이 truncation, eviction, compression 같은 context-management policy를 쓰면 중요한 state가 손실될 수 있다. 이 경우 문제의 직접 원인은 KV cache 자체가 아니라 **context/KV management policy**다.

### External State의 중요성

long-horizon agent에서는 중요한 상태를 LLM context에만 맡기지 않는 것이 중요하다.

```text
LLM context
= volatile working state

PRD / SPEC / Git / Test result
= external persistent state
```

따라서 harness가 SPEC, test evidence, git diff 같은 artifact를 외부에 materialize하는 것은 long-horizon coherence를 유지하는 데 유리하다.

---

## 4. Attention과 RAG의 구조적 유사성

Transformer attention과 RAG retrieval은 서로 다른 시스템이지만 고수준 abstraction에서 공통 구조를 가진다.

```text
Query
  ↓
Similarity scoring
  ↓
Relevant candidates 강조
  ↓
Selection / Aggregation
```

### Transformer Attention

```text
Query Q
  ↓
Q · K similarity-like scores
  ↓
softmax
  ↓
weighted V aggregation
```

### RAG

```text
Query embedding
  ↓
Document embedding similarity
  ↓
Ranking / Top-k
  ↓
Selected external context
```

### ❓ Q&A — Attention의 similarity score와 RAG의 Top-k는 비슷한가?

구조적으로는 상당히 비슷하다.

둘 다 **query와 후보 사이의 relevance / similarity를 계산하고 중요한 후보를 선택하거나 더 크게 가중**한다.

다만 차이는 있다.

```text
Attention
→ 주로 현재 context 내부의 token representation
→ soft weighting / aggregation

RAG
→ 외부 vector DB / document corpus
→ ranking + Top-k retrieval
```

따라서 고수준에서는:

> **Attention = internal relevance weighting**  
> **RAG = external similarity-based retrieval**

로 이해할 수 있다.

이 연결은 이전 Part의 Weaver와도 이어진다.

```text
Dot Product / Similarity
        ↓
Scoring
        ↓
Weighting / Ranking
        ↓
Selection / Aggregation
```

---

## 5. Current Takeaways

현재까지의 핵심은 다음과 같다.

1. Agent capability에는 **problem difficulty뿐 아니라 task horizon이라는 시간축**이 있다.
2. 긴 task에서는 state와 dependency가 늘어나며 작은 오류가 복리처럼 전파될 수 있다.
3. long-context 안에 정보가 존재하는 것과 실제로 안정적으로 활용되는 것은 다르다.
4. KV cache는 hallucination의 원인이 아니라 inference optimization이며, context-management policy와 구분해야 한다.
5. Attention과 RAG는 모두 similarity-based scoring이라는 공통 abstraction으로 연결할 수 있다.
6. 긴 agent workflow에서는 중요한 상태를 context 밖의 artifact로 materialize하는 것이 유리하다.

---

## References

Stanford CS329A Schedule:
- https://cs329a.stanford.edu/

Papers:
- Thomas Kwa et al., *Measuring AI Ability to Complete Long Software Tasks* — https://arxiv.org/abs/2503.14499
- Tejal Patwardhan et al., *GDPval: Evaluating AI Model Performance on Real-World Economically Valuable Tasks* — https://arxiv.org/abs/2510.04374
- Liana Patel et al., *DeepScholar-Bench: A Live Benchmark and Automated Evaluation for Generative Research Synthesis* — https://arxiv.org/abs/2508.20033

Related implementation:
- DHAA — https://github.com/Seung-zedd/dhaa
