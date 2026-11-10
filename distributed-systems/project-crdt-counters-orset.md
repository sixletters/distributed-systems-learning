---
layout: page
title: Project — CRDT Counters & OR-Set
permalink: /distributed-systems/project-crdt-counters-orset/
---

Implement CRDTs to learn eventual consistency the *right* way: by designing data types whose state merges are associative, commutative, and idempotent.

## Learning goals

- Implement CRDT merge semantics.
- Build simulation of message reordering/duplication.
- Verify convergence.

## CRDTs to implement

Pick 2:

1. **G-Counter** (grow-only counter)
2. **PN-Counter** (positive/negative)
3. **OR-Set** (Observed-Remove Set)

## System design

### Local operations

- `increment(nodeId)` for counters
- `add(element, tag)` and `remove(element)` for OR-Set

### Merge operation

When two replicas exchange state:

- merge via pairwise max for counters
- merge via union of add-set and remove-set tags for OR-Set

## Fault injection

Simulate network as:

- messages can be dropped or duplicated
- delivery can be delayed arbitrarily
- replicas may update without coordination

## Metrics

- Convergence time (how many merges until replicas equal)
- Number of versions/tags stored (OR-Set can grow; add compaction strategy if you stretch)

## Milestones

### M0 — Single CRDT implementation + unit tests

- Verify merge properties (A merge B == B merge A, etc.).

### M1 — Multi-replica simulation

- Run N replicas with random operations.
- Periodically sync states (or messages).

### M2 — Convergence tests

Define and run:

- if you stop generating new updates, all replicas converge within bounded time.

### M3 — Stress OR-Set

- Many add/remove cycles for same elements.
- Observe tag growth; implement pruning via version vectors or bounded history (stretch).

## What to write

- Explain why CRDTs avoid conflicts instead of resolving them.
- Provide at least one scenario where conflict resolution would require application logic but CRDT avoids it.

## Success criteria

- Deterministic test suite proving convergence.
- Clear README with invariants and merge properties.
