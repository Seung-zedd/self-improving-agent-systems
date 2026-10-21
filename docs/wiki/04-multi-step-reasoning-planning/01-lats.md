# Chapter 01 — LATS

> **Part:** 04 — Multi-Step Reasoning & Planning  
> **Status:** Checkpoint Complete

## 1. Problem

ReAct 계열 agent는 reasoning과 acting을 결합하지만, 한 번 선택한 trajectory를 대체로 계속 전진한다.

```text
S0 → A0 → S1 → A1 → S2 → ...
```

초기 선택이 잘못되면 이후 reasoning도 이미 좋지 않은 영역에서 계속될 수 있다.

LATS(Language Agent Tree Search)는 질문을 바꾼다.

> **"다음 행동은 무엇인가?"가 아니라 "가능한 여러 reasoning-action trajectory 중 어디를 탐색할 것인가?"**

즉 단일 trajectory를 tree-shaped search space로 확장한다.

## 2. Linked Trajectory → Tree Search

```text
Sequential trajectory

S0 → S1 → S2 → S3
```

```text
Tree search

             S0
        ┌────┼────┐
        ▼    ▼    ▼
       S1   S2   S3
      / \        / \
     ▼   ▼      ▼   ▼
    S4   S5    S6   S7
```

개념적으로 node는 agent state, edge는 action으로 볼 수 있다. 가능한 행동과 관찰이 누적되면서 trajectory search space가 만들어진다.

branching factor를 `b`, depth를 `d`라고 하면 naïve exhaustive search는 대략 다음과 같이 폭발할 수 있다.

```math
O(b^d)
```

따라서 모든 branch를 동일하게 탐색할 수 없고, inference compute를 어디에 배분할지 결정해야 한다.

## 3. MCTS — Exploration vs Exploitation

LATS는 Monte Carlo Tree Search(MCTS)를 사용해 유망한 trajectory를 탐색한다.

핵심 loop:

```text
Selection
   ↓
Expansion
   ↓
Evaluation
   ↓
Simulation
   ↓
Backpropagation
   ↓
Reflection on failure
   ↺
```

Selection에서는 지금까지 좋은 결과를 보인 node를 더 탐색하는 **exploitation**과 아직 충분히 방문하지 않은 node를 확인하는 **exploration**을 함께 고려한다.

직관적인 UCT 형태:

```math
UCT(s)
=
V(s)
+
w\sqrt{\frac{\ln N(parent(s))}{N(s)}}
```

- `V(s)`: 지금까지 해당 state가 얼마나 유망했는가
- `N(s)`: 해당 state 방문 횟수
- `N(parent(s))`: 부모 state 방문 횟수
- `w`: exploration의 상대적 비중

따라서:

```text
UCT
= exploitation
+ exploration
```

## 4. Expansion and Evaluation

선택한 state에서 LM이 여러 action candidate를 생성한다.

```text
             State
          /    |    \
         A1   A2    A3
         ↓     ↓     ↓
        S1    S2    S3
```

Part 01의 candidate search가 Part 04에서는 trajectory search로 확장된다.

```text
Part 01
Prompt → Output Candidates

Part 04 / LATS
State → Action Candidates → Future States → Trajectories
```

LATS는 **Language Model (LM)**을 value function으로도 활용하며 environment feedback과 self-consistency signal을 이용해 state의 유망함을 평가한다.

> **용어 캡션 — LM (Language Model):** 여기서는 다음 action을 생성하는 모델이면서, 동시에 현재 state가 얼마나 유망한지를 평가하는 heuristic evaluator 역할도 한다. LM의 높은 평가는 정답이나 허가를 보장하지 않는다.

중요한 구분:

> **LM value는 search heuristic이지 permission이 아니다.**

```text
Likely / Valuable
≠
Correct
≠
Permitted
```

이는 DHAA에서 분리해 온 `probable ≠ permitted`와 같은 경계 문제를 다시 보여준다. 이 연결은 현재 연구 mapping일 뿐 DHAA 구현 변경을 의미하지 않는다.

## 5. Environment Feedback and Reflection

LATS는 LM 내부 reasoning만 사용하는 것이 아니라 environment와 상호작용하며 feedback을 search에 반영한다.

```text
Action
  ↓
Environment
  ↓
Observation / Feedback
  ↓
Evaluation
  ↓
Next Search
```

terminal trajectory가 실패하면 실패 경험을 버리지 않고 reflection을 생성한다.

```text
Failed Trajectory
      +
Final Result
      ↓
Reflection
      ↓
Memory / Context
      ↓
Subsequent Search
```

이 과정은 model weight를 업데이트하는 training과 구별해야 한다. LATS에서 reflection은 search-time experience를 다음 탐색의 context로 재사용하는 방식이다.

## 6. Backpropagation

trajectory의 결과는 해당 경로의 ancestor node들로 다시 전파되어 이후 selection에 영향을 준다.

```text
S0
└─ S1
   └─ S4
      └─ S9 → result
          ↑
        S4
        ↑
        S1
        ↑
        S0
```

즉 search가 진행될수록 이전 탐색 결과가 다음 compute allocation에 영향을 준다.

---

## Connecting the dots — MCTS와 MCMC / NUTS

### ❓ Q&A — MCTS의 Monte Carlo는 Metropolis/NUTS에서 본 Monte Carlo와 같은 것인가?

같은 알고리즘은 아니다.

Bayesian inference에서의 MCMC 계열과 planning에서의 MCTS는 목적이 다르다.

```text
MCMC
= target probability distribution에서 sample을 얻는 문제

MCTS
= decision tree에서 좋은 action / trajectory를 찾는 문제
```

Metropolis-Hastings의 random-walk proposal이 고차원 공간에서 비효율적일 수 있어 HMC가 gradient와 Hamiltonian dynamics를 이용하고, NUTS가 HMC trajectory length 선택을 자동화하는 것과 달리, MCTS는 tree statistics와 value estimate를 이용해 어느 branch에 search compute를 배분할지 결정한다.

따라서 다음 연결은 잘못됐다.

```text
MCTS
→ Metropolis random walk
→ NUTS로 대체
```

하지만 더 높은 수준의 공통 abstraction은 존재한다.

```text
Unguided stochastic exploration
          ↓
Useful structural signal
          ↓
Guided, more efficient search
```

비교하면:

```text
PyMC / NUTS
Where should sampling move?
→ posterior geometry / gradient / dynamics

LATS / MCTS
Which branch should search explore?
→ value estimate / visit count / environment feedback
```

즉 둘은 서로 다른 문제를 풀지만, **naïve한 stochastic exploration보다 문제 구조에서 얻은 signal을 사용해 탐색 효율을 높인다**는 공통된 계산적 직관을 제공한다.

## 7. Part 01–04 Connection

```text
Part 01
Inference compute를 어디에 쓸 것인가?
        ↓
Part 02
찾은 결과를 어떻게 검증할 것인가?
        ↓
Part 03
긴 trajectory의 실제 성공을 어떻게 평가할 것인가?
        ↓
Part 04 / LATS
trajectory 자체를 search space로 만들 수 있는가?
```

## Glossary

| Term | Meaning in this chapter |
|---|---|
| **LM** | **Language Model.** action candidate를 생성하거나 state의 유망함을 평가하는 언어 모델. |
| **Value Function / Value Estimate** | 현재 state가 앞으로 좋은 결과로 이어질 가능성을 search 관점에서 추정하는 값. 정답 보장이나 permission이 아니다. |
| **MCTS** | **Monte Carlo Tree Search.** tree에서 exploration과 exploitation을 조절하며 유망한 trajectory를 탐색하는 방법. |
| **UCT** | MCTS selection에서 node의 현재 가치와 아직 덜 탐색된 정도를 함께 고려하는 기준. |
| **Self-Consistency** | 여러 생성 결과 사이의 일관성/합의를 추가 signal로 활용하는 방식. |
| **Environment Feedback** | 모델 자기평가가 아니라 실행 환경에서 돌아온 observation, reward, test result 등의 외부 signal. |
| **Trajectory** | state와 action이 시간 순서로 이어지는 경로, 예: \(S_0, A_0, S_1, A_1, \ldots, S_T\). |

> 이후 chapter에서도 처음 등장하는 약어나 직관적으로 파악하기 어려운 용어는 **첫 등장 시 풀네임 + 짧은 용어 캡션**으로 설명하고, 반복해서 참조할 핵심 용어는 Glossary에 모은다.

## 8. Current Takeaway

LATS를 한 문장으로 압축하면:

> **ReAct의 단일 reasoning/acting trajectory를 tree-shaped search space로 확장하고, MCTS와 value estimation, environment feedback, reflection을 이용해 유망한 trajectory에 inference compute를 선택적으로 배분한다.**

### Research → System Flag

- **KEEP / POST-READING CANDIDATE:** trajectory search와 authority boundary를 분리해서 볼 것.
- **KEEP:** environment feedback은 LM 자기평가와 구별되는 signal이다.
- **KEEP:** failure reflection은 training 없이도 이후 search context를 개선할 수 있다.
- **DO NOT IMPLEMENT YET:** 전체 paper-reading roadmap이 끝나기 전에는 DHAA에 역반영하지 않는다.

## References

- Zhou et al., [Language Agent Tree Search Unifies Reasoning, Acting, and Planning in Language Models](https://arxiv.org/abs/2310.04406)
- [Stanford CS329A — Self-Improving AI Agents](https://cs329a.stanford.edu/)
