# TraceForward

## What is TraceForward?

TraceForward is an MCP server that exposes OpenTelemetry-backed runtime context to coding agents.

## The problem

Coding agents ask good questions but have no way to look up the answers. They don't know what a service depends on, how it's been behaving, or what errors have been showing up. TraceForward gives coding agents a way to query that runtime context directly.

## How it works

```
┌──────────────┐     ┌──────────────┐     ┌─────┐     ┌──────────────┐
│ OTel backend │ ──▶ │ TraceForward │ ──▶ │ MCP │ ──▶ │ Coding Agent │
└──────────────┘     └──────────────┘     └─────┘     └──────────────┘
```

1. Your OpenTelemetry backend collects traces, metrics, and logs from your running services.
2. TraceForward connects to that backend and exposes the signal data through a standard adapter interface.
3. The MCP protocol makes that data available as tools your coding agent can call.
4. Your agent queries service dependencies, error rates, and recent failures before writing or modifying code.

## Quick start

### Fixture mode — the reference implementation

Fixture mode is TraceForward's reference implementation. It ships with realistic, deterministic sample data so you can develop, test, and evaluate TraceForward without connecting to a running backend. The fixture adapter implements the full `SignalAdapter` protocol — it is not a mock or a demo shortcut.

```bash
mise run dev
```

This starts TraceForward on `stdio` with the fixture adapter. Point your agent at it and start asking questions.

### Live mode — connect to your OTel backend

Set the adapter mode and configure your backend URLs:

```bash
export ADAPTER_MODE=otlp
export OTLP_TRACES_URL=http://localhost:4318/v1/traces
export OTLP_METRICS_URL=http://localhost:4318/v1/metrics
export OTLP_LOGS_URL=http://localhost:4318/v1/logs
mise run dev:otlp
```

### Register with your agent

TraceForward works with any MCP-compatible agent. Point your agent at the server using its MCP configuration.

**Claude Code** — add to `~/.claude/claude.json`:

```json
{
  "mcpServers": {
    "traceforward": {
      "command": "mise",
      "args": ["run", "dev"],
      "cwd": "/path/to/traceforward-mcp"
    }
  }
}
```

**Cursor** — add to `.cursor/mcp.json` in your project root:

```json
{
  "mcpServers": {
    "traceforward": {
      "command": "mise",
      "args": ["run", "dev"],
      "cwd": "/path/to/traceforward-mcp"
    }
  }
}
```

Other MCP-compatible agents (Windsurf, Cline, etc.) follow similar patterns — consult your agent's MCP configuration docs.

## Tools reference

TraceForward exposes a small tool surface aimed at the first questions an engineer or agent would ask during triage.

| Tool | What question it answers |
|------|------------------------|
| `traceforward_list_services` | What services exist in this environment? |
| `traceforward_get_service_map` | What does this service depend on, and what depends on it? |
| `traceforward_get_traces` | What do recent requests to this service look like end-to-end? |
| `traceforward_get_metrics` | How is this service performing — latency, throughput, error rate? |
| `traceforward_get_errors` | What's been failing in this service recently? |
| `traceforward_get_logs` | What has this service been logging? |

## Adding a backend

To add a backend, implement the `SignalAdapter` protocol in `adapters/`.

See `adapters/fixture.py` for the reference implementation.

## Sample session

An agent is asked to investigate checkout failures in payment-service and queries TraceForward before suggesting a fix.

**Agent calls `traceforward_get_errors(service="payment-service")`:**

```json
[
  {
    "operation_name": "process_payment",
    "service_name": "payment-service",
    "span_id": "span-011",
    "timestamp": "2026-03-18T10:05:00.005Z",
    "trace_id": "trace-002"
  }
]
```

**Agent calls `traceforward_get_logs(service="payment-service", level="error")`:**

```json
[
  {
    "level": "error",
    "message": "Payment gateway timeout after 5000ms",
    "service_name": "payment-service",
    "timestamp": "2026-03-18T10:05:00.300Z",
    "trace_id": "trace-002"
  },
  {
    "level": "error",
    "message": "Stripe API returned 402: card_declined",
    "service_name": "payment-service",
    "timestamp": "2026-03-18T10:05:00.350Z",
    "trace_id": "trace-002"
  }
]
```

Now the agent knows: the failures are a gateway timeout and a card decline from Stripe — not a crashed process. Restarting the service won't fix either problem. Instead of suggesting a restart, the agent can suggest investigating the upstream payment gateway.

## Architecture decisions

Architectural decisions are documented in ADRs before implementation. The ADRs define the adapter model, tool surface, security boundaries, and terminology used across the project.

See the [ADR index](docs/adr/README.md) for the full list.
