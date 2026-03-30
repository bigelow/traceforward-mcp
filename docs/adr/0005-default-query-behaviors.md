# 0005 — Default Query Behaviors

**Date:** 2026-03-24
**Status:** Accepted

## Context

Every TraceForward tool that queries signal data needs sensible defaults for time range and result limits. Coding agents often call tools without specifying these parameters — they want "recent, relevant data" without needing to reason about exact time windows or pagination.

Poor defaults create problems in both directions: too narrow and the agent misses relevant signals; too broad and responses become noisy, slow, or hit backend limits.

## Decision

All query tools default to:

- **`lookback_hours=24`** — queries cover the last 24 hours unless the caller specifies otherwise.
- **`limit=50`** — queries return at most 50 results unless the caller specifies otherwise.

These defaults are defined centrally and applied consistently across all tools and adapters.

### Rationale

- **24 hours** captures a full daily cycle — deployments, traffic patterns, batch jobs — without pulling in stale historical data. Most coding agents are working on recent changes and need recent signals.
- **50 results** is large enough to surface meaningful patterns (error clusters, latency outliers) but small enough to fit comfortably within an LLM's context window without overwhelming the agent's reasoning.

## Consequences

- **Zero-config usability.** Agents can call any tool with just a service name and get useful results immediately.
- **Predictable performance.** Backends receive bounded queries, reducing the risk of expensive full-table scans or unbounded result sets.
- **Trade-off.** 24 hours may miss intermittent issues with longer cycles (weekly batch jobs, weekend traffic patterns). Agents or users can override when needed.
- **Trade-off.** 50 results may truncate large error volumes. The limit prevents context window overflow, which is the higher-priority concern for agent usability.
