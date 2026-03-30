# 0003 — Tool Surface Design

**Date:** 2026-03-24
**Status:** Accepted

## Context

MCP tools are the interface that coding agents interact with. The tool surface must balance completeness (agents need enough data to make informed decisions) with simplicity (agents perform best with a small, well-named set of tools that do clear things).

TraceForward ingests three signal types — traces, metrics, and logs — plus needs to expose higher-order views like service topology and error patterns.

## Decision

TraceForward exposes **six MCP tools**:

| Tool | Signal Source | Type |
|---|---|---|
| `traceforward_get_traces` | Traces | Direct |
| `traceforward_get_metrics` | Metrics | Direct |
| `traceforward_get_logs` | Logs | Direct |
| `traceforward_list_services` | Services | Direct |
| `traceforward_get_service_map` | Traces | Derived |
| `traceforward_get_errors` | Traces | Derived |

**Direct tools** map 1:1 to `SignalAdapter` methods. **Derived tools** synthesize higher-order views from trace data — service maps show topology and dependency relationships; error summaries surface failure patterns.

All tools are prefixed with `traceforward_` to namespace them clearly in an agent's tool registry, where multiple MCP servers may be active.

## Consequences

- **Minimal surface.** Six tools are learnable. Agents can reason about which tool to call without extensive prompt engineering.
- **Derived tools add value.** Service maps and error summaries answer the questions agents most often need ("what talks to what?" and "what's failing?") without requiring agents to query raw traces and compute the answers themselves.
- **Namespacing.** The `traceforward_` prefix prevents collisions but adds verbosity. This is an acceptable trade-off for clarity in multi-tool environments.
- **Trade-off.** Fixing the tool surface at six tools means new capabilities require either expanding an existing tool's parameters or adding new tools. We prefer evolving parameters over proliferating tools, to keep the surface small.
