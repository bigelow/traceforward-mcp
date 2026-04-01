# 0014 — Action and Governance Boundary

**Date:** 2026-04-01
**Status:** Accepted

## Context

TraceForward gives coding agents runtime context from an OpenTelemetry backend before they modify services. A natural question arises: should TraceForward also let agents *act* on what they observe — restart a service, scale a deployment, roll back a release, create an alert rule?

This question becomes more pressing as MCP adoption grows and agents gain broader tool access. The boundary between "observe" and "act" defines TraceForward's identity, its security posture, and the trust model between operators and agents.

Adjacent tools in the ecosystem (IaC controllers, deployment pipelines, incident response platforms) already own the action space. TraceForward's value is upstream of those tools — it provides the intelligence that informs whether an action should be taken, not the mechanism to take it.

## Decision

TraceForward is **read-only by design**. Its tools observe, query, and correlate signal data. They never mutate environment state.

Specifically:

- **No write operations.** TraceForward tools shall not restart services, modify configurations, create or delete resources, trigger deployments, or alter alert/routing rules.
- **No action recommendations.** Tool responses return signal data — traces, metrics, logs, errors, service maps. They do not include suggested remediation steps, runbook links, or "next action" guidance. The consuming agent (or human) decides what to do with the data.
- **No autonomous behavior.** TraceForward does not schedule queries, poll for changes, or proactively push data. Every query is initiated by the caller.
- **Governance lives at the agent layer.** Decisions about whether to act on TraceForward's data — and what guardrails apply — belong to the agent framework, the operator's policies, or the human in the loop. TraceForward's role ends at delivering trusted signal data.

### Language enforcement

Consistent with ADR-0006 (Language and Terminology Conventions):

- Never use "autonomous," "automatically," or "AI makes decisions" in documentation, tool descriptions, or response content.
- Never use "production" — use "environment."
- Tool descriptions describe what question the tool answers, not what action to take based on the answer.

### Boundary with Meridian

TraceForward is the Observe→Generate bridge in the CDLC framework (attributed to Patrick Debois). Meridian's governance layer may eventually enforce policies about *when* and *whether* agents act on TraceForward data. That governance is Meridian's responsibility, not TraceForward's. TraceForward supplies the signal; Meridian (or another governance layer) supplies the policy.

## Consequences

- **Clear identity.** TraceForward is an intelligence provider, not an orchestrator. This simplifies the security model — read-only access to an observability backend is a much smaller trust surface than write access to environment infrastructure.
- **Composability.** Agents can combine TraceForward's signal data with action tools from other MCP servers (deployment, IaC, incident management) without capability overlap or conflict.
- **Contributor clarity.** New tool proposals are evaluated against a simple test: does this tool answer a question about the environment, or does it change the environment? Only the former belongs in TraceForward.
- **Trade-off.** Some users will want "observe and act" in a single tool. TraceForward intentionally does not serve this use case. The answer is composability — pair TraceForward with an action-oriented MCP server.
- **Trade-off.** No action recommendations means TraceForward's value depends on the consuming agent's ability to reason about raw signal data. Agents that struggle with interpretation get less value. This is acceptable — improving agent reasoning is not TraceForward's problem to solve.
