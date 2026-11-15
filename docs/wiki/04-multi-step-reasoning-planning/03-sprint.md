# Chapter 03 — SPRINT: Dependency-Aware Parallel Reasoning

> **Part:** 04 — Multi-Step Reasoning & Planning  
> **Status:** Study checkpoint — 1교시 및 Q&A 기록, 논문 전체 학습은 진행 중

## 1. Research Question

LATS는 **어느 trajectory를 탐색할까?**, ADaPT는 **언제 task를 더 분해할까?**를 묻는다.

SPRINT에서는 **추론 단계가 순차적으로 표현되었다는 이유만으로 반드시 순차적으로 실행해야 하는가?**를 묻는다.

독립적인 단계들을 찾아 병렬 실행하되, 실제 의존성을 보존해야 한다.

> **용어 캡션 — CoT (Chain-of-Thought):** 답에 도달하는 과정에서 생성되는 중간 추론 단계. 관측 가능한 CoT가 실제 내부 계산 전체를 그대로 드러낸다고 가정하면 안 된다.

## 2. Sequential Order vs Dependency Order

```text
Sequential:
A → B → C → D

Dependencies:
     A
    / \
   B   C
    \ /
     D
```

B와 C가 서로 의존하지 않는다면 A 완료 후 병렬로 실행하고, 둘 다 완료된 뒤 D를 실행할 수 있다.

> **용어 캡션 — DAG (Directed Acyclic Graph):** 간선에 방향이 있고 방향을 따라 순환하지 않는 그래프. 여기서는 작업 간 선행 의존성을 표현한다.

**예시 계산(논문 실험 수치가 아님):**

- A=2초, B=5초, C=4초, D=3초
- 순차: 2+5+4+3 = **14초**
- 병렬: 2+max(5,4)+3 = **10초**
- 이상적인 시간 단축 약 **28.6%**. 실제로는 동기화·스케줄링·통신 오버헤드가 발생한다.

## 3. Interleaved Planning and Parallelized Execution

```text
Planner
  ↓
Independent subtasks identified
  ↓
Parallel executors
  ↓
Join / synchronize results
  ↓
Planner (next round)
```

> **용어 캡션 — Interleaved:** 계획과 실행을 한 번씩 끝내는 것이 아니라 계획 → 실행 → 동기화 라운드를 번갈아 수행하는 방식.

SPRINT는 단순히 프롬프트에 '병렬로 해라'고 적는 방식이 아니다. 순차 추론을 의존성 기반 병렬 구조로 재구성하는 데이터·학습 접근과 병렬 실행 구조를 함께 다룬다. 구체적인 알고리즘과 실험 수치는 후속 논문 정독에서 재확인한다.

## 4. ❓ Q&A — Orchestrator–Worker와 Diamond Inheritance를 닮지 않았나?

### Orchestrator–Worker와의 관계

**유사하다.** Planner가 독립 작업을 식별하고, executors가 병렬로 수행하며, 결과를 동기화하고 다음 계획을 세우는 구조는 Orchestrator–Worker 패턴과 닮았다.

다만 **Orchestrator–Worker는 일반적인 실행 아키텍처 패턴**, **SPRINT는 추론의 병렬화를 위한 연구 프레임워크**다. 같은 것으로 동일시하면 안 된다.

```text
Dependency DAG → Ready tasks → Orchestrator → Workers
                                     ↓
                              Results / Join
                                     ↓
                              DAG state update
```

### Diamond Inheritance와의 관계

**그래프 모양은 닮았지만 간선의 의미가 다르다.**

| | Task Dependency DAG | Diamond Inheritance |
|---|---|---|
| 노드 | 추론 단계 / 작업 | 클래스 |
| 간선 | 선행 작업 의존성 | 상속 관계 |
| 문제 | 실행 순서, 병렬 가능성 | 중복 상속, 멤버 모호성 |
| 접근 | 의존성 기반 스케줄링 | 언어별 상속 규칙, 명시적 해결 |

> **Connecting the dots:** DAG는 의존성 표현, Orchestrator–Worker는 실행 아키텍처, Scheduler는 실행 가능한 작업의 선택을 담당한다.

## 5. ❓ Q&A — Frontier LM의 CoT가 블랙박스라면 실제 의존성을 어떻게 검증하나?

사용자 문제 제기: **B가 C의 결과를 암묵적으로 사용한다면 병렬화할 수 없는데, 모델의 내부 CoT가 비공개이거나 불충실할 수 있지 않은가?**

핵심 구분:

```text
Observable CoT ≠ Faithful Reasoning ≠ Verified Dependency
```

> **용어 캡션 — CoT Faithfulness:** 모델이 표현한 추론 설명이 실제 답변 생성에 사용된 내부 계산과 얼마나 충실하게 대응하는가.

- 내부 계산을 직접 관측하기 어렵다는 사실과 **외부 실행 결과를 전혀 검증할 수 없다**는 주장은 다르다.
- 공개된 CoT가 있어도 실제 내부 계산의 완전하고 충실한 기록이라고 보장할 수 없다.
- 도구 호출, 작업 입출력, 테스트, 외부 환경 피드백은 별도로 기록하고 검증할 수 있다.
- SPRINT에서 구성한 의존성 DAG를 **모델 내부 인과 구조를 완벽히 복원한 그래프**로 해석하면 안 된다.

### 누락된 dependency의 예

```text
Extracted (wrong):
A → B ─┐
       ├→ D
A → C ─┘

Actual dependency:
A → C → B → D
```

겉으로 독립적인 B와 C를 병렬 실행하면, B가 필요로 하는 C의 결과가 아직 없을 수 있다.

**두 검증 질문을 분리해야 한다.**

1. **Outcome Verification:** 병렬 실행 결과가 맞는가?
2. **Dependency Verification:** 필수 선행 조건을 누락하지 않았는가?

우연히 정답이 나왔다고 의존성 추출까지 올바른 것은 아니다. 이는 Part 02 Robust Verification 및 black-box testing의 한계와 연결된다.

> **Research question 🚩:** 추출된 DAG의 dependency가 틀렸을 때 이를 어떻게 탐지하고 복구할까? 이는 현재 학습에서 도출한 engineering question이며, SPRINT 논문의 검증된 해결책이라고 주장하지 않는다.

## 6. Glossary

| Term | Meaning |
|---|---|
| **SPRINT** | 순차 추론의 의존성을 분석해 병렬 계획·실행을 가능하게 하는 연구 프레임워크 |
| **CoT** | Chain-of-Thought, 중간 추론 단계 |
| **DAG** | Directed Acyclic Graph, 방향 비순환 그래프 |
| **Partial Order** | 모든 쌍의 순서를 강제하지 않고 필요한 선행 관계만 정하는 순서 |
| **Orchestrator–Worker** | 중앙 조정자가 작업을 나누고 worker의 실행 결과를 취합하는 패턴 |
| **Interleaved** | 계획과 실행을 번갈아 반복하는 방식 |
| **Dependency Extraction** | 단계 사이의 필수 선행 관계를 추출하는 과정 |
| **CoT Faithfulness** | 표현된 추론 설명이 실제 추론에 대응하는 충실도 |
| **Outcome Verification** | 결과가 성공 조건을 만족하는지 검사 |
| **Dependency Verification** | 실행 순서에서 필수 의존성을 보존했는지 검사 |
| **Critical Path** | 의존성 그래프에서 전체 완료 시간을 제약하는 최장 경로 |

## 7. Checkpoint / Deferred Decisions

- **Learned:** sequential presentation does not imply sequential execution.
- **Learned:** dependency graph, execution architecture, scheduling policy는 서로 다른 계층이다.
- **Open:** 자연어 추론 단계에서 dependency를 어떻게 추출·검증할 것인가?
- **Open:** 잘못 추출한 dependency를 어떻게 탐지·복구할 것인가?
- **NO DHAA WRITE:** 전체 리딩 로드맵 완료 전 DHAA 변경 금지.
- **GIST FLAG ONLY:** Part 04 종합에서 Playbook 영향 평가.
- **Public-repo boundary:** 회사 프로젝트의 비공개 구현·데이터·요구사항은 기록하지 않는다.

## References

- [Stanford CS329A — Self-Improving AI Agents](https://cs329a.stanford.edu/)

---

## 8. Stage Packing 복습 Q&A

### ❓ Step은 Stage에 포함되는 관계가 맞는가?

**맞다.** Step은 개별 추론 단위이고, Stage는 Step을 배치하는 실행 라운드다. 하나의 Stage에 하나 이상의 Step이 속한다.

- **Step:** 하나의 계획 또는 실행 작업
- **Stage:** 의존성에 따라 실행 시점을 공유할 수 있도록 묶은 Step들의 집합
- **Plan-only Step:** 실행 결과를 기다릴 필요가 없어, 후속 Step과 같은 Stage에 들어갈 수 있다.
- **Execution Step:** 후속 Step이 그 실행 결과를 필요로 한다면 Stage 경계를 넘어야 한다.

```text
Reasoning trajectory
├─ Stage 1
│  ├─ S1 (Plan-only)
│  ├─ S2 (Plan-only)
│  └─ S3 (Execution)
└─ Stage 2
   └─ S4 (Execution; depends on S3 result)
```

같은 Stage에 배치했다고 모든 Step이 *동시에* 실행되는 것은 아니다. DAG 간선을 지우는 대신 **의존성을 유지한 채 스케줄을 압축**한다.

> **용어 캡션 — Packing:** 의존관계를 보존하면서 복수 Step을 실행 Stage에 배치하는 과정.

## 9. SPRINT 결과 해석과 한계

핵심적으로 구분해야 할 세 지표:

1. **Sequential Tokens / Critical Path:** 병렬화 이후 순차적으로 기다려야 하는 생성 경로의 양
2. **Total Tokens / Total Compute:** 모든 병렬 실행 작업이 생성·처리한 총량
3. **Wall-clock Latency:** 실제로 최종 결과가 나올 때까지 경과한 시간

**Less Sequential Compute ≠ Less Total Compute ≠ Guaranteed Lower Latency.**

독립 작업을 병렬 실행하면 순차 경로가 짧아질 수 있으나 총 토큰 수는 더 늘어날 수도 있다. GPU 자원이 충분하지 않거나 계획·분배·통신·동기화 오버헤드가 크면 **실제 지연시간 단축은 보장되지 않는다.**

### ❓ “순차 연산을 덜 하면 왜 지연시간이 줄 수 있고, 왜 보장되지 않나?”

설명용 가상 예시(논문 실험값 아님):

- A, B, C가 각각 5초 걸릴 때 순차 실행은 **15초**.
- 세 작업이 독립적이고 자원이 충분하면 병렬 실행은 **5초**. 전체 작업량은 사라진 것이 아니라 동시에 처리된 것이다.
- 병렬 실행에 계획·동기화 오버헤드 **11초**가 추가되면 전체 **16초**, 오히려 순차 실행보다 느리다.

> **사용자의 복습 결론:** “동기화 및 추가 GPU 사용량 오버헤드에 따라 반드시 병렬 실행이 전체 작업 시간을 줄인다고 보장할 수 없다.”

정확하다. 더 일반적으로는 작업 분배, 자원 경쟁, 통신 및 Planner의 순차 처리도 오버헤드다.

> **용어 캡션 — Critical Path:** 의존성 DAG에서 전체 완료 시간의 하한에 영향을 주는 최장 선행 경로. 충분한 병렬 자원과 오버헤드가 없다는 이상적 조건을 구분할 것.

> **용어 캡션 — Wall-clock Latency:** 작업 시작부터 종료까지 실제 시계로 측정한 경과 시간.

### Amdahl's Law와 Connecting the dots

병렬화 가능한 비율이 `p`, 병렬 자원이 `N`개라면 이상화된 상한은:

```math
Speedup(N) = 1 / ((1-p) + p/N)
```

따라서 병렬화할 수 없는 Planner·의존 경로와 동기화 비용이 많을수록 자원 증가의 효용이 줄어든다. 이 식은 **이상화한 성능 모델**로, 실제 SPRINT 실험값을 산출하는 공식은 아니다.

### 결과 해석에서 지킬 것

SPRINT의 병렬화 장점을 **'총 토큰 비용 절약'** 혹은 **'실제 GPU 환경에서 확정적인 응답시간 단축'**으로 바꿔 말하지 않는다. 실측 wall-clock과 논문의 이론·추정 latency는 구분한다.

## 10. Chapter 03 종합 Checkpoint

```text
Sequential CoT
   ↓
Step Extraction (Plan / Execution)
   ↓
Dependency Extraction
   ↓
DAG Construction
   ↓
Stage Packing (Plan-only 압축)
   ↓
Parallel Executors + Synchronization
   ↓
Next Stage / Final Answer
```

**핵심:** 논리적 의존성은 유지하되 불필요한 순차 실행을 제거한다. 실제 시스템 성능은 dependency 추출의 신뢰도, 병렬 자원, 동기화 비용에 좌우된다.

**다음 연구:** Chapter 04 — AB-MCTS (width vs depth compute allocation). 이 장을 확인하는 동안 DHAA와 Human-First Playbook은 수정하지 않는다.

## Glossary 추가

| Term | Meaning |
|---|---|
| **Step** | 하나의 추론/계획/실행 단위 |
| **Stage** | 하나 이상의 Step을 묶은 실행 라운드 |
| **Plan-only Step** | 별도 실행 결과가 없는 계획 전용 Step |
| **Packing** | DAG 의존성을 보존하며 Step을 Stage에 배치 |
| **Sequential Tokens** | 병렬 경로를 고려했을 때 순차적으로 생성되어야 하는 토큰 수 |
| **Total Tokens** | 여러 실행 경로에서 생성되는 모든 토큰 수의 합 |
| **Wall-clock Latency** | 실제 시작~종료까지 걸린 경과 시간 |
| **Barrier Synchronization** | 여러 worker의 작업 완료를 기다리는 동기화 지점 |
| **Amdahl's Law** | 순차적으로 남는 작업 비율이 병렬 속도 향상의 상한을 제한한다는 법칙 |
