# 0012 — Canonical Entity and Identity Model

**Date:** 2026-03-25
**Status:** Accepted

## Context

TraceForward ingests signal data from diverse observability backends that each define operational concepts differently. Datadog identifies services through tags. Grafana derives service identity from span attributes. Loki uses label sets. Prometheus uses metric label combinations. Each backend has its own notion of what a "service" is, what an "environment" means, and how deployments are represented.

Without a canonical entity model, TraceForward cannot correlate signals across backends, apply consistent redaction policy (ADR-0011), or produce reliable tool responses. Two adapters returning data about the same service under different names would produce contradictory results. An agent asking "what's failing in staging?" needs a single, stable definition of both "service" and "staging" — not backend-specific interpretations.

The risk of over-normalizing is equally real. Backends carry rich, context-specific metadata (Datadog tags, Kubernetes labels, custom span attributes) that is valuable as evidence. Forcing all backend attributes into a rigid canonical schema destroys useful nuance without improving correlation.

## Decision

TraceForward defines a **canonical entity model** for the entities the core reasons over. Backend-specific attributes are preserved as attributed evidence, not discarded.

The design principle: **canonicalize what the core reasons over. Preserve everything else as attributed evidence.**

### MVP entity types

The MVP defines three canonical entity types. This is the minimum set that lets TraceForward answer: what service, in which environment, what changed.

#### Service

The primary operational target. A service is a distinct, named component in the caller's system that produces telemetry.

**Required fields:**
- **`name`** — the canonical service identifier. Adapters map backend-specific service concepts (Datadog service tags, OTel `service.name` attribute, Loki label values) to this field.
- **`environment`** — the deployment context. Maps from backend-specific environment indicators (`env` tag, `deployment.environment` attribute, namespace labels). Consistent with ADR-0006: always "environment," never "production."

**Optional qualifiers:**
- **`namespace`** — Kubernetes namespace or equivalent logical grouping.
- **`cluster`** — cluster identifier when the service spans or distinguishes between clusters.
- **`region`** — cloud region or data center identifier.

**Canonical identity rule:** Two service references are the same service if and only if `name` and `environment` match. Optional qualifiers refine but do not redefine identity. A service named `payment-api` in `staging` is the same canonical service regardless of which cluster or region it runs in — qualifiers describe *where* it runs, not *what* it is.

#### Environment

A deployment context in which services run and produce telemetry. Environments are not enumerated by TraceForward — they are discovered from signal data through adapters.

**Required fields:**
- **`name`** — the environment identifier (e.g., `staging`, `canary`, `us-east-1-prod`).

**Optional fields:**
- **`tier`** — an operator-defined classification (e.g., `development`, `staging`, `production`) for environments where the name alone does not convey operational risk level.

Environments are first-class because redaction policy (ADR-0011), error disclosure (ADR-0009), and trust boundaries (ADR-0008) all vary by deployment context. Making environment explicit — rather than embedded in service names or inferred from backend metadata — ensures these policies can be applied consistently.

#### Deploy (change)

A recorded change event associated with a service in an environment. Deploys are the first causal bridge: "what changed that might explain this behavior?"

**Required fields:**
- **`service`** — canonical service reference (`name` + `environment`).
- **`timestamp`** — when the deploy occurred, normalized to UTC.
- **`identifier`** — a backend-native deploy identifier (commit SHA, release tag, deployment ID, rollout name). TraceForward does not define what a "deploy" looks like — it accepts whatever the adapter provides as the identifier.

**Optional fields:**
- **`version`** — application version string, if available.
- **`initiator`** — who or what triggered the deploy (user, CI pipeline, rollback controller).
- **`status`** — deploy outcome if known (succeeded, failed, in-progress, rolled-back).

Deploy is not part of service identity. A service's identity is `name` + `environment`. Deploys are events that happen *to* a service — they provide temporal context for signal correlation, not identity.

### Entities deferred beyond MVP

- **Owner** — the team or individual responsible for a service. Valuable for routing and escalation, but not required for signal correlation.
- **Incident** — a declared operational event spanning multiple services and signals. Requires an incident source (PagerDuty, Opsgenie, etc.) and cross-service correlation that depends on the canonical model being stable first.

These are high-value enrichments. They should be added once the MVP entity types prove stable under real adapter usage.

### Adapter normalization contract

Adapters are responsible for mapping backend-specific concepts to canonical entities:

- Every adapter must populate `service.name` and `service.environment` on every signal response. If the backend does not provide an explicit environment, the adapter must either derive it from available metadata or fail with a `validation_error` (ADR-0009) — not silently omit it.
- Adapters map backend-native identifiers to canonical fields using adapter-specific logic. A Datadog adapter maps the `service` tag to `service.name` and the `env` tag to `service.environment`. A Grafana/Tempo adapter maps `service.name` and `deployment.environment` span attributes. The mapping is adapter-owned, but the output shape is core-defined.
- When a backend's service concept does not map cleanly (e.g., a metrics backend that has no native service concept, only metric label combinations), the adapter documents its mapping strategy and surfaces the ambiguity. Guessing silently is worse than surfacing uncertainty.

### Evidence preservation

Backend-specific attributes that do not map to canonical fields are preserved as **source attributes** on the response model:

```
source_attributes: {
    "backend": "datadog",
    "raw_tags": {"team": "payments", "cost_center": "eng-42"},
    "original_service_tag": "payment-api-v2"
}
```

Source attributes are:
- **Read-only.** The core does not reason over them — they exist for debugging, auditing, and advanced agent use.
- **Subject to redaction.** Source attributes pass through the core redaction layer (ADR-0011) like any other field. Credential and PII patterns are stripped regardless of origin.
- **Not used for identity.** Two signals with different source attributes but the same `service.name` and `service.environment` are signals about the same service.

## Consequences

- **Stable correlation.** `name` + `environment` as the canonical identity gives the core a reliable join key across adapters, tools, redaction policy, and error messages.
- **Adapter clarity.** The normalization contract is explicit: populate these fields, in this shape. Adapter authors know exactly what they must produce.
- **Evidence preserved.** Backend-specific richness is not destroyed. Agents that need raw Datadog tags or Kubernetes labels can access source attributes without the core depending on them.
- **Causal bridge.** Deploy as a first-class entity lets TraceForward answer "what changed?" — the question that turns signal retrieval into judgment support.
- **Trade-off.** The canonical identity rule (`name` + `environment`) means two services with the same name in the same environment are the same service, even if they run in different clusters. This is intentional — cluster is a qualifier, not an identity discriminator. Operators who need cluster-level discrimination can encode it in the service name or environment.
- **Trade-off.** Requiring adapters to always populate `environment` adds friction for backends that don't have an explicit environment concept. This is deliberate — environment is too central to redaction, disclosure, and trust policy to be optional.
- **Trade-off.** Deferring owner and incident means the MVP cannot answer "who owns this?" or "is this part of a known incident?" These are valuable but depend on stable service identity existing first.
