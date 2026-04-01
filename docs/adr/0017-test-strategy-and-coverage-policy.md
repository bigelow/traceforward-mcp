# 0017 — Test Strategy and Coverage Policy

**Date:** 2026-04-01
**Status:** Accepted

## Context

TraceForward's architecture has three distinct layers: adapters (ADR-0001), a signals layer that derives data from adapter output (ADR-0003), and MCP tools that expose that data to agents (ADR-0002). Each layer has different testing needs.

The fixture adapter (ADR-0016) provides deterministic data with no external dependencies, making it possible to test the full stack — adapter through tool response — without a running backend. This is a significant advantage, but only if the test strategy is designed around it.

ADR-0015 (ADR-Driven Development Methodology) establishes that tests trace back to ADRs via pytest markers. This ADR defines what those tests look like, how they are organized, and what coverage means for the project.

## Decision

### Test framework

pytest with pytest-asyncio. No other test frameworks, no unittest-style classes. Tests are functions, not methods.

### Test layers

Tests are organized into three layers that mirror the architecture:

**Adapter tests** verify that an adapter correctly implements the `SignalAdapter` protocol.

- Every adapter (fixture, OTLP, future backends) gets its own test module.
- Adapter tests verify method signatures, return types (Pydantic models), and data completeness.
- Fixture adapter tests assert against known values in the fixture data set (ADR-0016).
- Live adapter tests (when they exist) are marked `@pytest.mark.integration` and excluded from default CI runs.

**Signal layer tests** verify derived behavior: service map construction from trace spans, error extraction from spans, and any future derivations.

- These tests run against fixture adapter output.
- They verify the transformation logic, not the adapter — if the adapter returns valid data, the signal layer must produce correct derived results.

**Tool tests** verify the MCP tool interface: parameter validation, default behaviors (ADR-0005), response structure (ADR-0013), and that tools call the correct signal/adapter methods.

- Tool tests run end-to-end through the fixture adapter — no mocking of the adapter layer.
- They verify the contract between "agent calls a tool with these parameters" and "agent receives this response shape."

### Convention tests

A distinct category of tests verifies project-wide invariants drawn from ADRs:

- **Terminology (ADR-0006):** Scan tool descriptions, docstrings, and user-facing strings for forbidden terms ("production," "autonomous," "automatically," "AI makes decisions").
- **Governance boundary (ADR-0014):** Assert that no tool performs write operations or includes action recommendations in response content.
- **Tool naming (ADR-0003):** Assert all registered tools use the `traceforward_` prefix.
- **Defaults (ADR-0005):** Assert `lookback_hours` defaults to `24` and `limit` defaults to `50` when parameters are omitted.

Convention tests use the `@pytest.mark.adr("NNNN")` marker from ADR-0015. They are the automated enforcement of architectural decisions that would otherwise rely on code review alone.

### Drift detection

Convention tests also detect drift between ADR governance and implementation artifacts (ADR-0015).

- **Traceability drift.** A test scans all Python modules under the core package (`adapters/`, `signals/`, `tools/`) and flags any module missing a `Governing decisions:` docstring reference. This test warns but does not fail — missing references are tracked, not blocked.
- **Marker drift.** A test verifies that every ADR referenced in a `@pytest.mark.adr()` marker corresponds to an accepted ADR in `docs/adr/`. A test claiming to verify a nonexistent or superseded ADR is a stale reference.
- **Prompt drift.** If the session prompt is versioned in the repository (per ADR-0015), a test can verify that ADR numbers referenced in the prompt correspond to accepted ADRs. This catches prompts that reference renamed or superseded decisions.

Drift detection tests live in `test_conventions.py` alongside terminology and naming checks. They are lightweight — AST or string scanning, not semantic analysis. Their purpose is visibility, not enforcement. A warning in the test output is enough to prompt a developer to update a reference; a hard failure would create friction that discourages iteration.

### Mocking policy

- **No mocking of internal layers.** Tool tests call through the signal layer to the adapter. Signal tests call through to the adapter. The fixture adapter exists so that mocking is unnecessary for the default test path.
- **Mocking is permitted only at external boundaries** that do not exist in fixture mode: HTTP clients for live adapters, filesystem operations if ever needed, system clock if time-sensitive logic is introduced (via `freezegun` or equivalent).
- **No mock adapters.** The fixture adapter is the test adapter. Creating a separate mock adapter would duplicate the protocol implementation and drift from the reference.

### Coverage policy

- **Target: 90% line coverage** on the core package (adapters, signals, tools). This is a floor, not a ceiling.
- **Not 100%.** Defensive error handling paths (e.g., adapter raises an unexpected exception type) and edge cases in configuration loading may not be worth the test investment at this stage. Chasing 100% creates tests that verify implementation details rather than decisions.
- **Coverage is measured but not gated in CI** for the initial release. Once the test suite stabilizes, a CI gate at 90% can be added. Gating too early on a project this young creates friction that slows iteration without proportional safety benefit.
- **Coverage gaps are tracked.** When a module falls below the target, the gap is noted — not ignored, not immediately fixed, but visible.

### Test file naming and location

```
tests/
    test_adapter_fixture.py       # Fixture adapter protocol compliance
    test_adapter_protocol.py      # Abstract protocol shape tests
    test_signals_service_map.py   # Service map derivation
    test_signals_errors.py        # Error extraction
    test_tool_list_services.py    # One module per tool
    test_tool_get_traces.py
    test_tool_get_metrics.py
    test_tool_get_logs.py
    test_tool_get_errors.py
    test_tool_get_service_map.py
    test_conventions.py           # Terminology, naming, defaults, governance, drift
```

Test files mirror the thing they test, not the ADR they verify. ADR traceability comes from markers, not file organization.

### Running tests

```bash
mise run test              # All tests except integration
mise run test:integration  # Live backend tests (requires running backend)
mise run test:coverage     # With coverage report
pytest -m "adr('0003')"   # Only tests verifying ADR-0003
pytest -m "adr('0006')"   # Only terminology convention tests
```

## Consequences

- **Full-stack confidence.** Tests run through all three layers against deterministic data, catching integration issues that unit-level mocks would miss.
- **ADR verification.** Convention tests turn architectural prose into automated assertions. A decision in an ADR is not just documented — it is enforced.
- **Drift visibility.** Traceability, marker, and prompt drift tests surface governance gaps without blocking development. The project knows where its references are stale — and can fix them on its own schedule.
- **Fast feedback.** No external dependencies in the default test path means tests run in seconds, locally and in CI.
- **Trade-off.** No mocking of internal layers means a bug in the fixture adapter can cause failures in signal and tool tests. This is accepted — the fixture adapter is the reference implementation (ADR-0016), so a bug there *should* be visible everywhere.
- **Trade-off.** 90% coverage leaves room for untested paths. This is preferable to test bloat that verifies plumbing rather than behavior. The threshold is revisited as the project matures.
- **Trade-off.** Convention tests that scan strings (terminology, naming) are brittle if the scanning logic is naive. These tests should match against tool registrations and public-facing content, not grep the entire codebase.
- **Trade-off.** Drift detection tests that warn but do not fail may be ignored. This is accepted — the alternative (hard failure) would block commits for missing docstring references, which violates the "not a bureaucratic process" principle in ADR-0015.
