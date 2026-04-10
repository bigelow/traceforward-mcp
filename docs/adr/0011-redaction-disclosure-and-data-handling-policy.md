# 0011 — Redaction, Disclosure, and Data Handling Policy

**Date:** 2026-03-25
**Status:** Accepted

## Context

TraceForward queries observability backends that may contain sensitive operational data. Traces may include headers with API keys or session tokens. Logs may include PII such as usernames, email addresses, IP addresses, or request bodies with personal data. Metrics labels may reveal service names, deployment topology, or infrastructure identifiers.

ADR-0008 defines who can access TraceForward and how adapters authenticate with backends. That answers access, not exposure. An authenticated caller may still receive data that should be filtered, masked, or withheld before it enters an agent context window.

ADR-0009 requires errors to be actionable. But actionable errors can also disclose service existence, dependency structure, or other topology details. TraceForward therefore needs an explicit policy for what data may leave the system and what must be filtered first.

## Decision

TraceForward adopts a layered disclosure model: adapters limit retrieval where possible, and the core enforces redaction and disclosure policy before any data is returned through MCP tools.

### Redaction principles

1. **Credentials never pass through.** Secrets, tokens, passwords, API keys, and similar credential material are always redacted before results leave TraceForward.

2. **PII redaction is configurable.** TraceForward provides a default set of PII patterns, including email addresses, IP addresses, and common personal identifier formats. PII redaction is enabled by default and may be adjusted by operators.

3. **Topology disclosure depends on deployment scope.** In local deployments, service names, dependency maps, and detailed errors may be disclosed freely. In shared or remote deployments, topology disclosure is restricted according to deployment policy and caller scope.

4. **Error detail follows disclosure policy.** Errors remain actionable, but the amount of detail adapts to the deployment context. Local deployments may include available services or dependency details. Remote deployments may suppress those details unless the caller is authorized to see them.

5. **Adapters may pre-filter, but do not define policy.** Adapters should use backend-native filtering when available, but disclosure rules are defined and enforced by the core.

### Where filtering executes

* **Adapter-level pre-filtering.** Adapters use backend-native access controls and query scoping to reduce what data is retrieved.
* **Core redaction layer.** After adapter responses are normalized, the core applies credential stripping, PII redaction, and topology disclosure rules before returning data through MCP tools.
* **Core policy wins.** Backend-native filtering is additive, not authoritative. The core redaction layer runs regardless of what the backend or adapter already filtered.

### Configuration

Redaction policy is centralized:

* **`redaction.credentials`** — always enabled and not configurable
* **`redaction.pii`** — enabled by default and operator-configurable
* **`redaction.topology`** — governs whether service names, dependency details, and infrastructure identifiers are disclosed

### Policy semantics

#### Structured fields vs. free text

Redaction operates differently on structured and unstructured content:

* **Structured fields** such as headers, span attributes, labels, and typed model fields are matched by field name and value
* **Free text** such as log messages, event text, and error strings is scanned by pattern matching

Structured fields are the high-confidence path. Free text is inherently lower confidence and may produce false positives or false negatives.

#### Always-redacted data

Regardless of deployment phase or operator configuration, the following are always redacted:

* HTTP `Authorization` header values
* Bearer, Basic, API key, token, and password patterns
* Embedded environment-variable values matching common secret naming conventions
* Connection strings containing credentials

These form the credential floor. Operators cannot disable this behavior. Redacted values are replaced with `[REDACTED]`.

#### Conditionally redacted data

When `redaction.pii` is enabled:

* email address patterns
* IPv4 and IPv6 addresses
* common personal identifier patterns
* operator-defined custom patterns

When `redaction.topology` is restricted:

* service names in error messages may be replaced
* dependency lists may be omitted
* infrastructure identifiers such as hostnames, pod names, and namespaces may be stripped

#### Redaction markers

When values are withheld, TraceForward preserves that fact explicitly:

* redacted values are replaced with `[REDACTED]`
* withheld topology details are marked with `[topology restricted]`

This preserves response structure and makes it clear that information was filtered rather than absent.

### Audit and testing

* Redaction behavior is tested with fixtures containing known secret, PII, and topology patterns
* Credential-floor redaction is validated on every release
* Redaction classification failures are logged without including the sensitive value itself
* TraceForward redaction is a best-effort engineering control, not a regulatory compliance guarantee

## Consequences

* **Defense in depth.** Backend-native filtering reduces exposure early, and the core redaction layer enforces consistent policy before data leaves TraceForward.
* **Consistent disclosure policy.** Redaction rules are defined once and applied uniformly across adapters.
* **Credential safety floor.** TraceForward will not knowingly emit credential material, even if other policy settings are weakened or misconfigured.
* **Context-sensitive error disclosure.** The system can remain actionable without disclosing the same level of detail in every deployment mode.
* **Trade-off.** Redaction adds processing overhead, especially when scanning large result sets or free-text fields.
* **Trade-off.** Pattern-based PII detection is imperfect and should not be treated as a compliance guarantee.
* **Trade-off.** The same instance may disclose different levels of detail depending on deployment mode and configuration. This is intentional, but operators must understand it.
