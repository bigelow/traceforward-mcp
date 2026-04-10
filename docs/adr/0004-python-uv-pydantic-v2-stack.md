# 0004 — Python, uv, and Pydantic v2 Stack

**Date:** 2026-03-24
**Status:** Accepted

## Context

TraceForward needs an implementation stack that supports rapid iteration, strong model validation, and close alignment with the MCP ecosystem. The stack also needs to fit the project’s observability and infrastructure focus, where Python is widely used for tooling, automation, and backend integration.

Alternatives considered:

* **TypeScript** — strong MCP SDK support, but less aligned with the maintainer’s primary implementation background and with the surrounding observability and infrastructure tooling used by the project.
* **Go** — strong fit for systems tooling, but lacks an equivalent to Pydantic for model validation and has a less mature MCP ecosystem.
* **Python with pip or Poetry** — viable, but less aligned with the project’s preference for fast, deterministic dependency workflows.

## Decision

TraceForward will use Python as its implementation language.

The project will use:

* **uv** for dependency management and virtual environment handling
* **Pydantic v2** for all data models, including adapter responses, tool inputs and outputs, and configuration
* **mise** for tool version management and task execution across the project

Pydantic v2 provides the typed model boundary for the system, and FastMCP integration allows those models to flow directly into MCP tool schemas and validation.

## Consequences

* **Python ecosystem fit.** The implementation language aligns with the project’s observability, infrastructure, and automation focus.
* **Validated model boundary.** Pydantic v2 enforces structure at system boundaries, including adapter outputs, configuration, and MCP tool inputs and outputs.
* **FastMCP alignment.** The stack avoids impedance mismatch between the application layer, model layer, and MCP interface layer.
* **Deterministic workflow.** `uv` and `pyproject.toml` provide a fast, modern dependency workflow with standard Python packaging semantics.
* **Consistent project entry points.** `mise` provides a single task and tool management layer for development and CI workflows.
* **Trade-off.** Python is not the best choice for compute-heavy workloads. TraceForward is primarily I/O-bound, so this is an acceptable trade-off.
* **Trade-off.** `uv` is newer than older Python packaging tools. If the toolchain changes later, migration remains straightforward because the project is anchored on standard Python project metadata.tools.
