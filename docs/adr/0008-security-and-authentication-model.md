# 0008 — Security and Authentication Model

**Date:** 2026-03-25
**Status:** Accepted

## Context

TraceForward connects to observability backends that contain sensitive operational data — traces, logs, and metrics that may reveal internal service architecture, error details, API keys in headers, PII in log messages, and other information that should not be exposed without proper authorization.

The security model must address two boundaries:

1. **TraceForward ↔ Observability backend.** How does TraceForward authenticate with Grafana, Datadog, or other backends to query signal data?
2. **Agent ↔ TraceForward.** How does the calling agent or user authenticate with the TraceForward MCP server?

A purely adapter-owned authentication model fits the adapter architecture, because each backend has different requirements and `SignalAdapter` is intended to keep backend-specific behavior at the edge. But pure adapter ownership is too loose. If every adapter handles security differently, the system becomes inconsistent as soon as TraceForward moves beyond local `stdio` into shared or remote transports.

TraceForward should not become a credential broker.

## Decision

TraceForward adopts a hybrid security model: adapters authenticate to observability backends, while the core governs authentication policy and, in networked deployments, enforces client authentication and authorization.

### Explicit non-goal

The core does not persist, escrow, or centrally mint backend credentials. TraceForward is not a credential broker. Adapters receive credentials through environment variables or configuration files at startup, authenticate with their backends directly, and manage their own backend-specific credential lifecycle.

### What the core governs

* **Authentication policy.** The core defines the valid configuration shape, supported authentication modes, and required auth-related settings for each adapter.
* **Startup validation.** Before any adapter accepts queries, the core verifies that required credential material is present and non-empty. Invalid or missing configuration fails fast with a clear error message.
* **Disclosure rules.** The core defines what authentication-related information may appear in logs and error messages. Failures must not leak secrets, backend endpoints, or internal topology.
* **Client security in networked deployments.** When TraceForward is exposed over a network-accessible transport, the core is responsible for enforcing client authentication and authorization.

### What adapters own

* **Backend authentication execution.** Each adapter authenticates with its backend using the mechanism that backend requires, such as API keys, bearer tokens, service account credentials, or mTLS.
* **Backend-specific auth flows.** If a backend requires token exchange, session renewal, or multi-step authentication, the adapter owns that logic.
* **Failure mapping.** Adapters map backend-specific failures into TraceForward’s normalized error categories.

### Error taxonomy for auth failures

Backend auth failures map to two distinct TraceForward error categories:

* **`authentication_error`** — identity could not be established. This maps from HTTP 401, gRPC `UNAUTHENTICATED`, or equivalent conditions. The actionable meaning is that credentials are missing, expired, or invalid.
* **`authorization_error`** — identity was established, but access to the requested resource or operation is not permitted. This maps from HTTP 403, gRPC `PERMISSION_DENIED`, or equivalent conditions.

These categories remain distinct because they imply different recovery paths.

### Deployment phases

Security requirements escalate with deployment scope.

**Phase 1 — Local stdio (MVP).**
TraceForward runs as a local MCP server on the user’s machine. The trust boundary is the local machine. No additional authentication or authorization is required between agent and TraceForward. Adapter authentication to backends is the only active auth layer.

**Phase 2 — Local SSE.**
TraceForward runs as an SSE server bound to `127.0.0.1` only. The trust boundary remains the local machine, but opening a listening socket increases exposure to other local processes or users on the same host. This mode is acceptable for single-user development machines, but not for shared hosts without additional local access controls.

**Phase 3 — Remote SSE.**
TraceForward runs as a network-accessible SSE server. This phase requires:

* client authentication
* client authorization
* transport encryption
* rate limiting

Detailed Phase 3 design is deferred until Phase 1 is stable, but the hybrid model establishes the boundary that Phase 3 will build on: backend authentication remains adapter-owned, while remote client security is core-owned.

## Consequences

* **Clean boundaries.** Adapters remain responsible for backend-specific authentication, while the core defines and enforces consistent security policy.
* **Fail-fast safety.** Startup validation catches auth misconfiguration before sensitive data flows.
* **Deployment-aware security.** Security complexity increases only as deployment scope increases.
* **Actionable errors.** Distinguishing `authentication_error` from `authorization_error` gives agents and users clearer recovery paths.
* **Explicit non-goal.** Stating that the core never persists or mints backend credentials prevents scope creep toward credential management.
* **Trade-off.** The hybrid model expands the adapter contract beyond pure signal retrieval by requiring conformance to shared auth policy and normalized failure mapping.
* **Trade-off.** Remote multi-user deployments are deferred until later phases, which limits early deployment flexibility.
* **Trade-off.** Localhost-only SSE is acceptable for single-user development but not inherently safe on shared systems.
* **Trade-off.** This ADR defines authentication and authorization boundaries, but not the broader redaction and normal-response disclosure policy. That remains the responsibility of ADR-0011.
