# Self-Improving Agent Systems

> Paper-first notes on inference-time scaling, verification, evaluation, and authority-aware AI agent systems.

This repository is a living study notebook for **self-improving AI agents**.

The starting point is Stanford's **CS329A: Self-Improving AI Agents**, but the goal is not to reproduce lecture notes or summarize papers one by one.

Instead, this repository asks a more practical systems question:

> **How do research ideas about search, verification, feedback, and scaling translate into real agent architectures?**

## Why This Repository Exists

Modern agent systems are shaped by ideas that go beyond simply choosing a stronger language model:

- inference-time scaling
- repeated sampling and Best-of-N selection
- verifier-guided search
- adaptive compute allocation
- planning and long-horizon reasoning
- feedback-driven improvement
- agentic evaluation
- memory and context management
- authority and safety boundaries

A recurring theme is that model capability alone does not determine system capability.

```text
Model
  + Search
  + Verification
  + Orchestration
  + Evaluation
  + Constraints
  = Agent System
```

## Learning Approach

```text
Research question
      ↓
Stanford CS329A reading list
      ↓
Paper comparison
      ↓
Core algorithm / architecture
      ↓
2026 freshness check
      ↓
Mapping to real agent systems
      ↓
Apply / Keep in Mind / Skip
```

The default principle is:

```text
Paper First
Video Optional
```

Lecture videos are used only when historical context or an instructor's intuition materially improves understanding.

## Research-to-System Mapping

The notes are not purely theoretical. Research ideas are compared against practical agent-architecture work, especially **DHAA** and selected product experiments.

### DHAA — Domain / Harness / Agent Agnostic

[DHAA](https://github.com/Seung-zedd/dhaa) explores workflow continuity, evidence-first decision routing, deterministic guardrails, and explicit authority boundaries.

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

This provides a useful counterpoint to research on inference-time search and verification: **more compute may improve judgment quality, but it does not automatically expand what the system is allowed to decide.**

## Study Roadmap

| # | Topic | Priority | Status |
|---:|---|:---:|---|
| 01 | Test-Time Compute Scaling | ★★★★★ | In Progress |
| 02 | Robust Verification | ★★★★★ | Planned |
| 03 | Agentic Evaluations & Long-Horizon Tasks | ★★★★★ | Planned |
| 04 | Multi-Step Reasoning & Planning | ★★★★★ | Planned |
| 05 | Learning from Feedback | ★★★★☆ | Planned |
| 06 | Open-Ended Self-Improvement | ★★★★☆ | Planned |
| 07 | Search & Deep Research Agents | ★★★★☆ | Planned |
| 08 | Agentic Software Engineering | ★★★★☆ | Planned |
| 09 | Memory & Context | ★★★☆☆ | Planned |
| 10 | Train-Time Scaling & Reinforcement Learning | ★★☆☆☆ | Planned |

The course overview is intentionally not treated as a standalone study unit. Foundational concepts are introduced when they become necessary.

## Wiki

Detailed study notes live under [docs/wiki](./docs/wiki).

Each chapter follows roughly the same structure:

```text
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
```

The goal is not exhaustive paper reproduction.

A successful chapter should leave behind:

> **1–3 durable concepts and at least one useful connection to a real system.**

## Core Principle

A central idea explored throughout this repository is the separation between **capability** and **authority**.

```text
Inference-Time Scaling
= search harder inside an allowed space

Authority Guard
= define which space may be searched or acted upon
```

More computation does not automatically grant more authority. Likewise, a stronger verifier does not automatically make an action permissible.

## Status

This repository evolves as each topic is studied. Notes may change when newer research changes an interpretation, experiments reveal a counterexample, an architecture is implemented in practice, or a better abstraction emerges.

**Negative results are treated as first-class learning artifacts.**

## References

Primary starting point:

- Stanford CS329A — *Self-Improving AI Agents*: https://cs329a.stanford.edu/
- [DHAA](https://github.com/Seung-zedd/dhaa)

## License

Released under the [MIT License](./LICENSE).

## Disclaimer

This is an independent study repository.

It is **not affiliated with, endorsed by, or an official publication of Stanford University or the CS329A course staff**.
