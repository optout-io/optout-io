# Design: Live Workflow Events in Admin UI

**Status:** Implemented — pending end-to-end validation  
**Branch:** `abscond/events-from-backend`, `data-erasure-wf/interceptor-telemetry`  
**Date:** 2026-04-27

---

## Goal

Real-time workflow status updates in the admin dashboard without page reloads. When a workflow transitions state (RUNNING → COMPLETED, activity fails, etc.), the admin table and open drawer should reflect the change immediately.

This is also an experiment with [Datastar](https://data-star.dev/) (hypermedia + SSE) as a React alternative for admin tooling.

---

## Signal Flow

```
data-erasure-wf Worker
  → Temporal interceptors fire on workflow/activity lifecycle
  → EventPublisher.PublishWorkflowStatus() / PublishActivityStatus()
  → NATS core subject: com.optmeout.dataerasure.workflow.progress
      (NOTE: NATS core, not JetStream — ephemeral, no durability guarantee)

abscond (admin app)
  → natssubscriber.Subscriber.Subscribe() goroutine
  → ClientManager.Notify(LiveWorkflowEvent)
  → Broadcast to all SSE clients via typed chan LiveWorkflowEvent

Browser (Datastar)
  → SSE connection opened by EventTable component (data-on-load="@get('/admin/stream/events')")
  → Server sends PatchSignals: { live_event_ts, live_event_urn }
  → Server sends PatchElement: new EventRow appended to event table
  → data-effect on workflow table re-triggers GET /admin/workflows/fragment
  → data-effect on drawer re-triggers GET /admin/drawer (if open)
```

---

## Key Design Decisions

### NATS core (not JetStream) for live events
Events published to `com.optmeout.dataerasure.workflow.progress` via plain NATS pub/sub. JetStream durability is not needed here — abscond only cares about events that arrive while it's connected. Historical/audit events already go to lake via JetStream. This keeps the SSE path simple and low-latency.

### Datastar signals as the reactive bus
Instead of polling or websocket push, the browser maintains a reactive signal store (`$live_event_ts`, `$live_event_urn`). The server updates these signals via SSE `PatchSignals`. Components re-render by reacting to signal changes via `data-effect` — the server drives all re-renders, no client-side state management.

### Table re-renders from server on every event
`data-effect="if ($live_event_ts) { @get('/admin/workflows/fragment') }"` — the workflow table refetches from the server whenever `$live_event_ts` changes. This keeps the rendering logic server-side and means filters are always respected.

### Drawer refreshes without closing
The drawer effect: `if ($drawer_open) { const _ = $live_event_ts; @get('/admin/drawer') }` — subscribes to `$live_event_ts` as a dependency so it also re-fetches on any live event. The `const _ = $live_event_ts` is a Datastar trick to force the dependency registration without using the value directly.

---

## Components Affected

| Repo | File | Change |
|---|---|---|
| `data-erasure-wf` | `internal/dataerasure/interceptors/workflow_reporting.go` | Publishes RUNNING/COMPLETED/FAILED to NATS core |
| `data-erasure-wf` | `internal/dataerasure/interceptors/activity_reporting.go` | Publishes STARTED/COMPLETED/FAILED to NATS core |
| `data-erasure-wf` | `internal/publisher/events.go` | EventPublisher (Layer 1) — direct JetStream + NATS core |
| `abscond` | `internal/natssubscriber/subscriber.go` | NATS core subscriber, parses EventEnvelope |
| `abscond` | `apps/adminapp/internal/routes/clientmanager.go` | Upgraded to typed `chan LiveWorkflowEvent` |
| `abscond` | `apps/adminapp/internal/routes/app.go` | NATS goroutine, SSE PatchSignals handler |
| `abscond` | `apps/adminapp/templates/pages/index.templ` | Added `live_event_ts`/`live_event_urn` signals + data-effects |
| `abscond` | `apps/adminapp/templates/components/events.templ` | EventTable opens SSE, EventRow rendered per event |

---

## NATS Subject

```
com.optmeout.dataerasure.workflow.progress
```

Payload: protobuf `EventEnvelope` wrapping either `WorkflowStatusChanged` or `ActivityStatusChanged`.

---

## Bug Fixed During Implementation

`SaveWorkflowProjection` activity was failing with `invalid UUID length: 66` because `workflowrow.go` called `uuid.Parse(input.UserID)` on a full URN string (`urn:optmeout:principal/user:{uuid}`). Fixed in `data-erasure-wf/internal/database/workflowdb/workflowrow.go` by adding a `parseUserID()` helper that accepts both bare UUID and URN formats.

**Commit:** `dd694a0` on `interceptor-telemetry` branch.

---

## Validation Steps (post-reboot)

1. Start infra: `make docker-compose` in `data-erasure-wf/`
2. Start Temporal (separate docker-compose repo)
3. Start NATS monitor: `nats sub "com.optmeout.dataerasure.workflow.progress"`
4. `make compile && make start-worker` in `data-erasure-wf/`
5. `make start-service` in `data-erasure-wf/`
6. Start admin app: `make dev` in `abscond/apps/adminapp/`
7. Open `http://localhost:9140/admin` in browser
8. `make publish-test-event` in `data-erasure-wf/`
9. Verify: worker logs show no UUID error, NATS monitor shows event, browser table updates without page reload

---

## Open Items

- [ ] End-to-end validation not yet confirmed (pending reboot)
- [ ] Lake not running during initial test — timeline will be empty until lake restarts and drains JetStream queue
- [ ] ARCHITECTURE.md needs update: abscond now subscribes to NATS core (structural change)
- [ ] Admin workflow controls (cancel/terminate/signal) implemented in `data-erasure-wf` gRPC but UI buttons in drawer not yet wired to POST handlers
