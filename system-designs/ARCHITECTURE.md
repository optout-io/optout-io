# optout-io System Architecture

> **Living document.** Updated as the system evolves. For *why* decisions were made, see `decisions/`. For *what* is being built next, see `designs/`.

---

## Purpose

optout-io automates privacy opt-out requests (GDPR, CCPA, DPPA) on behalf of users across hundreds of data brokers. The system submits, tracks, and verifies erasure requests through a combination of API calls and browser automation.

---

## Component Map

```mermaid
graph TD
    subgraph Applications
        AU[User App]
        AA[Admin App / abscond]
    end

    subgraph Orchestration
        subgraph Onboarding Domain
            OS[Onboarding Service]
            OW[Onboarding Workers]
            ODB{{Onboarding Projection DB}}
        end
        subgraph Data Erasure Domain
            DES[data-erasure-wf Service]
            DEW[data-erasure-wf Workers]
            DEDB{{DataErasure Projection DB}}
        end
        TMP[[ Temporal Cluster ]]
    end

    subgraph Infrastructure
        subgraph Messaging
            CN[[ NATS ]]
            CJ[[ JetStream ]]
        end
        subgraph EventLake
            ELS[lake Service]
            ELDB{{Event Lake DB - PostgreSQL}}
        end
    end

    subgraph BrokerAutomation
        WFP[webform-playwright Workers]
    end

    AU -->|Start User Onboarding| OS
    OS <-->|Onboarding Workflow| TMP
    TMP <--> OW
    OS --> ODB
    OW -->|Read| ODB
    OS -.->|UserOnboardingCompleted| CJ

    CJ -.->|UserOnboardingCompleted| DES
    DES <-->|DataErasure Workflow| TMP
    TMP <--> DEW
    DES --> DEDB
    DEW -->|Read| DEDB
    DEW -.->|Broker Automation Tasks| CJ

    CJ -.->|Automation Tasks| WFP
    WFP -.->|Task Results| CJ

    DES -.->|Workflow/Activity Events| CJ
    CJ -.->|All Events| ELS
    ELS --> ELDB

    DES -.->|Workflow/Activity Events - NATS core| CN
    CN -.->|Live Events SSE| AA

    AU -->|Query status| DES
    AA -->|Query workflows| DES
    AA -->|Query event timeline| ELS
```

---

## Components

### data-erasure-wf
**Repo:** `data-erasure-wf/`  
**Role:** Core orchestration service. Owns the data erasure domain.

- Consumes `UserOnboardingCompleted` from NATS/JetStream to start erasure workflows
- Runs Temporal workflows (`DataErasureWorkflow`) that coordinate erasure across all relevant brokers
- Broker-specific sub-workflows (e.g., Acxiom, DeepSync) run as child workflows
- Publishes workflow/activity telemetry events to JetStream for the lake
- Exposes gRPC API for querying workflow state
- Stack: Go, Temporal v1.33+, NATS/JetStream, PostgreSQL

### lake
**Repo:** `lake/`  
**Role:** Append-only event store. Consumes all domain events from JetStream and persists them.

- Ingests workflow and activity lifecycle events
- Handles Temporal replay deduplication (idempotent inserts)
- Exposes gRPC query API: fetch events by aggregate ID (used by admin timeline)
- Stack: Go, NATS/JetStream, PostgreSQL, golang-migrate

### abscond
**Repo:** `abscond/`  
**Role:** Internal admin dashboard. Real-time observability over workflows.

- SSE-based real-time event streaming (Datastar hypermedia framework)
- Subscribes to NATS core (`com.optmeout.dataerasure.workflow.progress`) for live workflow events; broadcasts via SSE `PatchSignals` which triggers Datastar effects to refetch the workflow table and drawer
- Activity timeline entries pushed directly from the SSE write path into `#timeline-items` (prepend) — no polling, no lake round-trip for live updates
- Workflow drawer: per-workflow activity status timeline, Temporal deep link, workflow control actions (cancel/terminate/signal)
- Calls `data-erasure-wf` gRPC for workflow list/detail; calls `lake` gRPC for historical event timeline
- Stack: Go, Chi, Templ, Datastar v1.0.0-RC.6, DaisyUI v5 + Tailwind CSS v4 (standalone build, no Node.js), gRPC, NATS core, OpenTelemetry

### webform-playwright
**Repo:** `webform-playwright/`  
**Role:** Browser automation worker. Executes opt-out form submissions at data brokers.

- Config-driven: brokers defined in YAML with Mustache templating — no new code per broker
- Playwright handles DOM interaction, screenshot capture, form submission
- Runs as a Temporal worker, receives tasks from `data-erasure-wf`
- Stack: TypeScript, Playwright v1.53+, Node 22, Temporal worker

### contracts
**Repo:** `contracts/`  
**Role:** Shared protobuf schema definitions. The contract layer between all services.

- Published as both Go and TypeScript packages
- Covers: workflow telemetry events, URN definitions, lake gRPC service API
- Strict backward compatibility — never break existing fields
- Stack: Protocol Buffers, buf, GitHub Actions for codegen

### go-common
**Repo:** `go-common/`  
**Role:** Shared Go infrastructure library. Domain-agnostic — no business logic, no domain imports.

- `pkg/logger` — structured Zap logging; `WithContext(ctx, log)` injects OTel trace/span IDs
- `pkg/otel` — `NewOtelProvider(serviceName string)` initialises trace + metrics providers. Trace export gated on `OTEL_EXPORTER_OTLP_ENDPOINT`; metrics export gated separately on `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT`
- `pkg/workflowlogger` — Temporal-aware logger adapter

### Onboarding Service
**Repo:** not yet in this workspace (separate service)  
**Role:** Handles user signup/onboarding workflow. Publishes `UserOnboardingCompleted` which triggers data erasure.

---

## Cross-Cutting Concerns

### Identifiers — URNs
All entities (users, workflows, brokers, activities) are identified by URNs. Format defined in `contracts/`. Utilities in `go-common/`.

### Messaging — NATS/JetStream
All async communication is via JetStream. Events are durable, ordered per-subject. Services never call each other directly for async operations.

### Workflow Orchestration — Temporal
All multi-step, stateful processes run as Temporal workflows. Temporal provides retry logic, timeouts, visibility, and replay-safe execution. Workers are separate binaries from services.

### Observability
- Structured logging via `go-common/pkg/logger`. `logger.WithContext(ctx, log)` injects `trace_id`/`span_id` from the active OTel span.
- **Distributed tracing** — OpenTelemetry SDK in all Go services. Traces exported to Jaeger (local: `http://localhost:16686`) via `OTEL_EXPORTER_OTLP_ENDPOINT`. Metrics intentionally separate — only exported when `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` is also set (Jaeger is traces-only).
- **gRPC** — `otelgrpc.NewClientHandler()` on all gRPC clients; `otelgrpc.NewServerHandler()` on all gRPC servers. Trace context propagates across the HTTP → gRPC boundary automatically.
- **Temporal** — `opentelemetry.NewTracingInterceptor()` registered on the data-erasure-wf worker. Auto-spans for every workflow and activity execution.
- **NATS** — W3C trace context injected into JetStream message headers on publish (`PublishDataErasureInitiated`); extracted by the lake event ingestor to continue the trace across the async boundary. NATS server itself does not export OTel spans.
- **Event timeline** — lake stores all workflow/activity events; abscond queries them via gRPC for the per-workflow drawer timeline.
- **Temporal UI** — deep links from abscond drawer to Temporal UI for workflow-level debugging (search by workflow ID).

### Schema Evolution
All inter-service contracts live in `contracts/`. Changes require backward-compatible protobuf evolution. Breaking changes require a new message version.

---

## Key Invariants

1. **Domain-owned storage**: each domain service owns its own projection DB. No cross-domain DB reads.
2. **lake is append-only**: events are never updated or deleted. Deduplication is at ingestion.
3. **Temporal workers are stateless**: all durable state lives in Temporal + the projection DB.
4. **Broker automation is config-driven** *(aspiration)*: the webform layer is fully config-driven (YAML per broker in `webform-playwright`). The orchestration layer (`data-erasure-wf`) still requires a small Go child workflow file per broker — eliminating this is a roadmap item (see `ROADMAP.md`, initiative #1).
5. **contracts defines the API surface**: services communicate via protobuf contracts, not shared Go structs.
