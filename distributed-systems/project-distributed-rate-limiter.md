---
layout: page
title: Project — Distributed Rate Limiter
permalink: /distributed-systems/project-distributed-rate-limiter/
---

Build a distributed token bucket and learn the correctness/performance tradeoffs of enforcing global rate limits across many nodes.

## Learning goals

- Understand the difference between perfect and approximate global rate limiting.
- Learn which assumptions break under partitions.
- Evaluate error bounds and user-visible behavior.

## Design space (choose an approach per milestone)

### Milestone 0 — Single-node rate limiter

- Implement token bucket locally.
- Define semantics: what happens when the bucket is empty? block vs reject.

### Milestone 1 — Redis-backed global rate limit

- Use Redis as the source of truth.
- Enforce token bucket using Lua scripts (atomic updates).

### Milestone 2 — Clock and fairness issues

- Add synthetic clock skew to nodes.
- Measure how enforcement changes with drift.

### Milestone 3 — Approximate distributed limiter (stretch)

Implement one of:

- **Local buckets + occasional reconciliation**
- **Leaky bucket with probabilistic admission**
- **Gossip-based approximate enforcement**

Then quantify:

- overshoot (allowed requests above limit)
- undershoot (rejections while capacity exists)
- convergence time

## Fault injection harness

- Randomly pause one limiter node.
- Partition the network:
  - limiter nodes cannot reach Redis (or each other)

## Metrics

- admitted/rejected rate over time
- overshoot/undershoot bounds
- p99 admission latency
- recovery time after a partition heals

## What to learn (write these down)

- What does “correct” mean for a rate limiter during partitions?
- Which errors are safer: letting extra traffic through or rejecting too much?
- How do you communicate limiter behavior to clients (retry-after, backoff hints)?

## Temporal tie-in

If you use Temporal workers, global rate limits often sit around:
- activity execution
- external API calls
- workflow-triggered ingestion

This project teaches the enforcement layer for those boundaries.

## Success criteria

- A measurable comparison between Redis-based enforcement and approximate enforcement.
- A clear policy decision documented (what you prefer and why).
