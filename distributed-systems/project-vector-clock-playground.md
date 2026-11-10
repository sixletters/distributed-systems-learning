---
layout: page
title: Project — Vector Clock Playground & Conflict Resolver
permalink: /distributed-systems/project-vector-clock-playground/
---

A toy distributed KV store that tracks causality with **vector clocks**, detects concurrent writes, and forces you to implement a conflict-resolution strategy.

## Learning goals

- Build and validate vector clock updates.
- Detect concurrency: "happens-before" vs "concurrent".
- Store multiple versions and reconcile.

## System design

### Data model

For each key, store:

- value (or per-version value)
- vector clock (per version)
- metadata for causality (who produced it)

### Client behavior

Clients read a key and receive one or more versions.

Clients write by:

- choosing a base version (one of the received clocks)
- incrementing their own component
- producing a new version with the updated clock

### Reconciliation rule (you choose)

Implement one of:

- return all concurrent versions
- deterministic merge function (domain-specific)
- last-write-wins *as a last resort* (but you must document that it violates causality)

## Fault injection

- Partition the network into two halves.
- Allow clients in both halves to update the same key concurrently.
- Heal the partition.

**Invariant:** your system must detect concurrent writes and never claim an ordered history when writes were actually concurrent.

## Milestones

### M0 — Vector clock library

- Implement: `increment`, `merge`, `compare(happens-before / concurrent / after)`.
- Unit tests for clock comparisons.

### M1 — Two-node KV store

- Implement versioned storage.
- Partition simulation.

### M2 — Multi-version reconciliation UI

- Provide a way to list concurrent versions.
- Provide a merge button (or a merge strategy in code).

### M3 — Realistic workload (optional)

Generate a workload with:

- hot keys
- bursty clients
- partial replication delays

## What to write

- A section: “When do I get multiple versions?”
- A section: “What merge strategy did I choose and why?”
- Examples of clocks where events are concurrent.

## Success criteria

- Partition test proves concurrency is detected correctly.
- Reconciliation strategy is deterministic and documented.
