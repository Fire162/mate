---
title: "Tracing"
summary: Turn on OpenTelemetry tracing, view traces in the dashboard and export them.
audience: dev
order: 80
covers:
  - shared/utils/tracing/**
---

# OpenTelemetry Distributed Tracing

MATE uses OpenTelemetry for distributed tracing of agent executions, LLM calls, tool invocations, RBAC checks, and memory operations.

## Setup

### Environment Variables

```env
# Enable tracing (default: false)
OTEL_TRACING_ENABLED=true

# Optional: OTLP endpoint for Jaeger, Grafana Tempo, Datadog, Honeycomb
# When unset, only DB export runs (for dashboard)
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf

# Service name for traces
OTEL_SERVICE_NAME=mate

# Store spans in database for dashboard viewer (default: true when tracing enabled)
OTEL_TRACES_DB_EXPORT=true
```

### Zero Performance Overhead When Disabled

When `OTEL_TRACING_ENABLED=false` (default), tracing is completely disabled. Tracers and wrappers short-circuit without creating spans, incurring zero runtime overhead.

## Architecture and Exporters

- **TracerProvider integration**: When ADK has already set an SDK `TracerProvider`, MATE attaches its `BatchSpanProcessor(DatabaseSpanExporter())` directly to that provider. If no SDK provider exists, MATE creates an SDK `TracerProvider` with the configured resource name.
- **Database Span Exporter**: When `OTEL_TRACES_DB_EXPORT=true`, spans are batched (1-second flush delay) and persisted to the `trace_spans` table for visualization in the dashboard.
- **OTLP Exporter**: When `OTEL_EXPORTER_OTLP_ENDPOINT` is configured, spans are exported via HTTP/protobuf or gRPC to OTLP collectors.

## Instrumentation

| Instrumentation Point | Span Name | Attributes |
|---|---|---|
| LLM inference | `gen_ai.inference` | `gen_ai.operation.name`, `gen_ai.provider.name`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens` |
| RBAC check | `mate.rbac_check` | `mate.agent.name`, `mate.user.id`, `mate.rbac.allowed` |
| Tool invocation | `mate.tool` | `mate.tool.name`, `mate.agent.name` |
| Session memory | `mate.memory` | `mate.memory.operation` (add/search) |
| Memory blocks | `mate.memory_blocks` | `mate.memory.operation` |

## Dashboard

The `/dashboard/traces` page shows traces from the `trace_spans` table when `OTEL_TRACES_DB_EXPORT=true`. Traces are grouped by trace ID with a call graph showing span hierarchy and latency.

## OTLP Export

To export to Jaeger, Grafana Tempo, Datadog, or Honeycomb:

1. Run an OTLP-compatible collector (e.g. Jaeger all-in-one, Grafana OTLP collector).
2. Set `OTEL_EXPORTER_OTLP_ENDPOINT` to the collector URL (default: `http://localhost:4318`).
3. Set `OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf` (or `grpc`).

## W3C Trace Context

Trace context is propagated via standard `traceparent` and `tracestate` headers. Inbound and proxy requests preserve and inject trace context when tracing is enabled.

