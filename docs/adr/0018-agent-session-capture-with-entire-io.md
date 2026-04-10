# 0018 — Agent Session Capture

**Date:** 2026-04-01
**Status:** Accepted

## Context

TraceForward is developed with the assistance of coding agents under explicit architectural governance. When an agent produces code that a human reviews and commits, the implementation session that led to that code contains useful context that is not preserved by source control alone.

Git captures what changed. Commit messages capture a summary of why. Neither captures how the implementation was developed: the prompts given to the agent, the alternatives explored, the iterations taken, or the constraints applied during the session.

Without session capture, agent-assisted development leaves a gap between architectural intent and committed implementation. For a project that treats governance and traceability as part of its engineering discipline, that gap is material.

## Decision

TraceForward captures agent implementation sessions as part of its development audit trail.

Session capture preserves the implementation context between architectural decisions and committed code. It is used to provide an additional traceability layer between:

* **ADR** — why a decision exists
* **session** — how the implementation was developed
* **commit** — what code was ultimately recorded

The current mechanism used for session capture is **Entire.io**, which records commit-correlated agent checkpoints alongside the normal git workflow. Entire.io is an implementation choice, not the architectural invariant. The architectural decision is that session capture exists; the specific tool may change later.

## What is captured

* **Commit-correlated session checkpoints** that preserve the prompts, outputs, and implementation context associated with agent-assisted development
* **Session-to-commit linkage** so contributors can trace from committed code back to the development session that produced it

## What is not captured

* **Human review decisions** beyond what is visible in the final commit history, diffs, or code review system
* **Non-committed exploratory work** unless explicitly preserved through the chosen session capture mechanism

## Integration boundaries

* Session capture is part of the **development workflow**, not the TraceForward runtime
* Session capture does not replace **git**, code review, or ADR references
* Human review and human commits remain the authoritative acceptance step for any agent-produced code

## Consequences

* **Improved traceability.** The project preserves more of the path from decision to implementation.
* **Better auditability.** Contributors can inspect how agent-assisted changes were developed, not just the final code that landed.
* **Tool flexibility.** The architectural commitment is to session capture itself, not to a single vendor or product.
* **Trade-off.** Session capture introduces an additional workflow dependency beyond source control.
* **Trade-off.** If the current capture tool becomes unavailable, development can continue, but the additional audit layer is lost until replaced.
* **Trade-off.** Session capture is still narrower than full development history. Informal reasoning, verbal discussion, and non-committed exploration may remain outside the recorded trail.
