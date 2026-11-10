---
layout: page
title: Project — Idempotent Effects Simulator
permalink: /distributed-systems/project-idempotent-effects-simulator/
---

Build a small service that processes “commands” (e.g., payments, orders, emails) under **at-least-once delivery** and proves your effect is **exactly-once in practice** (exactly-once *effects*, not delivery).

Why this matters: every retry-related outage turns into duplicated side effects unless you design idempotency at the boundary.

## Learning goals

- Understand at-least-once delivery semantics.
- Implement idempotency keys + atomic result recording.
- Build a fault-injection proxy that duplicates/drops requests.
- Prove with tests that duplicates never double-execute.

## System design

### Components

- **API server**: accepts `POST /command` with body `{commandId, type, payload}`.
- **Effect executor**: implements the side effect for each type.
- **Idempotency store**: records `{commandId -> (status, result, timestamps)}`.

### Idempotency contract

- If `commandId` is new: execute effect once, store result atomically, return it.
- If `commandId` already exists: return the stored result (even if the previous attempt failed transiently).

You want the idempotency record creation + effect result persistence to be transactional relative to each other.

## Fault injection harness

- Put a proxy in front of the API server.
- Randomly:
  - **duplicate** requests (same HTTP body) to simulate retries/delivery duplicates
  - **drop** responses (client retries because it didn’t see the ack)
  - **delay** responses (to scramble timing)

Use a fixed random seed so failures are reproducible.

## Milestones

### M0 — Single command type

- Implement one command, e.g. `payment`.
- Add idempotency keys.
- Run 10k duplicated requests from N clients concurrently.
- **Invariant:** effect counter == number of distinct commandIds.

### M1 — Multiple command types

- Add 2–3 command types with different payload shapes.
- Ensure each type has independent result serialization.

### M2 — Failure modes

Simulate:

- effect executor crashes after writing some external state (use a mock “external payment ledger” with controllable failure points)
- effect executor timeouts
- idempotency store transient failures

**Invariant:** no duplicate final effects.

### M3 — Property-based testing (optional but powerful)

Generate arbitrary sequences:

- concurrent duplicates
- dropping responses mid-flight
- random crash points

Then assert invariants.

## What to measure

- Latency overhead of idempotency lookup.
- Throughput under duplication rates.
- How long idempotency records accumulate (and when to evict with TTL).

## Resources

- Temporal concepts: **at-least-once activities + idempotent side effects**.
- Patterns: outbox, dedup windows, idempotency keys.

## Success criteria

- A test suite that deterministically proves: duplicates never double-execute effects.
- A short writeup mapping each failure injection to the invariant it protects.
