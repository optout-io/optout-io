# ADR-003: Observability — OpenTelemetry and Jaeger

**Status:** Accepted  
**Date:** 2026-06-19

---

## Context

The system spans multiple services (data-erasure-wf, lake, abscond), a Temporal worker, NATS/JetStream messaging, and gRPC connections. We needed distributed tracing to understand request flows, debug failures, and validate that cross-service calls propagate correctly.

---

## Decisions

### 1. OpenTelemetry SDK via `go-common/pkg/otel`

All Go services initialise an OTel provider from `go-common/pkg/otel.NewOtelProvider(serviceName string)`. The service name comes from the app config (`conf.Variables.ServiceName`) — never hardcoded.

### 2. Trace and metrics exporters are gated independently

- `OTEL_EXPORTER_OTLP_ENDPOINT` enables **trace** export (set in each service's `.air.toml` `full_bin`)
- `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` enables **metrics** export (separate, because Jaeger only accepts traces)

This split prevents the metrics periodic reader from spamming errors when the OTLP backend is a trace-only collector (Jaeger).

### 3. Jaeger all-in-one for local development

Jaeger is included in `data-erasure-wf/local/docker-compose.yaml` and `lake/local/docker-compose.yaml`. Services export to `http://localhost:4317` (OTLP gRPC). The env var is injected via `full_bin` in each Air config, so it is a dev-only concern and does not appear in Makefiles.

### 4. gRPC tracing via `otelgrpc`

Both gRPC clients (abscond → data-erasure-wf, abscond → lake) use `grpc.WithStatsHandler(otelgrpc.NewClientHandler())`. Both gRPC servers use `grpc.StatsHandler(otelgrpc.NewServerHandler())`. This propagates W3C trace context on all gRPC calls.

### 5. NATS trace context propagation

The `EventPublisher.PublishDataErasureInitiated` activity injects OTel trace context into NATS message headers using a `natsHeaderCarrier` adapter. Lake's `EventIngestor.ProcessMessage` extracts it and starts a child span. This links the Temporal activity span to the lake ingestion span in a single trace.

The `EventPublisher.PublishWorkflowStatus/ActivityStatus` (used by interceptors) does **not** propagate context — those are fire-and-forget with no caller context available.

### 6. Temporal worker OTel interceptor

`go.temporal.io/sdk/contrib/opentelemetry.NewTracingInterceptor` is registered on the worker. It automatically creates spans for every workflow and activity execution. Temporal's own UI handles workflow-level correlation via workflow ID — OTel traces cover the intra-worker span tree.

### 7. logger.WithContext

`go-common/pkg/logger.WithContext(ctx, log)` returns a logger with `trace_id` and `span_id` fields injected from the active OTel span. Callers opt in explicitly; the base logger is unmodified.

---

## Consequences

- Full HTTP → gRPC → Temporal activity → NATS → lake trace chains are visible in Jaeger
- Metrics are not yet shipped anywhere; the separate endpoint gate means this can be added without touching trace config
- The Temporal worker's internal replays are handled correctly (interceptor only creates spans on first execution, not on replay — this is Temporal SDK behaviour)
- NATS server itself does not export OTel spans; W3C trace context is propagated at the client (Go SDK) level only
