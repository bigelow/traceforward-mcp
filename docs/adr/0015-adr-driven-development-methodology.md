# 0015 — ADR-Driven Development Methodology

**Date:** 2026-04-01
**Status:** Accepted

## Context

TraceForward has a growing set of architectural decisions (ADR-0001 through ADR-0014) that define how the system is designed, what conventions it follows, and where its boundaries are. These decisions are only valuable if the codebase reflects them — and if that relationship is traceable.

Without an explicit methodology, the connection between "why we decided X" and "where X is implemented and verified" lives in the developer's head. This breaks down as the project grows, as contributors join, and as AI coding agents are used for implementation. An agent handed a task with no link back to the governing ADR will make reasonable but ungoverned choices.

Traceability also matters for the project's public credibility. TraceForward is an opinionated tool — its ADRs are part of its value proposition. Demonstrating that the code follows its own governance is a signal of engineering rigor.

## Decision

Every implementation artifact in TraceForward — module, test, CI check — shall be traceable to one or more ADRs. This traceability is lightweight and convention-based, not tooling-heavy.

### Source code traceability

Modules include a docstring or header comment referencing the governing ADR(s):

```python
"""Service map derivation from trace spans.

Governing decisions:
    ADR-0001 (Adapter-based signal ingestion)
    ADR-0003 (Tool surface design — derived tools)
"""
```

This is not metadata for a tool to parse. It is context for the next developer (or agent) reading the file.

### Test traceability

Tests use pytest markers to declare which ADR(s) they verify:

```python
import pytest

adr = pytest.mark.adr

@adr("0006")
def test_no_forbidden_terms_in_tool_descriptions():
    """ADR-0006: Language and terminology conventions."""
    ...
```

A custom `adr` marker allows filtering: `pytest -m "adr('0003')"` runs only tests that verify ADR-0003 decisions. The marker is registered in `pyproject.toml` to avoid warnings.

### ADR coverage visibility

A simple mapping file (`docs/adr/TRACEABILITY.md`) maintains the relationship between ADRs and their implementing/verifying artifacts:

```markdown
| ADR  | Implemented in              | Verified by                        |
|------|-----------------------------|------------------------------------|
| 0001 | adapters/protocol.py        | tests/test_adapter_protocol.py     |
| 0003 | tools/*.py                  | tests/test_tool_surface.py         |
| 0006 | (convention)                | tests/test_terminology.py          |
| 0014 | (convention)                | tests/test_governance_boundary.py  |
```

This file is manually maintained. It is a reference, not a gate. If it drifts, a contributor can update it — the source-level references and test markers are the authoritative links.

### Human authority and accountability

AI coding agents write code for TraceForward. Humans commit it. This distinction is non-negotiable.

- **Humans own git.** No agent commits directly to any branch. Every commit is reviewed and authored by a human. The agent produces code and a recommended CHANGELOG entry and conventional commit string; the human decides whether to accept, modify, or reject.
- **The committer is accountable.** When a human commits agent-produced code, they accept responsibility for what that code does — its correctness, its adherence to ADRs, and its consequences. The agent is a tool; the human is the author of record.
- **Review is not optional.** Agent output is treated as a draft, not a deliverable. The human reviews for ADR alignment, code quality, and unintended side effects before committing. "The agent wrote it" is not a defense for governance violations.

### Prompt governance

The Claude Code session prompt is a governing artifact. It defines the constraints, conventions, and context that shape agent behavior during implementation. Changes to the session prompt can change the agent's output as much as changes to an ADR.

- **Session prompts reference governing ADRs.** When a Claude Code session targets a specific area of the codebase, the prompt includes the relevant ADR numbers so the agent has explicit architectural context.
- **Session prompts are versioned.** The greenfield session prompt lives in the repository and is tracked in version control. Changes to the prompt are committed with the same intentionality as changes to code or ADRs.
- **Prompt drift is a real risk.** An outdated or vague session prompt produces agent output that may not align with current decisions. Prompt maintenance is part of project maintenance.

### Agent session capture

Agent sessions that produce committed code are captured via Entire.io (ADR-0018). This creates a three-layer traceability chain:

- **ADR** — why the decision was made
- **Session** — how the agent interpreted and implemented the decision
- **Commit** — what code was produced

Session capture is not a replacement for code review or ADR references. It is the audit trail that connects the other two.

### What this is not

- Not a CI gate. There is no automated check that every file references an ADR. That would create friction disproportionate to the value.
- Not a coverage metric. Not every line of code maps to an ADR. Utility functions, configuration, and glue code exist without a governing decision — and that is fine.
- Not a bureaucratic process. If a developer writes a module and forgets the docstring reference, the code still ships. The reference gets added when someone notices, not as a merge blocker.
- Not agent autonomy. The methodology describes how agents are *used*, not how agents *operate independently*. Agents do not self-assign work, choose which ADRs to follow, or decide when their output is ready to commit.

## Consequences

- **Traceability.** Any contributor (human or agent) can start from an ADR and find the code that implements it, or start from code and find the decision that governs it.
- **Agent effectiveness.** Coding agents given ADR references as context produce implementations that align with project governance without needing to infer intent from code alone.
- **Lightweight.** Convention-based traceability (docstrings, markers, a markdown table) requires no tooling, no CI plugins, and no process overhead beyond the habit of referencing decisions.
- **Clear accountability.** Human review and human commits mean there is always a responsible party for every line of code, regardless of whether an agent wrote the first draft.
- **Audit depth.** The ADR → session → commit chain (via ADR-0018) provides traceability that most projects lack entirely — not just what was decided and what was built, but how the building happened.
- **Trade-off.** Manual maintenance means the traceability map can drift. This is accepted — the cost of enforcement tooling exceeds the cost of occasional drift for a project of this scale.
- **Trade-off.** Not all code traces to an ADR. This is intentional. Over-documenting trivial implementation choices would dilute the signal value of ADR references.
- **Trade-off.** Prompt governance adds a maintenance burden. Session prompts must be kept current as ADRs evolve. This cost is justified — an outdated prompt is an ungoverned agent.
