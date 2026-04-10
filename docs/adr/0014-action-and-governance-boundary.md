# 0014 — Action and Governance Boundary

**Date:** 2026-04-01
**Status:** Accepted

## Context

TraceForward gives coding agents runtime context from observability backends before they modify services. A natural architectural question follows: should TraceForward also allow agents to act on what they observe, such as restarting a service, scaling a deployment, rolling back a release, or modifying alerting configuration?

This boundary defines TraceForward’s identity, security posture, and trust model. Observability systems and action systems have different risk profiles. Read-only access to signals is materially different from write access to infrastructure or deployment controls.

Adjacent systems such as deployment pipelines, IaC controllers, and incident platforms already own execution and mutation. TraceForward’s role is to provide the evidence that informs decisions, not to perform the decisions.

## Decision

TraceForward is **read-only by design**. Its tools observe, query, and correlate signal data. They do not mutate environment state.

Specifically:

* **No write operations.** TraceForward tools do not restart services, modify configurations, create or delete resources, trigger deployments, or alter alerting or routing rules.
* **No action recommendations.** Tool responses return evidence such as traces, metrics, logs, errors, and service maps. They do not prescribe remediation steps or recommended actions.
* **No autonomous behavior.** TraceForward does not schedule queries, poll for changes, or push results proactively. Every query is initiated by the caller.
* **Governance is external.** Decisions about whether and how to act on TraceForward data belong to the consuming agent framework, operator policy, or human reviewer, not to TraceForward itself.

### Language enforcement

Consistent with ADR-0006:

* Do not describe TraceForward as autonomous, automatic, or decision-making
* Use **environment** rather than **production**
* Tool descriptions explain what question a tool answers, not what action should be taken

### Governance boundary

TraceForward may be used by higher-level governance or orchestration layers that decide whether and how agents act on observed data. Those policy decisions are outside TraceForward’s scope. TraceForward’s responsibility ends at delivering trusted signal data.

## Consequences

* **Clear system identity.** TraceForward remains an intelligence provider rather than an orchestrator or control plane.
* **Smaller trust surface.** Read-only access to observability data is easier to secure and reason about than write access to environment controls.
* **Composable design.** TraceForward can be combined with separate action-oriented tools without overlapping responsibilities.
* **Contributor clarity.** Proposed tools can be evaluated against a simple rule: if the tool changes the environment, it does not belong in TraceForward.
* **Trade-off.** Some users will want an observe-and-act experience in a single system. TraceForward intentionally does not provide that.
* **Trade-off.** Because TraceForward does not recommend actions, some of its value depends on the consuming agent or human being able to interpret the evidence correctly.
