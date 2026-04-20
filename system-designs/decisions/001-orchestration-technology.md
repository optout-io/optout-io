# ADR-001: Orchestration Technology Choices

**Status:** Accepted  
**Date:** 2024 (initial design phase)

---

## Context

The core requirement is orchestrating data erasure across many data brokers. Each broker may require different steps, timings, and retry strategies. The system must be:

- **Resilient** — steps that fail must retry without losing state
- **Observable** — we need to know what happened, when, and why
- **Scalable** — hundreds of brokers, thousands of concurrent user requests
- **Extensible** — adding a new broker should not require rearchitecting the system

This requires decisions on: workflow management, async messaging, event storage, and read-side projections.

---

## Decisions

### Workflow Management → Temporal.io

**Options considered:**
- Build a custom step-execution engine on top of NATS/JetStream
- Use Temporal.io

**Decision:** Temporal.io.

**Reasoning:** Temporal provides durable, replay-safe workflow execution with built-in retry logic, timeouts, activity heartbeating, and a UI for observability. Building equivalent functionality on NATS would mean reinventing Temporal's core guarantees. The combination of NATS + event sourcing would get us "halfway there" — Temporal gets us all the way.

---

### Async Messaging → NATS + JetStream

**Options considered:**
- NATS core (pub/sub only)
- JetStream (persistent, durable queues)
- NATS KV/Object Store for event sourcing

**Decision:** JetStream for all async domain events. NATS core for ephemeral signals if needed.

**Reasoning:** JetStream provides durable, ordered, consumer-group messaging with at-least-once delivery — sufficient for our event-driven needs. Using NATS KV as an event sourcing store was considered but rejected to avoid over-bundling with Synadia and to keep event storage concerns separate.

---

### Event Storage → PostgreSQL (via lake service)

**Options considered:**
- Third-party event sourcing service
- `eventsourcing-lite` (internal library)
- Custom append-only PostgreSQL tables

**Decision:** Append-only PostgreSQL tables, owned by a dedicated `lake` service.

**Reasoning:** A purpose-built event lake service (`lake`) consuming from JetStream and writing to PostgreSQL is simple, auditable, and avoids third-party lock-in. The `eventsourcing-lite` library was considered but a lightweight dedicated service gives better separation of concerns.

---

### Read-Side Projections → CQRS, in-house

**Decision:** Each domain service maintains its own projection DB (PostgreSQL). No shared read DB across domains.

**Reasoning:** CQRS is necessary for client-facing views. Building this in-house using events from NATS is straightforward. Each domain owns its own projection — this enforces domain boundaries and avoids cross-domain coupling.

---

## Consequences

- Services are decoupled via JetStream — no direct gRPC calls for async flows
- Temporal handles all retry/timeout/resilience logic — services do not implement their own
- Adding a new broker requires a new Temporal sub-workflow (or YAML config for webform brokers) — no changes to core orchestration
- The `lake` service is the authoritative event history for auditing and admin observability
