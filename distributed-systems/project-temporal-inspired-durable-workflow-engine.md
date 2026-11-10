---
layout: page
title: Project — Temporal-Inspired Durable Workflow Engine (Mini-Temporal)
permalink: /distributed-systems/project-temporal-inspired-durable-workflow-engine/
---

Implement a miniature durable workflow system inspired by Temporal’s core concepts: **event history**, **replay**, and **at-least-once activities with idempotent effects**.

You already built a JS DSL on Temporal; this project builds the durability/replay core that makes such DSLs possible.

## Learning goals

- Understand event sourcing for long-running orchestration.
- Implement replay-based determinism.
- Enforce idempotency at the side-effect boundary.

## Core concepts

### Event history

Persist events like:

- workflow started
- activity scheduled
- activity completed/failed
- timer fired
- signal received

Workflow state is derived by folding over events.

### Replay

On every workflow task:

1. Load history
2. Re-run the workflow logic from the beginning in a deterministic environment
3. Compare what would happen with the recorded history
4. Emit commands for what’s missing

### Activities

Activities are side effects:

- schedule + poll for completion
- must be idempotent (or deduped by activityId)

## Milestones

### M0 — Minimal state machine

- A workflow with: `start -> schedule activity -> complete`.
- Persist events in a local store (file or sqlite).
- Restart the engine mid-run and verify replay reconstructs the same pending work.

### M1 — Retries + idempotency keys

- Add retry policy for activities.
- Ensure activity side effects are not duplicated across retries.

### M2 — Timers

- Add a timer command.
- Persist timer fired events.

### M3 — Signals

- Add signal events that modify workflow input state.
- Ensure signal ordering is consistent with persisted history.

### M4 — continue-as-new (stretch)

- When history grows beyond a threshold, create a new run with carried state.

## Failure injection

Simulate:

- engine crash after scheduling activity but before recording completion
- engine crash after activity completion but before persisting event

**Invariant:** either the activity gets recorded completion, or it is safely re-run without double effects.

## Resources

- Temporal concepts: determinism sandbox, replay, versioning.

## Success criteria

- You can kill/restart the engine at arbitrary points and the workflow converges.
- A regression test asserts the workflow’s emitted command sequence is consistent for the same history.
