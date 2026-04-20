# ADR-002: V1 System Design

**Status:** Superseded by current architecture (see ARCHITECTURE.md)  
**Date:** 2024 (initial design phase)

> Kept for historical reference. The V1 design established the core event-sourcing and orchestration patterns that carried forward into V2.

---

## Flow Chart

```mermaid
graph TD
    subgraph Messaging System
        C[NATS + JetStream]
    end
    subgraph Persistent Data
        J[(Event Store DB)]
        D[(Query Side DB)]
    end

    A{API Client} -->|Start Opt Out| B[Orchestration Service]
    B --->|Sync: Persist Event| J
    B <-->|Post Step that Needs to be Done| C
    C -.->|Async Notifications| D
    C -.->|Consume Step to be Done| E[Step Processor]
    E -->|Sync: Step Started| J
    E -->|Sync: Step Completed| J
    E --->|Publish Step Completed| C
    B --->|Determine Next Step| J
    H[User Dashboard] -->|Query Current State| A
    A -->|Query Current State| D
    I[Admin Panel] -->|Query Event History| J
    I -->|Trigger Step| B
```

---

## Sequence Diagram — Opt Out

```mermaid
sequenceDiagram
    participant APIClient as API Client
    participant Orchestrator as Orchestration Service
    participant EventStore as Event Store DB
    participant NATS as NATS + JetStream
    participant StepProcessor as Step Processor

    APIClient ->> Orchestrator: Start Opt Out
    Orchestrator ->> EventStore: Persist "Step Determined" event
    Orchestrator ->> NATS: Publish "Step Determined"
    NATS ->> StepProcessor: Consume "Step Determined"
    StepProcessor ->> Orchestrator: Notify "Step Started"
    Orchestrator ->> EventStore: Persist "Step Started" event
    StepProcessor ->> StepProcessor: Execute Step Logic
    StepProcessor ->> Orchestrator: Notify "Step Completed"
    Orchestrator ->> EventStore: Persist "Step Completed" event
    Orchestrator ->> Orchestrator: Evaluate Next Step
    Orchestrator ->> EventStore: Persist "Next Step Determined" event
    Orchestrator ->> NATS: Publish "Next Step"
```

---

## Sequence Diagram — Customer Service Agent (Manual Trigger)

```mermaid
sequenceDiagram
    participant Admin
    participant Orchestrator
    participant EventStore
    participant StepProcessor
    Admin ->> Orchestrator: Request Trigger Step
    Orchestrator ->> EventStore: Validate StepRequest
    EventStore -->> Orchestrator: Validation Result
    alt If Valid
        Orchestrator ->> StepProcessor: Command: Execute Step
        StepProcessor ->> EventStore: Emit StepStarted
        EventStore -->> StepProcessor: Acknowledge Event
    else If Invalid
        Orchestrator -->> Admin: Reject Request
    end
```

---

## Key Differences from Current Architecture

| V1 Concept | Current Equivalent |
|---|---|
| Orchestration Service | `data-erasure-wf` service |
| Step Processor | Temporal workers + `webform-playwright` |
| Event Store DB | `lake` service + PostgreSQL |
| Query Side DB | Per-domain projection DBs in each service |
| Admin Panel | `abscond` |
