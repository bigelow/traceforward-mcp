# 0008 — Security and Authentication Model

**Date:** 2026-03-25
**Status:** Accepted

## Context

TraceForward connects to observability backends that contain sensitive operational data — traces, logs, and metrics that may reveal internal service architecture, error details, API keys in headers, PII in log messages, and other information that should not be exposed without proper authorization.

The authentication model must address two boundaries:

1. **TraceForward ↔ Observability backend.** How does TraceForward authenticate with Grafana, Datadog, or other backends to query signal data?
2. **Agent ↔ TraceForward.** How does the calling agent (or user) authenticate with the TraceForward MCP server?

A purely adapter-owned auth model fits the adapter architecture — each backend is different, and the whole point of `SignalAdapter` is to keep backend-specific behavior at the edge. But pure adapter ownership is too loose. Every adapter solving auth differently creates inconsistency, and that gets ugly the moment you move beyond local stdio into shared or remote transport.

TraceForward should not become a credential broker.

## Decision

We adopt a **hybrid authentication model**: adapters execute authentication, core governs authentication policy. Authorization — controlling what an authenticated caller may access — is a separate concern addressed per deployment phase.

### Explicit non-goal

The core does not persist, escrow, or centrally mint backend credentials. TraceForward is not a credential broker. Adapters receive credentials through environment variables or configuration files at startup, authenticate with their backends directly, and manage their own credential lifecycle. The core validates and governs — it never holds or issues secrets.

### What the core governs

- **Required auth configuration shape.** The core defines what a valid adapter auth configuration looks like — required fields, expected types, naming conventions.
- **Secret presence validation at startup.** Before any adapter accepts queries, the core verifies that required credentials are present and non-empty. Missing or invalid configuration fails fast with a clear message (consistent with ADR-0009).
- **Allowed auth modes per adapter.** The core maintains a registry of which authentication mechanisms each adapter supports (API key, bearer token, mTLS, etc.), preventing misconfiguration.
- **Rotation and refresh hooks.** The core defines a standard interface for credential rotation and refresh. The minimum contract: adapters that support renewable credentials implement a `refresh_credentials()` method that returns a boolean success indicator and updates the adapter's internal credential state. The core calls this method on a configurable interval (default: not called — adapters opt in by declaring `supports_refresh = True` in their configuration). If refresh fails, the core logs a warning and the adapter continues with existing credentials until the next refresh attempt or until a backend request fails with `authentication_error`. The core does not retry refresh on behalf of the adapter — retry logic is adapter-owned.
- **Disclosure rules.** The core enforces what auth-related information may appear in error messages and logs. Auth failures are normalized into TraceForward error categories (consistent with ADR-0009) without leaking credentials, backend URLs, or internal topology. Detailed redaction and data exposure policy is covered in ADR-0011.

### Error taxonomy for auth failures

Backend auth failures map to two distinct TraceForward error categories:

- **`authentication_error`** — the caller or adapter could not prove identity. Maps from HTTP 401, gRPC `UNAUTHENTICATED`, or equivalent. Actionable message: credentials are missing, expired, or invalid.
- **`authorization_error`** — identity was established but the caller lacks permission for the requested operation. Maps from HTTP 403, gRPC `PERMISSION_DENIED`, or equivalent. Actionable message: the caller is authenticated but does not have access to the requested resource.

Collapsing these into a single category would violate ADR-0009's principle that errors must be actionable — an agent needs to know whether to refresh credentials or request different permissions.

### What adapters own

- **Backend authentication execution.** Each adapter authenticates with its backend using whatever mechanism that backend requires — API key headers, OAuth token exchange, service account credentials, mTLS certificates.
- **Token and session refresh mechanics.** When a backend requires token refresh, the adapter implements `refresh_credentials()` using the core's rotation hook interface.
- **Backend-specific auth flows.** Some backends require multi-step authentication (OAuth code flow, SAML, etc.). The adapter owns the full flow.
- **Mapping backend auth failures to normalized errors.** A 401 from Grafana becomes `authentication_error`. A 403 from Datadog becomes `authorization_error`. An `UNAUTHENTICATED` from a gRPC backend becomes `authentication_error`. Adapters are responsible for mapping backend-specific failure codes to the correct TraceForward error category.

### Deployment phases

Security requirements escalate with deployment scope:

**Phase 1 — Local stdio (MVP).** TraceForward runs as a local MCP server on the user's machine. The trust boundary is the local machine — if you can run the server, you have access. No additional authentication or authorization between agent and TraceForward. Adapter authentication to backends is the only auth layer.

**Phase 2 — Local SSE.** TraceForward runs as an SSE server on the local machine, accessible to local agents and tools. Bind to `127.0.0.1` only — not `0.0.0.0`. No remote access without explicit configuration. The trust boundary is still the local machine, but SSE opens a listening socket, which means other local processes and users on the same machine can connect. This is acceptable for single-user development machines but is not safe on shared hosts. On multi-user systems, Phase 2 requires either OS-level access control (firewall rules, socket permissions) or upgrading to Phase 3 authentication. Documentation must state this assumption explicitly.

**Phase 3 — Remote SSE.** TraceForward runs as a network-accessible SSE server. This phase requires both authentication and authorization:

- **Authentication** between agent/user and TraceForward (mechanism TBD — likely bearer token or API key) to establish caller identity.
- **Authorization** to scope what an authenticated caller may access — which adapters, which services, which signal types. This is authz, not authn.
- **Transport encryption** (TLS).
- **Rate limiting** to prevent abuse.

Phase 3 design is deferred until Phase 1 is stable. The hybrid model is designed to support it — the core's governance of auth modes, disclosure rules, and configuration validation provides the foundation that Phase 3 builds on.

## Consequences

- **Clean boundaries.** Adapters stay focused on backend-specific concerns. The core prevents inconsistency without becoming a credential broker.
- **Fail-fast safety.** Startup validation catches misconfigured auth before any sensitive data flows.
- **Phased deployment.** Each phase adds security surface only when needed, avoiding premature complexity.
- **Precise error taxonomy.** Distinguishing `authentication_error` from `authorization_error` gives agents actionable recovery paths — refresh credentials vs. request different permissions (consistent with ADR-0009).
- **Explicit non-goal.** Stating that the core never persists or mints credentials prevents scope creep toward a credential management system.
- **Trade-off.** The hybrid model requires adapters to conform to core-defined auth configuration shapes, error category mappings, and optional rotation hooks. This is additional adapter contract surface beyond the four `SignalAdapter` methods.
- **Trade-off.** Phase 3 is deferred, which means TraceForward cannot serve shared or remote environments until that work is done. This is a deliberate scope choice — local-first, extend later.
- **Trade-off.** Phase 2's localhost binding assumption is safe for single-user machines but not for shared hosts. This is documented explicitly rather than silently assumed.
- **Trade-off.** Disclosure rules in this ADR establish that auth errors must not leak sensitive details, but the broader question of what data may be exposed in normal (non-error) responses is out of scope here. That decision belongs in ADR-0011 (Redaction, Disclosure, and Data Handling Policy).
