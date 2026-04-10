# 0010 — Structured Logging with structlog

**Date:** 2026-03-24
**Status:** Accepted

## Context

TraceForward is an observability tool and should produce operational signals that are easy to query, correlate, and analyze. Unstructured log output makes it harder to diagnose failures, understand request flow, and operate the server in real environments.

Alternatives considered:

* **stdlib logging** — broadly available, but unstructured by default and cumbersome to standardize for JSON-first logging.
* **loguru** — developer-friendly, but less aligned with structured logging conventions commonly used in observability systems.
* **structlog** — purpose-built for structured, key-value logging in Python and well aligned with JSON-based operational logging.

## Decision

TraceForward will use **structlog** for all application logging.

The project will apply the following conventions:

* **Structured output by default.** Log entries are emitted as structured events that can be consumed by standard log aggregation systems.
* **JSON in deployed environments.** Non-development environments emit JSON logs.
* **Readable development output.** Local development uses a human-readable renderer without changing the underlying structured event model.
* **Bound context.** Loggers are expected to carry relevant context such as adapter name, tool name, and service identifier.
* **No ad hoc output.** User-visible or operational logging does not use `print` statements.

## Consequences

* **Operational clarity.** Logs are easier to query, filter, and correlate across environments.
* **Consistent event model.** Structured logging aligns with TraceForward’s broader model-first approach to signals and interfaces.
* **Better debugging context.** Bound fields such as adapter, tool, and service make request paths easier to reconstruct.
* **Simple contributor rule.** “Use structlog, do not use print” is easy to review and enforce.
* **Trade-off.** `structlog` adds a dependency and requires contributors to follow a more deliberate logging style than plain stdlib logging.
* **Trade-off.** JSON logs are less readable in raw form, so development output requires a separate renderer for local use.
