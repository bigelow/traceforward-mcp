# 0005 — Default Query Behaviors

**Date:** 2026-03-24
**Status:** Accepted

## Context

TraceForward tools that query signal data need consistent default behaviors for time range and result limits. Coding agents will often call tools without specifying these parameters explicitly. In those cases, the system should return recent, bounded, and useful data without requiring the caller to reason about precise time windows or pagination.

Defaults that are too narrow risk hiding relevant signals. Defaults that are too broad increase latency, introduce noise, and can exceed backend or context-window constraints.

## Decision

TraceForward will apply the following default query parameters across signal-querying tools:

* **`lookback_hours=24`**
* **`limit=50`**

These defaults are defined centrally and applied consistently across tools and adapters unless the caller provides explicit overrides.

## Consequences

* **Usable without tuning.** Agents can call query tools with minimal parameters and still receive useful recent signal data.
* **Consistent behavior.** Tools behave predictably across traces, metrics, and logs because default time and result bounds are shared.
* **Bounded backend load.** Default queries remain constrained, reducing the risk of expensive or unbounded backend requests.
* **Bounded agent context.** Response sizes stay small enough to support tool use within LLM context limits.
* **Trade-off.** A 24-hour lookback may miss issues that occur on longer intervals, such as weekly jobs or weekend traffic patterns.
* **Trade-off.** A limit of 50 may truncate high-volume failure patterns. This is acceptable because bounded, readable responses are prioritized over exhaustive output by default.
* **Trade-off.** Central defaults become part of the usability contract. Changes to them affect tool behavior across the system and should be made deliberately.
