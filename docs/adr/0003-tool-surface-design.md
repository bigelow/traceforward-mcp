# 0003 — Tool Surface Design

**Date:** 2026-03-24
**Status:** Accepted

## Context

MCP tools are the interface coding agents use to interact with TraceForward. The tool surface must balance completeness with simplicity: agents need enough signal access to make informed decisions, but they perform better with a small set of clearly named tools than with a large or overlapping interface.

TraceForward ingests three primary signal types — traces, metrics, and logs — and also needs to expose higher-order views such as service topology and error patterns.

## Decision

TraceForward will expose six MCP tools:

| Tool                           | Signal Source | Type    |
| ------------------------------ | ------------- | ------- |
| `traceforward_get_traces`      | Traces        | Direct  |
| `traceforward_get_metrics`     | Metrics       | Direct  |
| `traceforward_get_logs`        | Logs          | Direct  |
| `traceforward_list_services`   | Services      | Direct  |
| `traceforward_get_service_map` | Traces        | Derived |
| `traceforward_get_errors`      | Traces        | Derived |

Direct tools map to `SignalAdapter` capabilities. Derived tools produce higher-order views from normalized trace data, including service topology and recent failure patterns.

All tools are prefixed with `traceforward_` to provide a clear namespace when multiple MCP servers are active in the same agent environment.

Derived tools do not expand the `SignalAdapter` contract defined in ADR-0001. Service map construction and error extraction are performed above the adapter layer from trace data returned by `query_traces()`.

## Consequences

* **Small, learnable surface.** Six tools are easier for agents to discover, select, and use reliably than a broader interface.
* **Higher-order value.** Derived tools answer common triage questions such as service relationships and recent failures without requiring agents to compute those views themselves from raw traces.
* **Stable adapter boundary.** New derived capabilities can be built above the adapter layer without expanding the core ingestion protocol.
* **Namespaced tool registry.** The `traceforward_` prefix reduces ambiguity and collisions in multi-server agent environments.
* **Trade-off.** A constrained tool surface limits how quickly new capabilities can be exposed as separate tools. TraceForward will prefer extending tool parameters and response models before adding new tool names.
* **Trade-off.** Derived tools depend on the quality and completeness of trace data. In backends with sparse or inconsistent trace coverage, service maps and error summaries may be incomplete.
