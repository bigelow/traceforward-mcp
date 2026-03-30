# 0011 — Redaction, Disclosure, and Data Handling Policy

**Date:** 2026-03-25
**Status:** Accepted

## Context

TraceForward queries observability backends that contain sensitive operational data. Traces may include HTTP headers with API keys or session tokens. Logs may contain PII — usernames, email addresses, IP addresses, request bodies with personal data. Metrics labels may reveal internal service names, deployment topology, or infrastructure details.

ADR-0008 establishes the authentication model — who can access TraceForward and how adapters authenticate with backends. But authentication answers *access*, not *exposure*. A caller who is authenticated to query a backend may still receive data that should be filtered, masked, or withheld before it reaches an agent's context window.

ADR-0009 establishes that errors should be actionable — but actionable errors can leak topology and service existence. Listing available services in a "not found" error is helpful for agents but may be an information disclosure concern in shared or remote deployments.

These two ADRs create a tension that needs an explicit policy: what data is allowed to leave TraceForward, and what must be filtered.

## Decision

TraceForward adopts a **layered redaction model** with policy enforcement in the core and filtering execution split between adapters and a common redaction layer.

### Redaction principles

1. **Credentials never pass through.** API keys, tokens, passwords, and secrets detected in trace headers, log messages, or metric labels are redacted before results leave TraceForward. This is non-negotiable regardless of deployment phase or caller trust level.

2. **PII redaction is configurable.** TraceForward provides a default set of PII patterns (email addresses, IP addresses, common personal data formats) that are redacted by default. Operators can extend, reduce, or disable PII redaction through configuration. The default is redact-on.

3. **Topology disclosure follows deployment phase.** In local deployments (ADR-0008 Phase 1 and 2), service names, dependency maps, and error details are disclosed freely — the caller is the operator. In remote/shared deployments (Phase 3), topology disclosure is governed by per-caller access scoping defined in ADR-0008.

4. **Error messages respect disclosure policy.** Actionable errors (ADR-0009) are still actionable, but the detail level adapts to context. In local deployments, "Service 'payment-api' not found. Available services: checkout, inventory, gateway" is appropriate. In remote deployments, error messages may omit the available services list unless the caller has access to that information.

5. **Adapters may pre-filter.** Some backends support server-side filtering or redaction (e.g., Grafana Loki's label-based access control). Adapters that connect to backends with native filtering should use it — reducing the volume of sensitive data that ever reaches TraceForward.

### Where filtering executes

- **Adapter-level pre-filtering.** Adapters use backend-native access controls and query scoping to limit what data is retrieved. This is the first line of defense.
- **Core redaction layer.** A common redaction step runs after adapter responses are returned and before results are delivered through MCP tools. This layer applies credential stripping, PII pattern matching, and topology disclosure rules. It operates on the normalized Pydantic models that adapters return, not on raw backend responses.
- **Adapters do not implement redaction policy.** Adapters may pre-filter using backend capabilities, but the redaction rules (what patterns to strip, what to disclose) are owned and enforced by the core. This prevents inconsistency across adapters.

### Configuration

Redaction configuration is centralized, not per-adapter:

- **`redaction.credentials`** — always enabled, not configurable. Credentials are always stripped.
- **`redaction.pii`** — enabled by default. Operators can configure pattern lists or disable for local-only deployments where PII handling is the operator's responsibility.
- **`redaction.topology`** — controls whether service names, dependency details, and infrastructure identifiers are included in responses and error messages. Default: disclose in local deployments, restrict in remote deployments.

### Policy semantics and enforcement guarantees

#### Field classification

Redaction rules apply differently to structured fields and free text:

- **Structured fields** (Pydantic model attributes — service names, metric labels, trace span attributes, header key-value pairs) are matched by field name and value against known credential and PII patterns. Structured fields are the high-confidence path — patterns match against typed, bounded values.
- **Free text** (log message bodies, error description strings, span event text) is scanned by regex pattern matching. Free text is the low-confidence path — higher false-positive and false-negative rates are expected and accepted.

The distinction matters because a field named `authorization` in a trace header is a structured match (high confidence), while the string "Bearer eyJ..." appearing inside a log message body is a free-text match (pattern-dependent).

#### What is always redacted (non-configurable)

Regardless of deployment phase or operator configuration:

- HTTP `Authorization` header values
- Values matching `Bearer`, `Basic`, API key, and token patterns in any field
- Environment variable values embedded in log messages that match known secret naming conventions (`*_SECRET`, `*_KEY`, `*_TOKEN`, `*_PASSWORD`)
- Connection strings containing embedded credentials

These are the **credential floor** — the minimum guarantee the platform makes. An operator cannot weaken this. If a credential pattern is detected, the value is replaced with `[REDACTED]`.

#### What is conditionally redacted (configurable)

When `redaction.pii` is enabled (default: on):

- Email address patterns
- IPv4 and IPv6 addresses
- Common personal identifier formats (SSN patterns, phone number patterns)
- Operator-defined custom patterns

When `redaction.topology` is restricted (default in Phase 3):

- Service names in error messages are replaced with `[service]`
- Dependency lists in "not found" errors are omitted
- Infrastructure identifiers (hostnames, pod names, namespace names) are stripped from responses

#### Conflict resolution: adapter pre-filtering vs core policy

When a backend's native filtering and TraceForward's core redaction overlap or disagree:

- **Core policy always wins.** If core policy says a field must be redacted, it is redacted even if the adapter's backend already considers the data "safe."
- **Adapter pre-filtering is additive, not authoritative.** Backend-native filtering reduces what reaches the core, but the core does not trust that filtering is complete. The core redaction layer runs unconditionally.
- **Adapters must not redact on their own initiative.** An adapter that strips data the core doesn't know about creates invisible data loss. If an adapter's backend requires mandatory filtering (e.g., Loki tenant isolation), the adapter documents what is pre-filtered so operators understand the full pipeline.

#### Redaction markers

When a value is redacted, the response includes a marker:

- Redacted values are replaced with `[REDACTED]`, not silently dropped. This preserves the structure of the response (an agent can see that a header existed but its value was removed, rather than the header appearing to not exist).
- When `redaction.topology` omits a list (e.g., available services in an error), the error message includes `[topology restricted]` so the agent knows information was withheld, not that zero services exist.

#### Audit and testing

- **Redaction is tested per release.** The test suite includes fixtures with known credential patterns, PII patterns, and topology identifiers. Tests verify that all credential-floor patterns are redacted regardless of configuration.
- **Redaction failures are logged.** If the redaction layer encounters a field it cannot classify (unrecognized structure, ambiguous content), it logs a warning via structlog (ADR-0010) with the field name and context — never the field value.
- **No compliance guarantee.** TraceForward's redaction is a best-effort engineering control, not a regulatory compliance boundary. Operators with compliance requirements (GDPR, HIPAA, SOC 2) must validate that TraceForward's redaction meets their specific obligations. The platform provides the mechanism; the operator owns the policy adequacy.

## Consequences

- **Defense in depth.** Adapter pre-filtering reduces sensitive data retrieval; core redaction catches anything that gets through.
- **Consistent policy.** Redaction rules live in one place, applied uniformly regardless of which adapter produced the data.
- **Credential safety.** The hardcoded credential stripping rule means TraceForward never becomes a vector for leaking secrets from observability data, even if an operator misconfigures other settings.
- **Resolves ADR-0009 tension.** Error disclosure adapts to deployment context rather than being unconditionally verbose or unconditionally opaque.
- **Trade-off.** The core redaction layer adds processing to every response. For local deployments with PII redaction disabled, this cost is minimal (credential stripping only). For deployments with full PII pattern matching, there is a measurable overhead on large result sets.
- **Trade-off.** Pattern-based PII detection is imperfect. It will miss novel formats and may false-positive on benign data. This is an inherent limitation of regex-based redaction and is acceptable as a first layer — not a compliance guarantee.
- **Trade-off.** Topology disclosure rules that differ by deployment phase mean the same TraceForward instance may behave differently depending on configuration. This is intentional — local and remote deployments have genuinely different threat models — but operators must understand the distinction.
