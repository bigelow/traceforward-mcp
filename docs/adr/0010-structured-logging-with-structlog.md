# 0010 — Structured Logging with structlog

**Date:** 2026-03-24
**Status:** Accepted

## Context

TraceForward is an observability tool — its own operational observability should be exemplary. Unstructured log output (plain text, print statements, inconsistent formats) makes it difficult to diagnose issues, correlate events, and monitor the server in deployment.

Alternatives considered:

- **stdlib logging** — available everywhere, but produces unstructured text by default. Structured output requires significant configuration and custom formatters.
- **loguru** — developer-friendly with nice defaults, but less flexible for structured/JSON output and less common in the observability ecosystem.
- **structlog** — purpose-built for structured, key-value logging in Python. Produces JSON output by default, integrates with stdlib logging, and aligns with the structured data philosophy of the observability space.

## Decision

We use **structlog** for all logging in TraceForward.

Conventions:

- **JSON output in deployment.** All log entries are emitted as structured JSON, queryable by any log aggregation tool.
- **Human-readable output in development.** structlog's console renderer is used for local development, providing colored, readable output without sacrificing structure.
- **Bound loggers.** Loggers are bound with contextual key-value pairs (adapter name, service being queried, tool being called) at creation time, so every log entry carries relevant context automatically.
- **No print statements.** All output goes through structlog. Print statements are a lint failure.

## Consequences

- **Dogfooding.** An observability tool that produces well-structured, queryable logs demonstrates credibility and makes the project easier to operate.
- **Correlation.** Bound context (adapter, service, tool) on every log entry makes it trivial to trace a request through the system.
- **Contributor clarity.** "Use structlog, no print statements" is a simple, enforceable rule.
- **Trade-off.** structlog is an additional dependency. It is well-maintained, has no transitive dependencies of concern, and is widely used in the Python observability ecosystem.
- **Trade-off.** Structured JSON logs are harder to read in raw form than plain text. The development console renderer mitigates this for local work.
