# 0012 — Canonical Entity and Identity Model

**Date:** 2026-03-25
**Status:** Accepted

## Context

TraceForward ingests signal data from observability backends that define operational concepts differently. Datadog uses tags, Grafana and Tempo derive identity from span attributes, Loki uses label sets, and Prometheus relies on metric label combinations. Without a canonical entity model, TraceForward cannot correlate signals across adapters, apply consistent disclosure policy, or return stable tool results.

Over-normalization is also a risk. Backend-specific metadata such as Datadog tags, Kubernetes labels, or custom span attributes may be valuable as evidence even when they should not define identity.

## Decision

TraceForward defines a canonical entity model for the entities the core reasons over. Backend-specific metadata is preserved as attributed evidence rather than discarded.

The governing principle is:

**Canonicalize what the core reasons over. Preserve everything else as source-attributed evidence.**

### MVP entity types

The MVP defines three canonical entity types: **Service**, **Environment**, and **Deploy**.

#### Service

A service is the primary operational entity TraceForward reasons about.

**Required fields:**

* **`name`** — canonical service identifier
* **`environment`** — canonical deployment context

**Optional qualifiers:**

* **`namespace`**
* **`cluster`**
* **`region`**

**Canonical identity rule:**
Two service references are the same canonical service if and only if `name` and `environment` match. Optional qualifiers refine context but do not change identity.

#### Environment

An environment is a deployment context in which services run and emit telemetry.

**Required fields:**

* **`name`**

**Optional fields:**

* **`tier`** — operator-defined classification such as `development`, `staging`, or `production`

Environments are first-class because disclosure, trust, and redaction behavior depend on deployment context.

#### Deploy

A deploy is a recorded change event associated with a service in an environment.

**Required fields:**

* **`service`** — canonical service reference
* **`timestamp`** — normalized UTC timestamp
* **`identifier`** — backend-native deploy identifier

**Optional fields:**

* **`version`**
* **`initiator`**
* **`status`**

Deploys are not part of service identity. They are causal events associated with a service.

### Deferred entity types

The following entity types are explicitly deferred beyond the MVP:

* **Owner**
* **Incident**

These are valuable enrichments, but they depend on stable service identity first.

### Adapter normalization contract

Adapters are responsible for mapping backend-specific concepts into the canonical model.

* Every adapter must populate `service.name` and `service.environment` on signal responses.
* If a backend does not provide an explicit environment, the adapter must derive it from available metadata or fail with `validation_error`.
* Adapter-specific mapping logic is adapter-owned, but the canonical output shape is core-defined.
* If a backend concept does not map cleanly, the adapter must document the ambiguity rather than silently guess.

### Evidence preservation

Backend-specific attributes that do not map to canonical fields are preserved as **source attributes** on response models.

Example:

```json
{
  "source_attributes": {
    "backend": "datadog",
    "raw_tags": {
      "team": "payments",
      "cost_center": "eng-42"
    },
    "original_service_tag": "payment-api-v2"
  }
}
```

Source attributes are:

* **Read-only** — the core does not use them for reasoning or identity
* **Subject to redaction** — they pass through the core disclosure and redaction policy
* **Not identity-bearing** — differing source attributes do not create distinct canonical services if `name` and `environment` match

## Consequences

* **Stable correlation.** `name` + `environment` provides a consistent join key across adapters, tools, and policy layers.
* **Explicit adapter contract.** Adapter authors know exactly which fields must be produced and normalized.
* **Evidence preserved.** Backend-specific richness remains available without leaking into core identity rules.
* **Causal context.** Deploy as a first-class entity enables reasoning about what changed, not just what is failing.
* **Trade-off.** `name` + `environment` intentionally treats services with the same name in different clusters as the same service unless that distinction is encoded elsewhere.
* **Trade-off.** Requiring `environment` everywhere increases adapter burden for backends that do not model it explicitly.
* **Trade-off.** Deferring `Owner` and `Incident` limits MVP scope, but keeps the canonical model focused and stable.
