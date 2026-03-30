# 0007 — Tessl Integration Strategy

**Date:** 2026-03-24
**Status:** Accepted

## Context

Tessl.io provides a platform for publishing and distributing AI-consumable skills through its Code Development Lifecycle (CDLC) framework: Generate → Evaluate → Distribute → Observe. TraceForward, as a tool that provides environment signal data to coding agents, naturally maps to the **Observe → Generate** bridge — closing the CDLC loop by feeding runtime intelligence back into the generation phase.

The question is whether Tessl integration should be deferred until the core MVP is stable, or run in parallel from day one.

## Decision

Tessl integration runs **in parallel with the TraceForward MVP** from day one, on two tracks:

1. **Publish TraceForward as a skill on Tessl's ship registry** under the `bigelow` namespace. This makes TraceForward discoverable and installable through Tessl's distribution mechanism.
2. **Map TraceForward's role to the CDLC model.** TraceForward is the Observe → Generate bridge: it observes environment signals and delivers that intelligence to the generation phase (coding agents writing or modifying code).

CDLC is attributed to Patrick Debois.

### Future: Staleness Detection

A backlog item explores staleness detection via scheduled evaluation runs — when Meridian context is published as Tessl skills, periodic eval runs can detect when published intelligence has drifted from current environment state. This is not in scope for the MVP.

## Consequences

- **Early ecosystem presence.** Publishing on Tessl from day one establishes TraceForward in the CDLC ecosystem before the space gets crowded.
- **CDLC coherence.** Explicitly mapping to Observe → Generate gives TraceForward a clear narrative position that differentiates it from generic MCP servers.
- **Parallel work.** Running both tracks simultaneously means neither blocks the other — core development continues while the Tessl skill packaging is a lightweight parallel effort.
- **Trade-off.** Tessl is an early-stage platform. If the platform pivots or adoption stalls, the Tessl-specific packaging work has limited reuse. The core MCP server is unaffected regardless.
- **Trade-off.** Maintaining two distribution paths (direct MCP installation and Tessl skill) adds surface area. The Tessl skill is a thin wrapper, so the maintenance cost is low.
