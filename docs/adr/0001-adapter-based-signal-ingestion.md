# 0001 — Adapter-Based Signal Ingestion

**Date:** 2026-03-24
**Status:** Accepted

## Context

TraceForward needs to ingest observability signals — traces, metrics, and logs — from diverse backend systems. Teams use different observability stacks such as Grafana LGTM, Datadog, New Relic, and Honeycomb, and no single API or query language covers them all.

Directly coupling TraceForward to any one backend would limit adoption and increase maintenance burden as new backends emerge. The system needs a clean boundary between the signal data TraceForward requires and the backend-specific mechanisms used to retrieve it.

## Decision

TraceForward will ingest traces, metrics, and logs exclusively through a `SignalAdapter` protocol. Each adapter implements four methods:

* `list_services()` — enumerate known services
* `query_logs()` — retrieve log entries
* `query_metrics()` — retrieve metric data
* `query_traces()` — retrieve trace spans

All adapter methods return normalized Pydantic v2 models. Backend-specific schemas, query languages, and response formats are contained within the adapter boundary.

Supporting a new backend requires a new adapter implementation, not changes to MCP tools or core orchestration.

## Consequences

* **Extensibility.** Adding a new observability backend is a single module implementing four methods against a known protocol.
* **Testability.** Adapters can be tested in isolation, and the tool layer can be tested against the fixture adapter (ADR-0016).
* **Contributor-friendly.** The adapter interface is the primary extension point for contributors.
* **Canonical model ownership.** The adapter boundary establishes TraceForward’s normalized signal model as the source of truth for downstream tools.
* **Trade-off.** The four-method protocol will not map perfectly to every backend. Some adapters may need to synthesize responses, such as deriving service lists from trace data rather than from a native service catalog.
* **Trade-off.** The `SignalAdapter` protocol becomes a stability boundary. Once adopted by multiple adapters, changes to it require deliberate versioning and migration planning.

This version is stronger. It reads more like a real architectural constraint and less like a pattern description.
