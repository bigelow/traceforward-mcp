# 0013 — Tool Response Contract

**Date:** 2026-03-25
**Status:** Accepted

## Context

TraceForward’s MCP tools return signal data to coding agents. The tool surface is defined in ADR-0003, but the response shape must also be explicit. Without a shared response contract, each tool would define its own output structure, metadata, and partial-failure behavior, making consistent agent reasoning difficult and contributor implementation error-prone.

Several architectural decisions depend on response shape being standardized:

* ADR-0009 defines error categories but not how errors coexist with partial results
* ADR-0011 defines redaction markers but not where they appear in responses
* ADR-0012 defines canonical entities but not how they are represented in tool output

TraceForward does not just return data. It returns evidence that agents use to make judgments. The response contract therefore determines whether that evidence is trustworthy, incomplete, stale, or contradictory.

## Decision

TraceForward will return all MCP tool results in a standard response envelope.

The envelope separates signal-specific data from the metadata an agent needs to assess the evidence: what was queried, how complete the result is, how fresh it is, what warnings apply, and what errors occurred.

### Response envelope

Every tool response uses this structure:

```json
{
  "data": [],
  "metadata": {
    "tool": "traceforward_get_traces",
    "service": {
      "name": "payment-api",
      "environment": "staging"
    },
    "query": {
      "lookback_hours": 24,
      "limit": 50,
      "filters": {}
    },
    "freshness": {
      "query_time": "2026-03-25T14:32:01Z",
      "newest_signal": "2026-03-25T14:28:44Z",
      "oldest_signal": "2026-03-25T02:15:03Z"
    },
    "completeness": {
      "status": "complete",
      "returned": 47,
      "limit": 50,
      "truncated": false,
      "gaps": []
    },
    "adapter": {
      "name": "grafana-lgtm",
      "backend_latency_ms": 230
    }
  },
  "warnings": [],
  "errors": []
}
```

### Envelope semantics

#### `data`

`data` contains the signal-specific results for the tool. The shape varies by tool, but every item may include canonical entity references and `source_attributes`.

When no results match the query, `data` is an empty list. Empty results are a valid outcome, not a failure.

#### `metadata`

`metadata` describes the evidence returned:

* **`tool`** — the tool that produced the response
* **`service`** — canonical service reference, when applicable
* **`query`** — effective query parameters, including defaults
* **`freshness`** — when the query ran and the temporal bounds of the returned evidence
* **`completeness`** — whether the result is complete, partial, empty, or degraded
* **`adapter`** — which adapter produced the response and backend latency information

#### `completeness`

`completeness` is the primary trust signal in the envelope.

* **`complete`** — all matching results within the requested bounds are included
* **`partial`** — some results are present, but part of the response is missing
* **`empty`** — the query succeeded but returned no matching data
* **`degraded`** — data was returned, but the adapter could not guarantee completeness or reliability

If the result hit the configured limit, `truncated` is set to `true`.

When status is `partial` or `degraded`, `gaps` describes what is missing and why.

Example:

```json
{
  "signal_type": "metrics",
  "reason": "backend_timeout",
  "message": "Metrics backend timed out after 10s. Traces and logs are included."
}
```

#### `warnings`

`warnings` capture non-fatal conditions such as:

* redaction applied
* stale data detected
* defaults applied
* backend degradation
* conflicting signals

Warnings are structured objects with `category` and `message`.

#### `errors`

`errors` capture failures using the normalized categories from ADR-0009. Errors may coexist with `data` when `completeness.status` is `partial`.

### Honest uncertainty

TraceForward is expected to represent uncertainty directly.

* **`empty`** means: the query ran successfully and found nothing
* **`partial`** means: some evidence is available, but part is missing
* **`degraded`** means: evidence is available, but completeness or reliability is uncertain
* **conflicting signals** are surfaced as warnings rather than resolved by TraceForward

TraceForward provides evidence, not conclusions.

### Freshness

TraceForward does not define a universal staleness rule. Instead, it reports freshness facts:

* `query_time`
* `newest_signal`
* `oldest_signal`

If `query_time - newest_signal` exceeds a configurable threshold, TraceForward adds a `staleness` warning. The threshold is operator-configurable.

### Provenance

Each item in `data` carries:

* **`adapter`**
* **`source_attributes`**

TraceForward provenance means: this data came from this adapter, querying this backend, at this time.

Full backend lineage is out of scope.

## Consequences

* **Consistent agent reasoning.** All tools return the same outer structure, so agents can assess completeness, freshness, and failures uniformly.
* **Honest uncertainty.** TraceForward can explicitly represent empty, partial, degraded, and conflicting evidence without forcing false certainty.
* **Partial results are first-class.** Data, warnings, and errors can coexist in a single response.
* **Debuggability.** Query metadata, backend latency, and provenance make tool behavior easier to inspect.
* **Trade-off.** The envelope adds metadata overhead to every response.
* **Trade-off.** TraceForward does not compute confidence scores or resolve contradictions. It exposes evidence and leaves judgment to the caller.
* **Trade-off.** Freshness warnings depend on operator-configured thresholds, so behavior may vary between deployments.
