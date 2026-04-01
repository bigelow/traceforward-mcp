# 0016 — Fixture Adapter as Reference Implementation

**Date:** 2026-04-01
**Status:** Accepted

## Context

TraceForward's adapter pattern (ADR-0001) defines a `SignalAdapter` protocol with four methods: `list_services`, `query_logs`, `query_metrics`, and `query_traces`. Any new backend integration implements this protocol.

Before a contributor can write a new adapter, they need a working example that demonstrates the full contract — not just the method signatures, but the shape of the data, the behavior of derived tools (`get_service_map`, `get_errors`), and the interaction between the adapter layer and the tool layer.

The fixture adapter already exists and ships realistic sample data. But its role has not been formally defined. Is it a demo? A test harness? A development tool? A specification? Without clarity, the fixture adapter risks drifting into a dumping ground for convenience data that no longer represents real adapter behavior.

## Decision

The fixture adapter (`adapters/fixture.py`) is the **reference implementation** of the `SignalAdapter` protocol. It serves three purposes:

### 1. Specification by example

The fixture adapter's return values define the canonical shape of adapter output. Every field, every type, every relationship between entities in the fixture data is a statement about what a real adapter must produce.

If a contributor is unsure whether `query_traces` should include error spans, they check the fixture adapter. If the fixture includes them, the contract expects them.

### 2. Development and testing harness

Fixture mode (`mise run dev`) starts TraceForward with no external dependencies. This is the default path for:

- Local development — iterate on tool logic without a running backend.
- CI — all tests run against the fixture adapter. No network, no containers, no flaky backend connections.
- Agent evaluation — test how a coding agent interacts with TraceForward tools using deterministic, reproducible data.

### 3. Contributor onboarding

The fixture adapter is the first file a new contributor reads after the README. Its structure teaches the adapter pattern by showing a complete, working implementation rather than describing an abstract interface.

### Fixture data requirements

The fixture data set shall be:

- **Realistic.** Service names, operation names, error messages, and metric values should resemble what a real microservices environment produces. The sample session in the README is drawn from fixture data.
- **Complete.** Every tool must return meaningful results against the fixture data. If `traceforward_get_service_map` returns an empty graph, the fixture data is incomplete.
- **Deterministic.** No randomness, no timestamps derived from `now()`. Tests depend on fixture data being identical across runs.
- **Minimal but sufficient.** Enough services, traces, and relationships to exercise all tools and edge cases (multi-hop dependencies, error spans, log levels). Not so much that the fixture file becomes hard to read or maintain.

### Fixture data as test oracle

Tests verify tool behavior against fixture data with known expected outputs. When fixture data changes, tests that depend on specific values must be updated. This is intentional — it forces fixture changes to be deliberate and reviewed, not accidental.

### What the fixture adapter is not

- Not a mock. Mocks simulate behavior with shortcuts. The fixture adapter implements the full `SignalAdapter` protocol with the same code paths a real adapter would use.
- Not a demo backend. It does not simulate latency, authentication, or failure modes. Those concerns belong to integration tests against real backends.

## Consequences

- **Single source of truth.** Contributors learn the adapter contract from a working implementation, not from documentation that may drift from reality.
- **Zero-dependency testing.** CI and local development never require an external observability backend, keeping the feedback loop fast.
- **Fixture maintenance cost.** When the `SignalAdapter` protocol evolves, the fixture adapter must be updated first. This is a feature — it forces the protocol change to be expressed concretely before any real adapter is modified.
- **Trade-off.** Fixture data is static and cannot represent all real-world edge cases (partial failures, rate limiting, schema variations across backends). Integration tests against real backends are still needed for production confidence, but are not gated in CI.
- **Trade-off.** Treating fixture data as a test oracle creates coupling between test assertions and fixture values. This coupling is accepted — it is the mechanism by which fixture changes are made visible and intentional.
