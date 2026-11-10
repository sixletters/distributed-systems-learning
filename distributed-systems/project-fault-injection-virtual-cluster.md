---
layout: page
title: Project — Fault Injection Virtual Cluster Harness
permalink: /distributed-systems/project-fault-injection-virtual-cluster/
---

A reusable harness to run distributed-systems experiments locally with controllable failures.

If you build this once, you can reuse it to test Raft, CRDTs, idempotency, and more.

## Learning goals

- Build a simulated network with drop/delay/reorder/duplication.
- Build node lifecycle: start, stop, restart.
- Provide deterministic scenario replay via seeds.

## Architecture

### In-process simulator

Run N “nodes” as actors/threads inside one process.

### Network model

Represent the network as a component that mediates message delivery:

- on send: decide delay, drop, and duplication
- maintain a priority queue of scheduled deliveries
- deliver messages in simulation time order

## Fault primitives

Support:

- message drop
- message delay distribution
- message duplication
- partitions (split nodes into groups that can’t exchange)
- node pause (simulate long GC) and kill (crash)

## Determinism

- Everything must be driven by a single PRNG seed.
- Record the seed and scenario config so failures are reproducible.

## Milestones

### M0 — Message drop + delay

- Build send/receive API.
- Write a tiny ping test.

### M1 — Duplication + reorder

- Verify invariants on a toy reliable/unreliable protocol.

### M2 — Kill + restart with persisted state

- Let nodes persist some local disk state to simulate WAL/replay.

### M3 — Partition

- Assert that after heal, the system returns to a consistent state.

### M4 — Build a small “test DSL”

Example scenario:

- at time 200ms, partition leader from follower
- at 400ms, kill a node
- at 800ms, heal

## What this unlocks

- Jepsen-style linearizability testing.
- Chaos testing in CI.
- Reliable regression tests for your distributed algorithms.

## Success criteria

- A scenario that fails deterministically 10/10 times.
- A documentation section: “How to write new tests with this harness.”
