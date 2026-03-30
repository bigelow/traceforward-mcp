# 0006 — Language and Terminology Conventions

**Date:** 2026-03-24
**Status:** Accepted

## Context

TraceForward's public-facing language — documentation, tool descriptions, error messages, and marketing — shapes how people understand what the project does and how it fits into their workflow. Inconsistent or loaded terminology creates confusion and missets expectations.

Two specific concerns:

1. **"Production" implies a specific environment.** Many teams have staging, canary, or preview environments that also produce valuable signals. TraceForward works with any environment that emits telemetry.
2. **"Autonomous" / "automatically" / "AI makes decisions" implies unsupervised action.** TraceForward provides intelligence *to* agents and humans — it does not take action itself. Overstating AI agency erodes trust and misrepresents the tool's role.

## Decision

The following terminology rules apply to all TraceForward documentation, tool descriptions, README content, and public communications:

- **Always use "environment"** — never "production." Example: "Before your agent touches the payment service, it checks how that service actually behaves in your environment."
- **Never use "autonomous," "automatically," or "AI makes decisions."** TraceForward delivers signal data. Agents and humans decide what to do with it. Frame TraceForward as providing intelligence, not taking action.

These conventions are enforced through documentation review, not tooling.

## Consequences

- **Broader applicability.** "Environment" correctly describes TraceForward's scope — any environment with telemetry, not just production.
- **Trust.** Avoiding autonomy language sets accurate expectations about what TraceForward does and does not do.
- **Contributor clarity.** New contributors have explicit guidance on voice and framing, reducing inconsistency in PRs.
- **Trade-off.** "Production" is a well-understood shorthand that many practitioners reach for instinctively. Contributors may need reminders. The convention is simple enough to enforce in review.
