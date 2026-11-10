---
layout: home
title: Distributed Systems, Storage, and Performance Engineering
---

This repo is a public learning log for becoming a strong distributed systems engineer. The center of gravity is building infrastructure that survives reality: nodes die, networks lie, clocks disagree, disks corrupt — and the system keeps its promises anyway. Performance engineering is a first-class track, because a correct system that misses its p99 is still a broken system. Applied ML shows up where it belongs: as workloads that stress the systems (inference serving, feature pipelines, vector search), not as the main subject.

The site is organized around **topics** and **projects**. Every topic section ends in hands-on projects, and every project has a chosen language — C++, Rust, Go, or Kotlin — picked for what the project teaches, not out of habit. See the [language strategy](languages/) for the reasoning.

---

## Start here

1. **Distributed systems** — [topic map + project ladder](distributed-systems/)
2. **Language strategy** — [why C++, Rust, Go, and Kotlin each earn their place](languages/)
3. **Storage engines** — [LSM trees, WAL, buffer pools, indexes](storage-databases/)
4. **Performance engineering** — [profiling, benchmarking, and making code fast for understood reasons](performance-engineering/)
5. **Full project ladder** — [every project, ordered, with languages assigned](projects/)

---

## Focus areas

**Distributed Systems** — partial failure, delivery semantics, consensus, sharding, replay, observability, and the discipline of designing for partitions.

**Storage & Streaming** — the systems underneath: LSM trees, WAL, buffer pools, log-oriented messaging, backpressure.

**Performance Engineering** — CPU caches, memory layout, profiling, benchmarking without lying to yourself, and reading flame graphs fluently.

**Networking** — kernel networking, eBPF/XDP, RDMA, and the host-level stack that high-performance systems are built on.

**Applied ML Systems** — inference serving, vector search, and feature pipelines as distributed-systems problems with latency budgets.

---

## Sections

| Section | Focus | Primary languages |
|---------|-------|-------------------|
| [Distributed Systems](distributed-systems/) | Failure modes, messaging, clocks, consensus, replication, sharding | Go, Rust |
| [Storage & Databases](storage-databases/) | LSM trees, WAL, B-trees, buffer pools, query execution | Rust, C++ |
| [Streaming & Messaging](streaming-messaging/) | Log brokers, queues, delivery semantics, backpressure | Go, Kotlin |
| [Networking](networking/) | Kernel networking, eBPF/XDP, io_uring, RDMA, DPDK | C, C++, Rust |
| [Performance Engineering](performance-engineering/) | Benchmarking, profiling, CPU/memory behavior, JVM internals | C++, Kotlin/JVM |
| [Languages](languages/) | C++, Rust, Go, Kotlin — what each is for here | all four |
| [ML Systems](ml-systems/) | Inference serving, vector search, feature pipelines | Go, Rust, Kotlin |
| [CS Fundamentals](cs-fundamentals/) | Algorithms, SQL, and the SQL-engine deep dive | language of choice |
| [Projects](projects/) | The full ladder, ordered, with languages assigned | all four |

---

## The goal

Become the person who can:

- Design a distributed system that still makes sense when a node dies, the network lies, or clocks disagree
- Build a storage engine, a log broker, and a consensus module from scratch — and know exactly which invariants the tests enforce
- Read a flame graph, a GC log, a `perf` report, and a packet capture with equal comfort
- Choose the right language for a system component and defend the choice on engineering grounds
- Write project docs that show how you think, not just what you built

If you are choosing one thing to start with, start with [Distributed Systems](distributed-systems/) and build the fault-injection harness first — every later project needs a way to fail on purpose.
