# 0004 — Python, uv, and Pydantic v2 Stack

**Date:** 2026-03-24
**Status:** Accepted

## Context

TraceForward needs a language and toolchain that supports rapid development, strong typing for data models, and alignment with the MCP ecosystem.

Alternatives considered:

- **TypeScript** — strong MCP SDK support, but the maintainer's deep expertise is in Python and the observability/infrastructure ecosystem is Python-heavy.
- **Go** — excellent for systems tooling, but lacks a mature MCP SDK and Pydantic-equivalent data validation.
- **Python + pip/poetry** — viable, but pip lacks deterministic lockfiles by default and poetry's resolver can be slow.

## Decision

We use:

- **Python** as the implementation language.
- **uv** for dependency management and virtual environment handling. Fast, deterministic, and replacing pip/poetry in modern Python workflows.
- **Pydantic v2** for all data models (adapter responses, tool inputs/outputs, configuration). Pydantic v2's Rust-based validation is fast, and its integration with FastMCP provides automatic tool input validation.
- **mise** (mise.toml) for tool version management and task running across the project.

## Consequences

- **Speed.** uv's dependency resolution and installation is significantly faster than pip or poetry, improving CI and contributor onboarding.
- **Type safety.** Pydantic v2 models enforce structure at the boundary — adapters must return well-typed data, and MCP tools receive validated inputs.
- **FastMCP alignment.** FastMCP is Python-native and uses Pydantic for tool schemas. This stack choice eliminates impedance mismatch.
- **mise consistency.** All project tasks (test, lint, run) are defined in mise.toml, providing a single entry point regardless of environment.
- **Trade-off.** Python is slower than Go or Rust for compute-heavy work. TraceForward is I/O-bound (querying backends), so this is not a concern.
- **Trade-off.** uv is newer than pip/poetry. If uv development stalls, migration to another tool is straightforward since uv uses standard pyproject.toml.
