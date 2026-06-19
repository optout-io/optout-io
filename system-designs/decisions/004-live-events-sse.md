# ADR-004: Live UI Updates — NATS → SSE → Datastar

**Status:** Accepted  
**Date:** 2026-06-19

---

## Context

The admin dashboard needs real-time visibility into workflow and activity progress without page reloads. Activities complete in near-real-time (seconds) and admins need to see status changes as they happen — particularly for the email confirmation signal flow.

---

## Decisions

### 1. Write path: Temporal interceptors → NATS core

`WorkflowReportingInterceptor` and `ActivityReportingInterceptor` (in `data-erasure-wf/internal/dataerasure/interceptors/`) wrap every workflow and activity execution. On lifecycle events (RUNNING, STARTED, COMPLETED, FAILED), they call `EventPublisher.PublishWorkflowStatus/ActivityStatus()` which publishes a serialised `EventEnvelope` (proto) to the JetStream subject `com.optmeout.dataerasure.workflow.progress`.

Core NATS subscribers on the same subject receive JetStream-published messages at publish time (JetStream is layered over core NATS).

### 2. Abscond subscribes via core NATS

`natssubscriber.Subscriber` opens a persistent core NATS subscription to `com.optmeout.dataerasure.workflow.progress`. This runs in a goroutine with `context.Background()` (never cancelled) for the lifetime of the process.

### 3. SSE connection initiated by Datastar data-effect

The event table `<tbody>` uses `data-effect="@get('/admin/stream/events')"`. A `data-effect` with no signal reads fires exactly once on Datastar init — this establishes the persistent SSE connection. (`data-on-load` and `data-on:load` do not fire on server-rendered elements in Datastar v1.0.0-RC.6.)

### 4. SSE broadcasts to all clients

`ClientManager` (a `sync.Map[channel → struct{}]`) holds one buffered channel per connected SSE client. `Notify()` fans out to all channels with a non-blocking send (drops if full). This is intentional — live events are best-effort; the read path (lake gRPC) is the source of truth.

Each SSE event sends two patches:
- `PatchSignals({live_event_ts, live_event_urn})` — triggers Datastar effects
- `PatchElementTempl(ActivityTimelineEntry, "#timeline-items", prepend)` — pushes a timeline row directly from the write path without querying lake

### 5. Timeline updates are push-only (write path)

The activity timeline in the drawer receives new entries directly from the SSE stream (`#timeline-items` prepend). No polling. The initial timeline on drawer open comes from lake (read path). This avoids the race condition where SSE fires before lake's pull consumer has ingested the event.

Lake's pull consumer interval is 200ms to minimise the window where the initial drawer load might be stale relative to in-flight events.

### 6. After email signal, navigate to parent drawer

When the email confirmation signal is sent from a child workflow drawer, the POST response patches `$drawer_open` to the parent workflow's URN. This ensures subsequent live events (children completing, parent completing) update the parent drawer rather than stale child data.

---

## Future work: SSE subscription router

The current `ClientManager` broadcasts all events to all clients. A proper subscription router would:
- Accept subscription criteria at SSE connection time (workflow URN, user URN, event type)
- Authenticate and authorise the subscription
- Route only matching events to each client

Design doc: `system-designs/designs/sse-router/` (to be written).

---

## Consequences

- Live updates work for workflow start, all activity lifecycle events, and workflow completion
- Timeline in the drawer is near-real-time (< 200ms from activity fire to UI update)
- All SSE clients receive all events — filtering is client-side (timeline shows wrong-workflow events briefly until drawer navigates)
- The SSE router is the right long-term fix for client-side filtering
