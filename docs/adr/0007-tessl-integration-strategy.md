# 0007 — External Distribution of Agent Guidance

**Date:** 2026-03-24
**Updated:** 2026-04-01
**Status:** Accepted

## Context

TraceForward’s core runtime is an MCP server that exposes environment signal data to coding agents. In addition to the server itself, agents may benefit from external guidance describing how to use TraceForward effectively — when to call specific tools, how to interpret responses, and what governance boundaries apply.

Platforms such as Tessl provide packaging, distribution, and evaluation mechanisms for agent-facing context such as skills, rules, and documentation. These platforms may be useful for distributing TraceForward usage guidance, but they are external to the core TraceForward runtime.

The project needs to decide whether external distribution platforms are part of the core architecture or optional ecosystem integrations.

## Decision

TraceForward will keep its MCP server and core runtime architecture independent from any single external distribution platform.

External systems such as Tessl may be used to package and distribute:

* agent usage guidance
* tool documentation
* governance rules
* evaluation artifacts

These materials are separate from the TraceForward server implementation. They are optional ecosystem integrations, not part of the core runtime boundary.

TraceForward remains installable and usable directly through its own MCP server regardless of whether external packaging or registry integrations exist.

## Consequences

* **Core independence.** TraceForward’s runtime architecture does not depend on any external registry, packaging system, or evaluation platform.
* **Optional ecosystem reach.** External platforms such as Tessl can improve discoverability, packaging, and agent guidance distribution without becoming runtime dependencies.
* **Clear separation of concerns.** The MCP server remains the source of runtime signal access; external packaging systems distribute guidance about how to use it.
* **Lower lock-in risk.** If a distribution platform changes direction, adoption stalls, or disappears, TraceForward’s core runtime remains unaffected.
* **Trade-off.** Guidance distributed through external systems may drift from current TraceForward behavior unless maintained alongside the project.
* **Trade-off.** Maintaining both direct MCP distribution and external packaging adds some overhead, even when the external artifacts are intentionally thin.
* **Trade-off.** External evaluation results may be useful signals, but they do not define TraceForward’s architectural correctness or overall value.
