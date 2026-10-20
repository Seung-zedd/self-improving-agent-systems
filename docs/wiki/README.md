# Wiki

이 디렉터리는 **Self-Improving Agent Systems**의 paper-first 학습 노트를 모은다.

각 챕터는 논문 요약 자체보다 inference-time scaling, verification, evaluation, orchestration, authority boundary가 실제 agent system 설계에 어떻게 연결되는지를 중심으로 정리한다.

## Chapters

| # | Topic | Status |
|---:|---|---|
| 01 | [Test-Time Compute Scaling](./01-test-time-compute-scaling.md) | Complete |
| 02 | [Robust Verification](./02-robust-verification.md) | Complete |
| 03 | [Agentic Evaluations & Long-Horizon Tasks](./03-agentic-evaluations-long-horizon-tasks.md) | Complete |
| 04 | Multi-Step Reasoning & Planning | Planned |
| 05 | Learning from Feedback | Planned |
| 06 | Open-Ended Self-Improvement | Planned |
| 07 | Search & Deep Research Agents | Planned |
| 08 | Agentic Software Engineering | Planned |
| 09 | Memory & Context | Planned |
| 10 | Train-Time Scaling & Reinforcement Learning | Planned |

## Chapter Template

각 챕터는 대체로 다음 구조를 따른다.

1. Problem
2. Prerequisites
3. Paper Map
4. Core Idea
5. Algorithm / Architecture
6. Paper Comparison
7. Freshness Check
8. System Mapping
9. Apply or Archive
10. Takeaways
11. Inline ❓ Q&A — 관련 개념 바로 아래에 학습 중 질문 배치

## Conventions

- **Paper First, Video Optional**
- 논문 전체 재현보다 오래 남는 abstraction을 우선한다.
- negative result를 first-class evidence로 취급한다.
- capability와 authority를 분리해서 본다.
- 구현 아이디어는 **APPLY NOW / KEEP IN MIND / SKIP**으로 판정한다.
- 최신 연구가 해석을 실질적으로 바꾸면 기존 노트를 갱신한다.
- 학습 도중 실제로 나온 중요한 질문은 **관련 개념 바로 아래 `❓ Q&A` 형식으로 inline 배치**한다.
- 단순 확인 질문은 생략하고, 개념 이해를 바꾸거나 수학·CS 기초·기존 프로젝트와 연결되는 질문만 보존한다.
- 수학, CS 기초, 기존 프로젝트와의 연결은 **Connecting the dots** 관점에서 명시적으로 기록한다.

## Current Focus

다음 챕터는 **Multi-Step Reasoning & Planning**이다.
