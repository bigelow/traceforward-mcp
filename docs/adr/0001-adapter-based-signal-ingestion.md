# 0001 — Adapter-Based Signal Ingestion

**Date:** 2026-03-24
**Status:** Accepted

## Context

TraceForward needs to ingest observability signals — traces, metrics, and logs — from diverse backend systems. Teams use different observability stacks (Grafana LGTM, Datadog, New Relic, Honeycomb, etc.), and no single API or query language covers them all.

Directly coupling TraceForward to any one backend would limit adoption and create maintenance burden as new backends emerge. The system needs a clean boundary between "what data do I need?" and "how do I get it from this specific backend?"

## Decision

We adopt the **Adapter pattern** via a `SignalAdapter` protocol. Each adapter implements four methods:

- `list_services()` — enumerate known services
- `query_logs()` — retrieve log entries
- `query_metrics()` — retrieve metric data
- `query_traces()` — retrieve trace spans

All adapters return normalized data structures (Pydantic v2 models), decoupling the MCP tool layer from backend-specific response formats.

New backends are supported by implementing a new adapter — no changes to the tool layer or core logic required.

## Consequences

- **Extensibility.** Adding a new observability backend is a single module implementing four methods against a known protocol.
- **Testability.** Adapters can be tested in isolation; the tool layer can be tested against the fixture adapter (ADR-0016).
- **Contributor-friendly.** The adapter interface is the primary extension point for the community.
- **Trade-off.** The four-method protocol may not perfectly fit every backend's capabilities. Some adapters may need to synthesize responses (e.g., deriving service lists from trace data rather than a native service catalog). This is acceptable — adapters own that translation.
- **Trade-off.** The protocol surface must remain stable once published, as adapters depend on it. Changes require careful versioning.
