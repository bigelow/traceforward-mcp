# 0006 — Language and Terminology Conventions

**Date:** 2026-03-24
**Status:** Accepted

## Context

TraceForward’s user-facing language shapes how the project is understood by contributors, users, and agent environments. Inconsistent or overstated terminology can misrepresent the scope of the system and create the wrong expectations about what TraceForward does.

Two terminology risks are especially important:

1. **“Production” is too narrow.** TraceForward is intended to work with any environment that emits telemetry, including staging, canary, preview, and production.
2. **Autonomy language overstates the system’s role.** Terms such as “autonomous,” “automatically,” or claims that “AI makes decisions” imply independent action. TraceForward does not act on systems. It exposes signal data and runtime context to agents and humans.

## Decision

TraceForward will apply the following terminology conventions across documentation, tool descriptions, error messages, README content, and other public-facing text:

* Use **“environment”** rather than **“production”** unless the distinction is explicitly required.
* Do not describe TraceForward as **autonomous**, **automatic**, or as a system that **makes decisions**.
* Describe TraceForward as exposing runtime context, signal data, or observability information to agents and humans, not as taking action on their behalf.

These conventions are part of the project’s public interface and will be enforced through review and automated convention checks.

## Consequences

* **Accurate scope.** “Environment” reflects the intended deployment model more accurately than “production.”
* **Clear trust boundary.** Avoiding autonomy language makes it clear that TraceForward informs decisions but does not execute them.
* **Consistent project voice.** Contributors have explicit guidance for user-facing language across docs, tools, and code comments.
* **Reduced expectation drift.** Readers are less likely to infer unsupported capabilities such as independent action or production-only scope.
* **Trade-off.** Some contributors will naturally reach for familiar shorthand such as “production” or “automatic.” The convention is simple, but it must be reinforced in review and linting.
* **Trade-off.** Strict language rules can feel overly editorial if they are not tied to architecture. In TraceForward’s case, these terms define the system boundary and are therefore architectural, not cosmetic.
