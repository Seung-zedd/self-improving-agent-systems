# 03. Agentic Evaluations & Long-Horizon Tasks

> **Status:** Complete  
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

## 5. GDPval — Real-World Economically Valuable Tasks

앞의 *Measuring AI Ability to Complete Long Software Tasks*가 **task horizon**을 평가축으로 가져왔다면, GDPval은 평가 대상을 **실제 경제적으로 가치 있는 전문 업무**로 확장한다.

기존 benchmark는 대체로 잘 정제된 문제와 명확한 채점 기준을 가진다.

```text
Input
 ↓
Well-defined task
 ↓
Model
 ↓
Answer
 ↓
Score
```

하지만 실제 professional task는 불완전한 요구사항, 여러 자료, domain knowledge, 긴 작업 과정과 최종 deliverable의 품질이 함께 얽힌다.

```text
Incomplete requirements
+ Multiple sources
+ Domain knowledge
+ Work process
+ Deliverable quality
        ↓
Real-world professional task
```

따라서 평가 단위도 단순한 **answer correctness**에서 **work product quality**로 확장된다.

```text
Question → Answer
```

보다:

```text
Professional task
      ↓
Work process
      ↓
Deliverable
```

에 가까워진다.

이런 업무에서는 하나의 정답 문자열보다 결과의 정확성, 요구사항 충족도, 근거의 적절성, 전문적 품질, 실제 사용 가능한 deliverable인지 같은 여러 평가축이 중요해진다.

이 지점에서 Part 02의 **Robust Verification** 문제가 다시 등장한다.

```text
Agent generates work
        ↓
Who verifies it?
        ↓
What counts as ground truth?
        ↓
How reliable is that verifier?
```

### Task Horizon과 Economic Value는 다른 축이다

앞 논문의 time horizon과 GDPval의 real-world value를 함께 보면 agent evaluation은 적어도 두 개의 서로 다른 질문을 갖는다.

```text
             Real-world / Economic Value
                        ↑
                        │
                        │
                        └──────────────→ Task Horizon
```

> **얼마나 긴 작업을 완주할 수 있는가?**

와

> **그 작업의 결과가 실제로 가치 있고 usable한가?**

는 같은 질문이 아니다. 오랫동안 autonomous하게 실행되더라도 결과물이 쓸모없다면 높은 task horizon만으로 좋은 agent라고 할 수 없다.

### Connecting the dots

Part 02와 현재 Part를 연결하면 agent capability는 단일 accuracy score가 아니라 여러 독립적인 축으로 보이기 시작한다.

```text
Agent Capability

├─ Correctness
├─ Verification reliability
├─ Task horizon
├─ Real-world usefulness
├─ Robustness
└─ Authority / Safety
```

> **주의:** Authority / Safety는 GDPval이 직접 제안하는 평가축이 아니라, DHAA에서 다뤄 온 authority-aware system design을 연결해 일반화한 별도의 축이다.

핵심 구분은 다음과 같다.

> **Technical completion ≠ Real-world task success**

시스템의 각 기능이 기술적으로 동작한다는 사실과, 그 시스템이 원래 해결하려던 실제 문제를 제대로 해결한다는 사실은 서로 다른 evaluation layer에 속한다.
---

## 6. DeepScholar-Bench — Generative Research Synthesis

DeepScholar-Bench는 단순 QA가 아니라 **Generative Research Synthesis**를 평가한다.

```text
Research Question
       ↓
Search / Retrieval
       ↓
Relevant Sources
       ↓
Read & Understand
       ↓
Cross-source Synthesis
       ↓
Claims + Citations
       ↓
Research Report
```

최종 보고서만 채점하면 agent가 실제로 어느 단계에서 성공하거나 실패했는지 알기 어렵다. 따라서 research agent의 능력을 여러 capability로 분해해 평가하는 관점이 중요하다.

```text
Research Agent Quality
        │
        ├── Retrieval
        │    필요한 source를 찾았는가?
        │
        ├── Synthesis
        │    여러 source의 정보를 올바르게 연결했는가?
        │
        └── Verifiability
             claim을 citation으로 검증할 수 있는가?
```

이 구조는 Part 02의 **Process Verification / failure localization**과 직접 연결된다.

```text
관련 source 자체를 못 찾음
→ Retrieval failure

source는 찾았지만 잘못 해석
→ Synthesis failure

해석은 맞지만 citation이 claim을 지지하지 않음
→ Verifiability failure
```

따라서 핵심은 다음과 같다.

> **Agent evaluation도 monolithic score 하나가 아니라 capability decomposition이 필요하다.**

### Long-Horizon Reliability와의 연결

research agent는 search, read, compare, filter, synthesize, cite, revise 같은 여러 단계를 하나의 trajectory에서 수행한다. 각 component를 개별적으로 잘 수행하는 것과 전체 trajectory를 안정적으로 완주하는 것은 다른 문제다.

```math
P(\text{success}) = P(R \cap S \cap V \cap \cdots)
```

즉 Retrieval, Synthesis, Verifiability 등의 조건을 같은 trajectory에서 동시에 만족해야 한다.

> **Component capability ≠ End-to-End agent reliability**

### ❓ Q&A — 신모델의 Coding / Math / Reasoning benchmark 점수도 결국 Input → Output 평가인가?

대체로 개별 benchmark는 주어진 문제에 대한 output을 ground truth, unit test, rubric 등으로 평가하고 그 결과를 aggregate한다. 따라서 모델 발표에서 보이는 영역별 단일 점수는 주로 **component capability를 압축한 지표**로 이해할 수 있다.

```text
Input
  ↓
Model
  ↓
Output
  ↓
Ground truth / Test / Rubric
  ↓
Score
```

다만 모든 benchmark가 동일한 방식은 아니므로 전부 단순한 문자열 정답 비교라고 일반화해서는 안 된다. coding benchmark는 test execution을 사용할 수 있고, 일부 reasoning benchmark는 과정이나 judge를 포함할 수도 있다.

중요한 구분은 다음이다.

```text
Capability benchmark
"이 문제를 풀 수 있는가?"
        ↓
Long-horizon evaluation
"얼마나 긴 task까지 완주하는가?"
        ↓
Real-world evaluation
"실제 가치 있는 일을 수행하는가?"
        ↓
Decomposed evaluation
"어디서 성공/실패하는가?"
```

따라서 높은 Math / Coding / Reasoning 점수가 곧바로 같은 수준의 autonomous-agent reliability를 의미하지 않는다.

### Connecting the dots

```text
Long Software Tasks
→ Task Horizon

GDPval
→ Real-World Task Quality

DeepScholar-Bench
→ Capability-Decomposed Evaluation
```

Part 02의 Robust Verification까지 연결하면 긴 agent trajectory의 중간 process를 검증하고, 최종 deliverable을 평가하며, 실패가 발생한 capability를 localization하는 구조로 이어진다.

---

### ❓ Q&A — 기존 모델이 y=f(x)라면, Agent는 실생활의 trajectory를 실행하므로 benchmark도 다축이어야 하는가?

좋은 추상화다. 다만 기존 모델을 문자 그대로 입력값 1개·출력값 1개라고 보기보다, 많은 benchmark가 **주어진 input에 대한 output 품질을 하나의 평가 문제로 환원하기 쉬웠다**고 표현하는 편이 정확하다.

```math
y=f(x)
```

```text
Input x
  ↓
Model f
  ↓
Output y
  ↓
Ground truth / Test / Rubric
```

반면 agent는 한 번의 mapping보다 **시간에 따라 환경과 상호작용하는 trajectory**에 가깝다.

```math
\tau=(S_0,A_0,S_1,A_1,\ldots,S_T)
```

agent의 action은 환경을 바꾸고, 바뀐 환경은 다시 다음 observation/input을 만든다.

```math
S_{t+1}=T(S_t,A_t)
```

```text
Observe
  ↓
Action
  ↓
Environment / State change
  ↓
New observation
  ↓
Next action
  ↓
...
```

따라서 최종 output 하나만으로는 전체 agent behavior를 충분히 설명하기 어렵고, capability, task horizon, real-world usefulness, 단계별 failure, end-to-end reliability 같은 여러 평가축이 자연스럽게 필요해진다.

> **모델 benchmark는 output을 평가하기 쉬웠지만, agent benchmark는 동적으로 변화하는 system trajectory를 평가해야 한다.**

이것이 Part 03의 세 논문이 각각 task horizon, real-world task quality, capability decomposition을 추가하는 이유를 하나의 관점으로 묶어 준다.

---
## 7. Current Takeaways

현재까지의 핵심은 다음과 같다.

1. Agent capability에는 **problem difficulty뿐 아니라 task horizon이라는 시간축**이 있다.
2. 긴 task에서는 state와 dependency가 늘어나며 작은 오류가 복리처럼 전파될 수 있다.
3. long-context 안에 정보가 존재하는 것과 실제로 안정적으로 활용되는 것은 다르다.
4. KV cache는 hallucination의 원인이 아니라 inference optimization이며, context-management policy와 구분해야 한다.
5. Attention과 RAG는 모두 similarity-based scoring이라는 공통 abstraction으로 연결할 수 있다.
6. 긴 agent workflow에서는 중요한 상태를 context 밖의 artifact로 materialize하는 것이 유리하다.
7. GDPval은 benchmark evaluation을 실제 경제적으로 가치 있는 professional task와 deliverable quality로 확장한다.
8. **Task horizon과 real-world usefulness는 서로 다른 평가축**이다.
9. **Technical completion과 real-world task success를 구분**해야 한다.
10. DeepScholar-Bench는 research workflow를 Retrieval / Synthesis / Verifiability 등으로 분해해 평가한다.
11. **Component capability와 end-to-end agent reliability는 동일하지 않다.**
12. 영역별 benchmark 점수는 유용하지만, 긴 autonomous trajectory의 신뢰성을 직접 보장하지는 않는다.
13. 모델의 output 평가와 달리 agent evaluation은 **state-action trajectory와 feedback loop**를 평가해야 하므로 본질적으로 다축적이다.

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
