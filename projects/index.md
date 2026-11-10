---
layout: page
title: The Full Project Ladder
permalink: /projects/
---

Every project in this repo, in build order, with its language. The order is deliberate: each project builds testing infrastructure or intuition the next ones rely on. The language is part of the spec — see the [language strategy](../languages/) for the reasoning, and the decision heuristic there before you second-guess any assignment.

## Phase 1 — Foundations (do these first, in order)

| # | Project | Language | What it teaches |
|---|---------|----------|-----------------|
| 1 | [Fault-injection virtual cluster harness](../distributed-systems/project-fault-injection-virtual-cluster/) | **Go** | drop/delay/reorder/duplicate/partition, deterministic seeds, replayable scenarios — the test bed for everything below |
| 2 | [Vector clock playground & conflict resolver](../distributed-systems/project-vector-clock-playground/) | **Go** | causality, concurrent writes, merge strategies |
| 3 | [Replicated log mini-system](../distributed-systems/project-replicated-log-mini-system/) | **Go** | majority commit, leader change, safety invariants as testable properties |
| 4 | Cache & false-sharing lab | **C++** | cache lines, memory hierarchy, mechanical sympathy — see [performance engineering](../performance-engineering/) |
| 5 | Benchmarking harness study | **C++** | how benchmarks lie: DCE, constant folding, warm-up — Google Benchmark |

The order matters: 1–3 give you distributed-systems muscle memory and a way to fail on purpose; 4–5 give you the measurement discipline to know when you're lying to yourself.

## Phase 2 — The infrastructure core

| # | Project | Language | What it teaches |
|---|---------|----------|-----------------|
| 6 | [Consistent-hashing KV with live rebalancing](../distributed-systems/project-consistent-hashing-kv/) | **Go** | sharding, membership changes, key movement under load |
| 7 | LSM-tree storage engine | **Rust** | WAL, SSTables, compaction, Bloom filters, crash recovery — see [storage engines](../storage-databases/) |
| 8 | [CRDT counters & OR-Set](../distributed-systems/project-crdt-counters-orset/) | **Rust** | mergeable data types, eventual consistency, type-system-enforced merge semantics |
| 9 | Mini log broker | **Go** | segments, offsets, delivery semantics, consumer groups — see [streaming](../streaming-messaging/) |
| 10 | B+ tree over paged files | **C++** (or Rust) | buffer pools, page layout, LSM-vs-B-tree measured on your own benchmarks |

Project 10 exists so that "LSM vs B-tree" is something you measured, not something you read.

## Phase 3 — Services and the network

| # | Project | Language | What it teaches |
|---|---------|----------|-----------------|
| 11 | [Scriptable workflow runtime](../distributed-systems/project-scriptable-workflow-runtime/) | **Kotlin** | deterministic replay, durable execution — with Temporal's actual JVM source as your reference |
| 12 | Work-queue service with at-least-once workers | **Kotlin** | visibility timeouts, redelivery, DLQs, poison messages — see [streaming](../streaming-messaging/) |
| 13 | XDP packet filter / epoll server / io_uring exercises | **C/C++** | the kernel's packet path — see [networking](../networking/) |
| 14 | eBPF per-flow latency tool | **Rust** (aya) | kernel observability, BPF maps — see [networking](../networking/) |
| 15 | RDMA ping-pong + bandwidth benchmark | **C++** | verbs API, memory registration, one-sided ops — see [networking](../networking/) |

## Phase 4 — ML as a systems workload

| # | Project | Language | What it teaches |
|---|---------|----------|-----------------|
| 16 | Inference gateway with dynamic batching | **Go** | batching windows, admission control, load shedding — see [ML systems](../ml-systems/) |
| 17 | HNSW vector index | **Rust** | ANN graphs, recall/latency/memory tradeoffs |
| 18 | Feature-serving service with point-in-time correctness | **Kotlin** | online/offline features, training/serving skew |
| 19 | Two-stage recsys capstone | all of the above | 100ms budget decomposition, tracing, the performance review writeup |

## Phase 5 — Choose-your-own depth

Pick based on what pulled at you:

- **Consensus, for real** — extend project 3 into full Raft with membership changes and log compaction (**Go**), then run it through Jepsen-style checking.
- **The broker, replicated** — project 9's M4 stretch: partition replication with high-watermark commits (**Go**).
- **RDMA-backed KV** — FaSST-style: serve project 6's KV over RDMA verbs (**C++**), compare against your gRPC version, explain the delta.
- **Stream processor** — windowing, watermarks, checkpointed state over project 9 (**Go** or **Kotlin**).

## What to write in every project README

- The problem you are solving
- The failure mode you are defending against
- The invariants your tests enforce
- The tradeoff you chose and why
- The benchmark or profile that justifies the design
- What you would change at 10x traffic

## Good project habits

- Build the smallest working version first, then make it fail on purpose.
- Every simulator and harness gets a deterministic seed.
- Measure before and after every optimization, and write the [performance review](../performance-engineering/#the-performance-review-template) when the change matters.
- Keep a bug log: the wrong assumption, and the test that would have caught it earlier.
- If a project's language felt wrong for it, write up why — that's a lesson too.
