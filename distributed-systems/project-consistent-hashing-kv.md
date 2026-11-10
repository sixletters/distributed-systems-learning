---
layout: page
title: Project — Consistent Hashing KV with Live Rebalancing
permalink: /distributed-systems/project-consistent-hashing-kv/
---

A KV store that scales by sharding with consistent hashing. The focus is correctness during membership changes.

## Learning goals

- Implement consistent hashing with virtual nodes.
- Quantify key movement when nodes join/leave.
- Implement rebalancing with minimal disruption.

## System design

### Components

- **Shard workers**: each owns a set of hash ring positions.
- **Coordinator / membership service**: manages ring membership.
- **Client router**: routes keys to the node owning the key's ring position.

### Key movement measurement

Compare:

- naive `mod N` sharding: large movement when N changes
- consistent hashing: ~1/N movement

## Milestones

### M0 — Consistent hashing ring (no live traffic)

- Implement ring + virtual nodes.
- Write a tool that simulates adding/removing nodes and reports key movement.

### M1 — KV operations with stable membership

- Implement `GET` and `PUT` routed to the correct shard owner.
- Verify correctness under concurrent load.

### M2 — Live rebalancing (the hard part)

When membership changes:

- update ring
- re-route keys
- ensure data for moved keys is available during transition

You can implement one of:

- stop-the-world migration (simpler, but acceptable for learning)
- dual-write during transition
- read-repair: if you miss locally, ask the previous owner

### M3 — Hot key mitigation (stretch)

- Add a policy to detect hot keys and allocate extra capacity or replica routing.

## Fault injection

- Delay migration messages.
- Partition a shard for some time.

Decide and document behavior:

- will requests fail fast?
- will they redirect?
- will you accept temporary unavailability?

## What to write

- Key movement graphs.
- A transition correctness section: what guarantees do you give during migration?

## Success criteria

- Automated tests for membership change that verify no key returns stale data after migration completes.
- Clear measurement of movement vs naive hashing.
