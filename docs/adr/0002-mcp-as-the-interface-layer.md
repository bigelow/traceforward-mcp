# 0002 — MCP as the Interface Layer

**Date:** 2026-03-24
**Status:** Accepted

## Context

TraceForward's purpose is to give AI coding agents access to environment signal data before they write or modify code. The interface must be natively consumable by LLM-based agents without requiring the agent to learn a bespoke API, authenticate separately, or parse unfamiliar response formats.

Alternatives considered:

- **REST API** — universal, but requires each agent framework to build a custom integration. No standard discovery mechanism for agent tooling.
- **GraphQL** — flexible query surface, but adds schema complexity and is not natively understood by any major agent framework.
- **MCP (Model Context Protocol)** — emerging standard for tool-use by LLM agents, with native support in Claude, Cursor, Windsurf, and others. Provides tool discovery, structured input/output, and a standardized transport layer.

## Decision

We use **MCP** as TraceForward's interface layer, implemented with **FastMCP** (Python).

FastMCP provides decorator-based tool registration, automatic input validation via Pydantic, and built-in transport support (stdio, SSE) — aligning with our Python + Pydantic v2 stack.

TraceForward exposes its capabilities as MCP tools, discoverable and callable by any MCP-compatible agent.

## Consequences

- **Agent-native.** Coding agents that support MCP can discover and use TraceForward without custom integration code.
- **Low friction.** FastMCP's decorator pattern keeps tool definitions close to the implementation, reducing boilerplate.
- **Ecosystem alignment.** MCP adoption is growing across agent frameworks; TraceForward benefits from that momentum.
- **Trade-off.** MCP is still a young protocol. Breaking changes in the spec or in FastMCP could require updates. We mitigate by pinning FastMCP versions and tracking the spec.
- **Trade-off.** Teams without MCP-compatible agents cannot use TraceForward directly. A REST wrapper is a potential future addition but is not in scope for the MVP.
