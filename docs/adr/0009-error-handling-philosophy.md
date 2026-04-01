# 0009 — Error Handling Philosophy

**Date:** 2026-03-24
**Status:** Accepted

## Context

TraceForward sits between coding agents and observability backends. Every error includes a category (e.g., connection_error, query_error, not_found, validation_error, authentication_error, authorization_error) and a human-readable message. The full error taxonomy is defined across this ADR and ADR-0008 (Security and Authentication Model).

How TraceForward communicates errors matters more than in a typical API, because the consumer is often an LLM agent that needs to reason about what went wrong and decide what to do next. Opaque error codes or stack traces are useless to an agent. Vague messages like "internal error" prevent recovery.

## Decision

TraceForward follows these error handling principles:

1. **Errors are structured.** All errors returned through MCP tools use consistent, structured formats — not raw exception strings. Every error includes a category (e.g., `connection_error`, `query_error`, `not_found`, `validation_error`) and a human-readable message.

2. **Errors are actionable.** Error messages describe what happened and, where possible, what the caller can do about it. Example: "Service 'payment-api' not found. Available services: checkout, inventory, gateway" — not "Service not found."

3. **Partial results over total failure.** If a query partially succeeds (e.g., logs retrieved but metrics backend is unreachable), return the partial results with a clear indication of what failed. Agents can work with incomplete data; they cannot work with nothing.

4. **Adapter errors are normalized.** Backend-specific errors (HTTP 429, gRPC UNAVAILABLE, etc.) are translated into TraceForward's error categories by the adapter. The tool layer never leaks backend-specific error formats.

5. **Fail fast on configuration.** Missing or invalid adapter configuration (bad endpoint, missing API key) fails at startup with a clear message — not at first query time.

## Consequences

- **Agent-friendly.** Structured, actionable errors give agents enough information to retry, try a different tool, or report the issue to the user.
- **Debuggable.** Consistent error categories make it straightforward to diagnose issues in logs and monitoring.
- **Partial results preserve utility.** An agent investigating a service still gets value from logs even if the metrics backend is temporarily down.
- **Trade-off.** Normalizing errors across backends requires each adapter to map backend-specific failures to TraceForward categories. This is additional adapter work, but keeps the tool layer clean.
- **Trade-off.** Actionable error messages may inadvertently leak information about available services or backend topology. This intersects with the security model (ADR-0008) and may need revisiting.
