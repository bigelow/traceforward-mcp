# 0013 — Tool Response Contract

**Date:** 2026-03-25
**Status:** Accepted

## Context

TraceForward's six MCP tools (ADR-0003) return signal data to coding agents. The tool surface is defined, but not the shape of what comes back. Without a response contract, each tool invents its own output structure, partial-failure handling, and metadata conventions. Agents cannot reason consistently across tools, and contributors cannot implement new tools or adapters without guessing at the expected output.

Several architectural decisions depend on response shape being explicit:

- ADR-0009 (error handling) defines error categories but not how errors coexist with partial results in a single response.
- ADR-0011 (redaction) applies redaction markers (`[REDACTED]`, `[topology restricted]`) but does not define where those markers appear in the response structure.
- ADR-0012 (canonical entities) defines the identity model but not how canonical entities are represented in tool output.
- The ChatGPT review identified truth, confidence, freshness, and "I don't know" signals as essential for preventing agents from speaking with false certainty.

The harder version of this problem: TraceForward is not just returning data. It is providing evidence that agents use to make judgments. The response contract determines whether that evidence is trustworthy or misleading.

## Decision

All TraceForward MCP tools return a **standard response envelope** that wraps signal-specific data with metadata an agent needs to assess the evidence.

### Response envelope

Every tool response contains:

```
{
  "data": [ ... ],              # signal-specific results
  "metadata": {
    "tool": "traceforward_get_traces",
    "service": { "name": "payment-api", "environment": "staging" },
    "query": {
      "lookback_hours": 24,
      "limit": 50,
      "filters": { ... }
    },
    "freshness": {
      "query_time": "2026-03-25T14:32:01Z",
      "newest_signal": "2026-03-25T14:28:44Z",
      "oldest_signal": "2026-03-25T02:15:03Z"
    },
    "completeness": {
      "status": "complete | partial | empty | degraded",
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

### Envelope fields

#### `data`

The signal-specific results. Shape varies by tool (traces, metrics, logs, service map, error summaries, service list). Each item in `data` carries canonical entity references (ADR-0012) and may carry `source_attributes` for backend-specific evidence.

When no results match the query, `data` is an empty list — not null, not omitted. An empty list means "I looked and found nothing," which is different from a failure.

#### `metadata`

Context about the query and the evidence returned.

- **`tool`** — which tool produced this response.
- **`service`** — the canonical service reference queried, when applicable.
- **`query`** — the effective query parameters, including defaults that were applied. This lets the agent know what it actually asked for, not just what it thought it asked for.
- **`freshness`** — temporal bounds of the evidence. `query_time` is when TraceForward executed the query. `newest_signal` and `oldest_signal` are the timestamps of the most and least recent signals in the result set. If the newest signal is hours old, the agent should reason about whether the data is stale. TraceForward surfaces the facts; the agent makes the judgment.
- **`completeness`** — whether the response represents the full picture or a partial view.
- **`adapter`** — which adapter produced the data and how long the backend took to respond. Useful for debugging, not for agent reasoning.

#### `completeness` semantics

Completeness is the most important metadata field. It tells the agent how much to trust the result set:

- **`complete`** — the query returned all matching results within the requested bounds. The agent can reason over this data with full confidence that nothing was omitted.
- **`partial`** — some results were returned but the response is incomplete. The `gaps` field describes what is missing (e.g., "metrics backend unreachable — traces and logs included, metrics omitted"). Consistent with ADR-0009: partial results over total failure.
- **`empty`** — the query executed successfully but returned no matching data. This is a meaningful answer, not an error. "No errors in the last 24 hours" is useful intelligence.
- **`degraded`** — the response was returned but the adapter reported reduced reliability (e.g., the backend was under load, returned a timeout on some partitions, or served from a stale cache). Data is present but its completeness cannot be guaranteed.

The `truncated` flag indicates whether the result count hit the `limit`. If `truncated` is true, the agent knows more data exists beyond what was returned.

The `gaps` array lists specific missing components when status is `partial` or `degraded`:

```
"gaps": [
  {
    "signal_type": "metrics",
    "reason": "backend_timeout",
    "message": "Metrics backend timed out after 10s. Traces and logs are included."
  }
]
```

#### `warnings`

Non-fatal conditions the agent should be aware of:

- Redaction was applied (`"Fields redacted per policy — credential patterns detected in trace headers"`)
- Stale data detected (`"Newest signal is 6 hours old — backend may have ingestion lag"`)
- Adapter reported degraded performance
- Query defaults were applied because the agent did not specify parameters

Warnings are structured: each warning has a `category` (e.g., `redaction`, `staleness`, `defaults_applied`, `backend_degraded`) and a `message`.

#### `errors`

Errors that prevented part or all of the response from being produced. Uses the error categories from ADR-0009 (`connection_error`, `query_error`, `not_found`, `validation_error`, `authentication_error`, `authorization_error`). Errors and data can coexist in the same response when `completeness.status` is `partial`.

### "I don't know" as a first-class response

TraceForward is allowed — and expected — to say it does not know. This is not a failure mode. It is an honest answer.

Specific forms:

- **`completeness.status: "empty"` with no errors** — "I looked and found nothing." The agent should treat this as evidence of absence within the query bounds, not as a system failure.
- **`completeness.status: "partial"` with gaps** — "I have some evidence but not all of it. Here is what I have and what I am missing." The agent should reason with reduced confidence.
- **`completeness.status: "degraded"`** — "I have data but I cannot guarantee it is complete. The backend was unreliable during this query." The agent should treat this as provisional evidence.
- **Contradictory signals** — when data from different signal types within the same response appears contradictory (e.g., traces show success but logs show errors for the same time window), TraceForward does not resolve the contradiction. It returns both signals with their respective metadata and adds a warning: `"category": "conflicting_signals"`. The agent decides which evidence to weight. TraceForward provides intelligence, not conclusions (ADR-0006).

### Freshness and staleness

TraceForward does not define a global staleness threshold. What counts as "stale" depends on the operational context — a 6-hour-old metric may be fine for a nightly batch service and dangerously outdated for a high-frequency trading system.

Instead, TraceForward surfaces freshness facts:

- `freshness.query_time` — when the query ran.
- `freshness.newest_signal` — the most recent data point.
- `freshness.oldest_signal` — the earliest data point in the result.

If the gap between `query_time` and `newest_signal` exceeds a configurable threshold (default: 1 hour), TraceForward adds a `staleness` warning. The threshold is operator-configurable, not hardcoded.

The agent receives the freshness data and the warning. Whether to trust stale data is the agent's judgment, not TraceForward's.

### Provenance

Every item in the `data` array carries:

- **`adapter`** — which adapter produced this item.
- **`source_attributes`** — backend-specific metadata preserved as evidence (ADR-0012).

This is not a full provenance chain. TraceForward does not track data lineage through the backend's own pipeline (e.g., whether Loki ingested a log entry with delay, or whether a metric was aggregated before reaching TraceForward). Provenance in TraceForward means: "this data came from this adapter, querying this backend, at this time."

Full backend provenance is out of scope. TraceForward is honest about what it knows and does not fabricate certainty about what it does not.

## Consequences

- **Consistent agent reasoning.** Every tool returns the same envelope. Agents parse one structure, assess completeness and freshness uniformly, and handle partial results with a single code path.
- **Honest uncertainty.** Completeness status, gap descriptions, staleness warnings, and conflicting-signal markers give agents the information to calibrate confidence. TraceForward never forces false certainty.
- **Partial results work.** The envelope explicitly supports coexistence of data, warnings, and errors. An agent investigating a service gets logs and traces even when the metrics backend is down.
- **Debuggable.** Query metadata shows what was actually queried (including applied defaults), adapter latency surfaces backend performance issues, and provenance traces data to its source.
- **Trade-off.** The envelope adds metadata overhead to every response. For small result sets, metadata may be a significant fraction of the payload. This is acceptable — the metadata is what makes the evidence trustworthy.
- **Trade-off.** TraceForward does not resolve contradictions or compute confidence scores. It surfaces evidence and lets the agent reason. This is a deliberate product decision (consistent with ADR-0006) but means agents must be sophisticated enough to handle ambiguity.
- **Trade-off.** Freshness thresholds are operator-configurable, which means agents in different deployments may receive staleness warnings at different thresholds. This is intentional — operational context varies — but means agent behavior is not fully portable across TraceForward instances.
