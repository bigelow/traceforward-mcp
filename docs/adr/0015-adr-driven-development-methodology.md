# 0015 — ADR-Driven Development Methodology

**Date:** 2026-04-01
**Status:** Accepted

## Context

TraceForward’s ADRs define system boundaries, conventions, and architectural decisions. Those decisions are only valuable if contributors can trace them into implementation and verification artifacts.

Without an explicit methodology, the relationship between “why this decision exists” and “where it is implemented or tested” remains implicit. That breaks down as the project grows, as contributors join, and as coding agents are used to assist implementation.

TraceForward’s ADRs are part of the project’s engineering discipline. The codebase should make those decisions visible and traceable without requiring heavy process or specialized tooling.

## Decision

TraceForward adopts a lightweight ADR traceability methodology for code, tests, and key implementation artifacts.

The methodology is convention-based rather than tool-enforced. Its purpose is to make architectural intent visible at the point of implementation and verification.

### Source code traceability

Modules may include a docstring or header comment referencing the governing ADR or ADRs.

Example:

```python
"""Service map derivation from trace spans.

Governing decisions:
    ADR-0001 (Adapter-based signal ingestion)
    ADR-0003 (Tool surface design — derived tools)
"""
```

These references exist for maintainers and contributors reading the code. They are contextual guidance, not a machine-enforced metadata system.

### Test traceability

Tests may declare the ADRs they verify using a pytest marker.

Example:

```python
import pytest

adr = pytest.mark.adr

@adr("0006")
def test_no_forbidden_terms_in_tool_descriptions():
    """ADR-0006: Language and terminology conventions."""
    ...
```

This allows ADR-oriented test selection and makes verification intent explicit.

### ADR coverage visibility

A simple reference file such as `docs/adr/TRACEABILITY.md` may map ADRs to implementation and verification artifacts.

Example:

```markdown
| ADR  | Implemented in              | Verified by                        |
|------|-----------------------------|------------------------------------|
| 0001 | adapters/protocol.py        | tests/test_adapter_protocol.py     |
| 0003 | tools/*.py                  | tests/test_tool_surface.py         |
| 0006 | (convention)                | tests/test_terminology.py          |
| 0014 | (convention)                | tests/test_governance_boundary.py  |
```

This file is a navigational aid, not a source of truth. If it drifts, it can be corrected through normal maintenance.

### Methodology boundaries

This methodology is intended to make decisions easier to trace, not to create a coverage or compliance regime.

It does **not** require:

* every file to reference an ADR
* every line of code to map to an ADR
* CI enforcement that blocks changes when traceability references are missing

Traceability is expected for architectural and behavior-defining code paths, not for every incidental utility or piece of glue code.

## Consequences

* **Architectural visibility.** Contributors can move from a decision to the code and tests that implement it, and from code back to the governing decision.
* **Better implementation alignment.** Humans and coding agents alike can use ADR references to understand architectural intent before changing code.
* **Low process overhead.** Docstrings, test markers, and a simple reference file provide useful traceability without adding heavy tooling or merge friction.
* **Maintainable discipline.** The methodology supports rigor without turning ADR traceability into a bureaucratic gate.
* **Trade-off.** Because the system is convention-based, traceability references may drift or be omitted.
* **Trade-off.** Not all code will map directly to an ADR, and that is intentional. Over-documenting trivial implementation details would dilute the value of the traceability signal.
