# 0016 — Fixture Adapter as Reference Implementation

**Date:** 2026-04-01
**Status:** Accepted

## Context

TraceForward’s adapter model, defined in ADR-0001, establishes a `SignalAdapter` protocol with four methods: `list_services`, `query_logs`, `query_metrics`, and `query_traces`. Contributors implementing new backends need more than signatures alone. They need a working example that shows expected model shapes, data relationships, and how adapter outputs support both direct and derived tools.

The fixture adapter already provides deterministic sample data and a complete implementation of the adapter interface. Its architectural role needs to be explicit so it does not drift into a convenience-only development artifact.

## Decision

The fixture adapter in `adapters/fixture.py` is the reference implementation of the `SignalAdapter` protocol.

It serves three roles:

### 1. Specification by example

The fixture adapter demonstrates the expected behavior and normalized output shape of a conforming adapter. Contributors use it to understand how canonical models, signal data, and derived-tool inputs should look in practice.

The formal contract remains the `SignalAdapter` protocol and the project’s typed models. The fixture adapter is the concrete example of that contract.

### 2. Development and testing harness

Fixture mode is the default zero-dependency path for local development, CI, and agent evaluation.

It is used for:

* local development of tool logic without a running backend
* CI execution without network or backend dependencies
* deterministic agent evaluation against known data

### 3. Contributor onboarding

The fixture adapter is the first complete adapter implementation contributors should read. It teaches the adapter pattern through a working example rather than abstract interface documentation.

## Fixture data requirements

Fixture data must be:

* **Realistic** — service names, operation names, errors, and metrics should resemble a plausible microservices environment
* **Complete** — every tool should return meaningful results against fixture data
* **Deterministic** — no randomness or runtime-derived timestamps
* **Minimal but sufficient** — enough data to exercise all tools and key edge cases without becoming unwieldy

## Fixture data as test oracle

Tests may assert against known fixture outputs. Changes to fixture data that affect expected results must be reviewed and updated deliberately.

This coupling is intentional. It makes fixture changes visible and prevents accidental drift in the reference implementation.

## What the fixture adapter is not

* **Not a mock** — it implements the full adapter protocol rather than a shortcut testing surface
* **Not a simulation of all backend behavior** — it does not attempt to model latency, authentication, rate limiting, or backend-specific failure modes

## Consequences

* **Concrete contract example.** Contributors learn the adapter contract from a complete working implementation.
* **Fast feedback loop.** Local development and CI do not depend on an external observability backend.
* **Protocol-first evolution.** When the adapter contract changes, the fixture adapter must be updated immediately, forcing the change to be expressed concretely.
* **Trade-off.** Static fixture data cannot represent every real-world backend behavior or failure mode.
* **Trade-off.** Tests become intentionally coupled to fixture values, which increases maintenance when fixture data changes.
* **Trade-off.** Real backend integration testing is still required for production confidence, even though fixture mode is the default development path.
