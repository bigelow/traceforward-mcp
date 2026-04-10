# 0017 — Test Strategy and Coverage Policy

**Date:** 2026-04-01
**Status:** Accepted

## Context

TraceForward has three architectural layers: adapters, a signals layer that derives higher-order views from adapter output, and MCP tools that expose those results to agents. Each layer has different testing needs.

The fixture adapter provides deterministic data with no external dependencies. That makes it possible to test the stack end-to-end without relying on a running backend, but only if the test strategy is designed around that capability.

ADR-0015 establishes lightweight ADR traceability in tests. This ADR defines the project’s testing structure, mocking boundaries, and coverage expectations.

## Decision

TraceForward adopts a layered test strategy that mirrors the architecture and uses the fixture adapter as the default zero-dependency test path.

### Test framework

* **pytest**
* **pytest-asyncio**
* tests are functions, not unittest-style classes

### Test layers

#### Adapter tests

Adapter tests verify conformance to the `SignalAdapter` protocol.

* each adapter has its own test module
* tests verify method behavior, return types, and required data shape
* fixture adapter tests assert against known fixture values
* live backend adapter tests are marked `integration` and excluded from default CI runs

#### Signal layer tests

Signal tests verify derived behavior such as service map construction and error extraction.

* these tests run against fixture adapter output
* they verify transformation logic, not backend integration behavior

#### Tool tests

Tool tests verify MCP-facing behavior.

* parameter validation
* default behaviors
* response envelope structure
* correct use of signal and adapter layers

Tool tests run end-to-end through the fixture adapter rather than mocking internal layers.

### Convention tests

Convention tests verify project-wide invariants derived from ADRs, including:

* terminology rules
* governance boundary rules
* tool naming conventions
* default query parameters

These tests may use ADR markers to make the governing decision explicit.

### Drift visibility

The test suite may include lightweight checks for traceability drift, such as:

* missing ADR references in key modules
* stale ADR markers that reference nonexistent or superseded ADRs

These checks are intended to surface drift, not create heavy process gates.

### Mocking policy

* internal layers are not mocked in the default test path
* mocking is allowed only at true external boundaries, such as HTTP clients for live adapters or clock control for time-sensitive logic
* no separate mock adapter is introduced; the fixture adapter is the test adapter

### Coverage policy

* target **90% line coverage** across core packages
* 100% coverage is not required
* coverage is measured but not initially gated in CI
* visible coverage gaps are preferred over artificial tests that only exercise plumbing

### Test organization

Test modules mirror the components they verify: adapters, signals, tools, and conventions.

## Consequences

* **Full-stack confidence.** The default test path exercises adapters, signal derivation, and tool behavior together against deterministic data.
* **Fast feedback.** Most tests run without network or backend dependencies.
* **Architectural verification.** Convention tests turn selected ADR rules into executable checks.
* **Clear mocking boundary.** The project avoids brittle internal mocks by relying on the fixture adapter as the default integration surface.
* **Trade-off.** A bug in the fixture adapter can affect multiple test layers at once. This is acceptable because the fixture adapter is the reference implementation.
* **Trade-off.** A 90% target leaves some paths untested. This is preferable to bloated tests that overfit implementation details.
* **Trade-off.** Lightweight drift checks may be ignored if they do not fail builds. This is accepted to avoid turning traceability into bureaucracy.
