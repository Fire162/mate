---
title: "Tracing"
summary: Turn on OpenTelemetry tracing, view traces in the dashboard and export them.
audience: dev
order: 80
covers:
  - shared/utils/tracing/**
  - server/proxy_routes.py
---

# OpenTelemetry Distributed Tracing

MATE uses OpenTelemetry for distributed tracing of agent executions, LLM calls, tool invocations, RBAC checks, and memory operations.

## Setup

### Environment Variables

```env
# Enable tracing (default: false)
OTEL_TRACING_ENABLED=true

# Optional: OTLP endpoint for Jaeger, Grafana Tempo, Datadog, Honeycomb (OTLP over HTTP)
# There is no default: when unset, spans only go to the database
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318

# Service name for traces (only applies when MATE initializes its own TracerProvider)
OTEL_SERVICE_NAME=mate

# Store spans in database for dashboard viewer (default: true when tracing enabled)
OTEL_TRACES_DB_EXPORT=true
```

### Performance When Disabled

When `OTEL_TRACING_ENABLED=false` (default), tracing is disabled. Tool wrapping is skipped, but calls via `trace_context` still route through a no-op tracer.

## Architecture & Tracer Providers

- **TracerProvider integration**: When ADK has already initialized an SDK `TracerProvider`, MATE adds only its `BatchSpanProcessor(DatabaseSpanExporter())` to it and returns. In this case, MATE's OTLP exporter and `OTEL_SERVICE_NAME` configuration do not apply.
- **Standalone TracerProvider**: When no SDK provider is found, MATE initializes its own `TracerProvider` with `service.name` set to `OTEL_SERVICE_NAME`.
- **Database Span Exporter**: When `OTEL_TRACES_DB_EXPORT=true`, spans are batched (1-second flush delay) and persisted to the `trace_spans` table for visualization in the dashboard.
- **OTLP Exporter**: When `OTEL_EXPORTER_OTLP_ENDPOINT` is explicitly set, spans are exported using OTLP over HTTP (`opentelemetry.exporter.otlp.proto.http`). gRPC is not supported, and `OTEL_EXPORTER_OTLP_PROTOCOL` is not read.

## Instrumentation

| Instrumentation Point | Span Name | Attributes |
|---|---|---|
| LLM inference | `gen_ai.inference` | `gen_ai.operation.name`, `gen_ai.provider.name`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens` |
| RBAC check | `mate.rbac_check` | `mate.agent.name`, `mate.user.id`, `mate.rbac.allowed` |
| Tool invocation | `mate.tool` | `mate.tool.name`, `mate.agent.name` |
| Session memory | `mate.memory` | `mate.memory.operation` (add/search) |
| Memory blocks | `mate.memory_blocks` | `mate.memory.operation` |

## Dashboard

The `/dashboard/traces` page shows traces from the database when `OTEL_TRACES_DB_EXPORT=true`. Traces are grouped by trace ID with a call graph showing span hierarchy and latency.

## OTLP Export

To export to Jaeger, Grafana Tempo, Datadog, or Honeycomb:

1. Run an OTLP-compatible collector (e.g. Jaeger all-in-one, Grafana OTLP collector).
2. Set `OTEL_EXPORTER_OTLP_ENDPOINT` to the collector's OTLP HTTP URL (for example `http://localhost:4318`). There is no default: without this variable, spans only go to the database. Only OTLP over HTTP is supported (`opentelemetry.exporter.otlp.proto.http`). gRPC is not supported, and `OTEL_EXPORTER_OTLP_PROTOCOL` is ignored.

## W3C Trace Context

Trace context is injected by the auth server proxy (`_inject_trace_headers` in `server/proxy_routes.py`) when forwarding requests to ADK if tracing is enabled. Nothing extracts trace context from inbound external requests.
