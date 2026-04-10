# 0002 — MCP as the Interface Layer

**Date:** 2026-03-24
**Status:** Accepted

## Context

TraceForward exists to give AI coding agents access to environment signal data before they write or modify code. The interface must be consumable by LLM-based agents without requiring each agent framework to integrate a bespoke API, manage separate authentication flows, or translate backend-specific response formats.

Alternatives considered:

* **REST API** — broadly understood, but requires each agent framework to implement a custom integration. It also provides no standard mechanism for tool discovery.
* **GraphQL** — flexible query surface, but adds schema complexity and is not natively supported as an agent tool interface by major coding-agent environments.
* **MCP (Model Context Protocol)** — emerging standard for LLM tool use, with growing support in Claude, Cursor, Windsurf, and similar agent environments. Provides tool discovery, structured inputs and outputs, and a standard transport model.

## Decision

TraceForward will expose its interface exclusively through MCP.

The MCP layer is implemented in Python using **FastMCP**. TraceForward capabilities are exposed as MCP tools that can be discovered and invoked by MCP-compatible agents.

FastMCP aligns with the existing Python and Pydantic v2 stack, supports decorator-based tool registration, and provides transport options such as `stdio` and SSE.

## Consequences

* **Agent-native integration.** MCP-compatible agents can discover and use TraceForward without custom integration code.
* **Consistent tool contract.** Tool discovery, invocation, and structured input/output are handled through a single protocol boundary.
* **Implementation simplicity.** FastMCP keeps tool definitions close to the implementation and reduces interface boilerplate.
* **Ecosystem alignment.** TraceForward benefits from adoption of MCP across coding-agent environments instead of maintaining framework-specific integrations.
* **Trade-off.** MCP remains an evolving protocol. Changes in the specification or in FastMCP may require updates to TraceForward.
* **Trade-off.** Non-MCP clients are out of scope for the MVP. Supporting them would require an additional interface layer, such as a REST wrapper.
