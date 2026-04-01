# 0018 — Agent Session Capture with Entire.io

**Date:** 2026-04-01
**Status:** Accepted

## Context

TraceForward is built using AI coding agents (Claude Code) governed by architectural decisions (ADR-0015). When an agent writes code that a human reviews and commits, the session that produced that code — the prompts, reasoning, iterations, and decisions made during the session — is valuable context that would otherwise be lost.

Git captures *what* changed. Commit messages capture *why* at a summary level. Neither captures *how* the change was developed: what the agent was asked, what it tried, what it rejected, and what constraints it operated under. This session history is the missing layer between "a decision was made" (ADR) and "code was committed" (git).

Without session capture, agent-assisted development has an audit gap. If a contributor questions a design choice six months from now, the ADR explains the governing decision and the code shows the implementation, but the reasoning that connected the two is gone.

This gap also matters for TraceForward's relationship with Meridian. Agent session data — prompts, reasoning chains, code changes correlated with commits — is a future adapter source for Meridian's ingestion pipeline. Capturing it now in a structured format creates the foundation for that integration.

## Decision

TraceForward uses **Entire.io** for agent session capture. Entire.io hooks into the git workflow to capture AI agent sessions (prompts, reasoning, changes) as searchable checkpoints alongside commits.

### What is captured

- **Session checkpoints.** Entire.io captures Claude Code session state at commit-aligned checkpoints on the `entire/checkpoints/v1` namespace. Each checkpoint preserves the prompts given to the agent, the agent's reasoning and output, and the code changes produced.
- **Correlation with commits.** Checkpoints are tied to the git history, making it possible to trace from a commit back to the agent session that produced it.
- **No credentials or secrets.** Session capture follows the same redaction principles as ADR-0011 (Redaction, Disclosure, and Data Handling Policy). Sensitive values in prompts or agent output are the developer's responsibility to exclude before committing.

### What is not captured

- **Human review decisions.** Entire.io captures what the agent produced, not the human's review process (what was accepted, rejected, or modified before commit). That context lives in the commit diff and any PR discussion.
- **Informal sessions.** Exploratory agent conversations that do not result in committed code are not captured. This is acceptable — the audit trail matters for code that ships, not for brainstorming.

### Integration points

- **Git workflow.** Entire.io operates alongside git, not as a replacement for any part of the commit workflow. The developer commits code normally; Entire.io captures the session context that produced it.
- **Claude Code.** Entire.io is active during Claude Code sessions on the TraceForward repo. Session prompts reference governing ADRs (per ADR-0015), and the captured checkpoints preserve that context.
- **Future Meridian adapter.** Agent session data captured by Entire.io is a candidate source for a Meridian Bronze adapter — agent sessions correlated with commits, correlated with traces, feeding the governed intelligence pipeline. This integration is deferred but the data format is being established now.

## Consequences

- **Session traceability.** Any committed code can be traced back through the agent session that produced it, closing the gap between ADR (why) → session (how) → commit (what).
- **Audit trail.** For a project that positions itself as a model for agent-assisted development, demonstrating that agent sessions are captured and searchable is a credibility signal.
- **Meridian pipeline readiness.** Capturing structured session data now avoids retrofitting when the Meridian adapter integration is built.
- **Trade-off.** Entire.io is an external dependency for the development workflow, not the runtime. If Entire.io becomes unavailable, the development process continues — session capture is lost but code production is unaffected.
- **Trade-off.** Session checkpoints add volume to the repository's ancillary data. The `entire/checkpoints/v1` namespace keeps this separate from source code, but storage growth should be monitored.
- **Trade-off.** Only sessions that result in commits are captured. Valuable exploratory sessions that inform decisions without producing code are not preserved. This is an acceptable scope limit — capturing everything would create noise that dilutes the audit trail's value.
