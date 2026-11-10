---
layout: page
title: Distributed Systems
permalink: /distributed-systems/
---

This is the systems spine of the repo. Learn the topics below, then build the projects that force those topics to become real. The point is not to memorize vocabulary — it is to be able to reason about split brain, duplicated messages, clock skew, replay, and consistency under failure.

## Topic map

| Topic | What you should be able to explain |
|-------|------------------------------------|
| Messaging semantics | at-most-once vs at-least-once vs exactly-once effects, retries, idempotency keys, dedupe stores |
| Time and causality | wall vs monotonic clocks, happens-before, Lamport clocks, vector clocks, hybrid logical clocks |
| Failure detection | heartbeats, suspicion, timeouts, leases, fencing tokens, split brain |
| Consistency and CAP | linearizability, sequential, causal, eventual; CP vs AP under partitions |
| Replication | leader/follower, quorum writes, anti-entropy, read repair, multi-leader conflicts |
| Consensus | leader election, majority commit, term changes, log safety, Raft |
| Sharding | consistent hashing, virtual nodes, hot keys, membership changes, rebalancing |
| Distributed transactions | 2PC, sagas, outbox pattern, compensating actions |
| Replay and durability | WAL, event sourcing, deterministic replay, versioning |
| Observability | metrics, logs, traces, SLOs, p99 latency, chaos testing |

If you can explain why exactly-once delivery is impossible but exactly-once *effects* are achievable, you are already thinking in the right model.

## Project ladder

These are the projects that turn the topics above into muscle memory.

1. [Fault Injection Virtual Cluster Harness](project-fault-injection-virtual-cluster/) — learn drop, delay, reorder, duplication, partitions, and deterministic replay.
2. [Vector Clock Playground & Conflict Resolver](project-vector-clock-playground/) — learn causality, concurrent writes, and merge strategies.
3. [Replicated Log Mini-System](project-replicated-log-mini-system/) — learn majority commit, leader changes, and safety invariants.
4. [Consistent Hashing KV with Live Rebalancing](project-consistent-hashing-kv/) — learn sharding, membership changes, and key movement.
5. [CRDT Counters & OR-Set](project-crdt-counters-orset/) — learn eventual consistency through mergeable data types.
6. [Scriptable Workflow Runtime](project-scriptable-workflow-runtime/) — learn replay, determinism, and durable execution.

### Next builds to add after these

- Retry playground with exponential backoff, full jitter, and a circuit breaker
- Idempotent payment endpoint with idempotency keys and an outbox
- Distributed rate limiter with bounded disagreement
- Gossip-based membership and failure detection

## How to work through this

- Start with the fault-injection harness so every later project has a way to fail on purpose.
- Write tests before optimizations. If you cannot break a system deterministically, you cannot trust it.
- Keep a failure notebook: for every bug, record the wrong assumption and the test that would have caught it earlier.
- Prefer deterministic seeds and replayable scenarios so bugs are reproducible.
- Treat Temporal as a real-world reference point: it is a production system where replay, history, idempotency, and activity semantics are not theoretical.

## Resources

### Books

- **Designing Data-Intensive Applications — Kleppmann** — the main reference for this track
- **Patterns of Distributed Systems — Joshi** — concise patterns for logs, gossip, leader election, and more
- **Database Internals — Petrov** — storage engines, logs, and replication
- **Distributed Systems — van Steen & Tanenbaum** — formal and broad once the intuition is in place

### Courses

- **MIT 6.824 Distributed Systems** — the labs map almost perfectly to the project ladder above
- **CMU 15-445** — useful when you want to connect storage internals to distributed systems

### Papers and references

- **Raft** — the consensus paper to read while you implement
- **Dynamo** — vector clocks, quorums, and eventual consistency
- **Spanner** — TrueTime, consistency, and commit latency tradeoffs
- **The Tail at Scale** — why p99 dominates production experience
- **Temporal docs** — especially architecture, history, and replay semantics
- **Jepsen analyses** — the best way to see real failure modes in the wild

## Quick checks

- What happens to your system during a network partition: does it reject work or accept divergence?
- What does your idempotency strategy do when the client retries after a timeout?
- How do you detect a leader that is alive but no longer safe to trust?
- Why are wall clocks not enough to order events across machines?
- What exact failure sequence breaks your design?

## If you only do three things

1. Build the fault-injection harness.
2. Build the replicated log / Raft-style project.
3. Build the vector clock playground.

Those three teach most of the rest.
