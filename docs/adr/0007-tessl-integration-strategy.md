# 0007 — Tessl Integration Strategy

**Date:** 2026-03-24
**Updated:** 2026-04-01
**Status:** Accepted

## Context

Tessl.io is a platform for managing context for coding agents, treating agent skills and context as software with a complete lifecycle: Use → Create → Evaluate → Distribute. Tessl provides a package manager (CLI), a registry for discovery and distribution, and an evaluation framework that measures how effectively context improves agent behavior.

TraceForward, as a tool that provides environment signal data to coding agents, has a natural relationship with this ecosystem. In Patrick Debois's Code Development Lifecycle (CDLC) framework — Generate → Evaluate → Distribute → Observe — TraceForward maps to the **Observe → Generate** bridge: it observes environment signals and delivers that intelligence to the generation phase (coding agents writing or modifying code). Tessl's own Context Lifecycle provides the packaging and distribution mechanism for making TraceForward discoverable and installable.

The question is whether Tessl integration should be deferred until the core MVP is stable, or run in parallel from day one.

## Decision

Tessl integration runs **in parallel with the TraceForward MVP** from day one, on two tracks:

### Track 1: Publish to the Tessl Registry

Publish TraceForward as a **tile** (Tessl's versioned packaging unit) under the `bigelow` namespace on the Tessl Registry. A tile can contain skills, documentation, and rules. The TraceForward tile includes:

- **Skill.** Instructions that teach coding agents how to use TraceForward's MCP tools effectively — when to call each tool, how to interpret responses, and how to combine tools for common workflows (e.g., check errors before suggesting a restart). This is not TraceForward's code; it is the context that helps agents use TraceForward correctly.
- **Documentation.** Agent-consumable reference for tool parameters, response shapes, and adapter configuration.
- **Rules.** Constraints aligned with ADR-0014 (Action and Governance Boundary) — TraceForward tools are read-only, responses contain signal data only, no action recommendations.

Installation via the Tessl CLI:

```
npx tessl i github:bigelow/traceforward-mcp
```

### Track 2: Map to the CDLC model

TraceForward is the Observe → Generate bridge in Debois's CDLC: it observes environment signals and delivers that intelligence to the generation phase. This mapping gives TraceForward a defined role within a recognized lifecycle model rather than existing as an unpositioned MCP server.

Tessl's Context Lifecycle (Use → Create → Evaluate → Distribute) is the mechanism by which TraceForward's skill context reaches agents. The two models are complementary: CDLC describes TraceForward's *role*; Tessl's lifecycle describes how that role is *packaged and delivered*.

CDLC is attributed to Patrick Debois.

### Evaluation

Tessl's evaluation framework measures agent success rates with and without context. Once the TraceForward tile is published, Tessl evals validate that agents using the TraceForward skill make better decisions (e.g., checking error context before suggesting a restart) compared to agents without it. Eval scores are visible on the registry, providing public evidence of the skill's effectiveness.

### Future: Staleness detection

Tessl evals can also detect staleness — when published skill context has drifted from current TraceForward behavior. Periodic eval runs against the latest TraceForward release surface regressions in skill quality. This aligns with Tessl's position that context should be treated as software with continuous validation, not a static artifact.

### Future: GitHub Actions automation

Tessl supports review and publish via GitHub Actions. A future TraceForward CI workflow could automatically lint, evaluate, and publish updated tiles to the registry when the skill content changes. This is not in scope for the MVP but is a natural automation step.

## Consequences

- **Ecosystem alignment.** Publishing on the Tessl Registry makes TraceForward discoverable through the same tooling agents already use for context management. The registry is the natural distribution point for agent-consumable skills.
- **CDLC coherence.** Mapping to Observe → Generate gives TraceForward a defined role within a recognized lifecycle model, making its purpose clear to adopters and contributors.
- **Evaluated quality.** Tessl evals provide measurable validation that the TraceForward skill improves agent behavior. Eval scores are public on the registry.
- **Security posture.** Published tiles receive automated Snyk security scanning through the Tessl Registry, adding a layer of third-party validation.
- **Parallel work.** Running both tracks simultaneously means neither blocks the other — core development continues while tile packaging is a lightweight parallel effort.
- **Trade-off.** Tessl is an early-stage platform. If the platform pivots or adoption stalls, the Tessl-specific packaging work has limited reuse. The core MCP server is unaffected regardless.
- **Trade-off.** Maintaining two distribution paths (direct MCP installation and Tessl tile) adds surface area. The tile is a thin wrapper containing skill instructions and docs, so the maintenance cost is low.
- **Trade-off.** Tessl evals depend on the eval framework's design and scoring criteria. Eval results are useful but should not be the sole measure of TraceForward's effectiveness.
