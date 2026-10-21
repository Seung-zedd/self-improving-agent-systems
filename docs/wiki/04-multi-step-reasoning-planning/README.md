# Part 04 — Multi-Step Reasoning & Planning

> **Status:** In Progress

## Common Problem

Part 03에서 agent를 하나의 정답이 아니라 상태와 행동이 이어지는 trajectory로 보았다.

```text
τ = (S0, A0, S1, A1, ..., ST)
```

Part 04의 질문은 그 다음이다.

> **복잡한 목표를 여러 단계에 걸쳐 수행할 때, 좋은 trajectory를 어떻게 계획하고 탐색하며, 실패했을 때 어떻게 수정할 것인가?**

## Research Map

| Chapter | Focus | Role |
|---:|---|---|
| 01 | LATS | trajectory search |
| 02 | ADaPT | adaptive decomposition |
| 03 | SPRINT | dependency-aware parallel execution |
| 04 | AB-MCTS | width vs depth compute allocation |
| 05 | SWiRL | learning from multi-step trajectories |

각 논문은 별도 문서에서 읽고 이해한다. 이 README는 Part 04의 **티저이자 cross-paper synthesis entry point**로 사용한다.

```text
Complex Goal
     │
     ├─ Which trajectory? ───────── LATS
     ├─ When to decompose? ──────── ADaPT
     ├─ What can run in parallel? ─ SPRINT
     ├─ Explore wider or deeper? ── AB-MCTS
     └─ How to learn trajectories?  SWiRL
```

Part 전체가 끝나면 이 문서에 cross-paper synthesis와 최종 system mapping을 추가한다.

---

## Connecting the dots — 이 학습 과정 자체가 DeepScholar-style research synthesis 아닌가?

### ❓ Q&A — 지금 우리가 하고 있는 과정이 DeepScholar-Search 과정 아님?

그렇다. Part 03에서 살펴본 DeepScholar-Bench의 **Generative Research Synthesis** 관점으로 보면, 현재의 paper-first 학습 workflow 자체가 거의 같은 구조를 가진다.

```text
Research Question
      ↓
Search / Retrieval
      ↓
Relevant Papers
      ↓
Read / Understand
      ↓
Cross-source Synthesis
      ↓
Claims + Evidence
      ↓
Research Report
```

현재 이 저장소의 workflow로 대응시키면:

```text
Part의 Common Problem 정의
      ↓
Stanford reading list / 논문 retrieval
      ↓
Paper 단위 Read / Understand
      ↓
Q&A + Connecting the dots
      ↓
Cross-Paper Synthesis
      ↓
System Mapping
      ↓
Research Notes
```

Part 04부터 논문별 문서를 분리하는 이유도 여기에 있다.

```text
04-multi-step-reasoning-planning/
├─ README.md       ← cross-source synthesis
├─ 01-lats.md      ← source-level understanding
├─ 02-adapt.md     ← source-level understanding
├─ 03-sprint.md    ← source-level understanding
├─ 04-ab-mcts.md   ← source-level understanding
└─ 05-swirl.md     ← source-level understanding
```

즉 각 paper note는 **source-level analysis artifact**, Part README는 **cross-source synthesis artifact**가 된다.

### Part 02와 Part 03까지 다시 연결하면

이 workflow는 단순 요약보다 더 세분화해서 실패를 볼 수 있다.

```text
Research result가 이상하다
      ↓
Retrieval failure?
Reading / understanding failure?
Synthesis failure?
System-mapping failure?
Practice-distillation failure?
```

이 관점은 Part 02의 process verification과 Part 03의 capability-decomposed evaluation을 학습 과정 자체에 적용한 것이다.

Part 04에서는 여기에 planning, decomposition, dependency, replanning이라는 관점을 추가로 검토한다.

> **학습 workflow 자체도 연구에서 얻은 abstraction을 이용해 점진적으로 개선할 수 있다.**

---

## Documentation Strategy

Part 01–03은 하나의 문서로 충분했지만, Part 04는 논문별 planning mechanism이 서로 독립적인 학습 단위를 이룬다.

따라서:

```text
Paper 완료
→ 해당 chapter 문서 checkpoint

Part 04 전체 완료
→ README cross-paper synthesis
→ system mapping / candidate flags 정리
→ practice impact 판정
→ Part 04 Complete
```

연구에서 발견한 아이디어를 곧바로 실제 시스템에 구현하지 않는다. 먼저 학습 노트에서 mapping과 candidate를 보존하고, 전체 연구 로드맵 이후 별도로 검토한다.

## References

- [Stanford CS329A — Self-Improving AI Agents](https://cs329a.stanford.edu/)
- [DeepScholar-Bench](https://arxiv.org/abs/2508.20033)
