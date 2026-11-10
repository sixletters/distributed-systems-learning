---
layout: page
title: Language Strategy — C++, Rust, Go, Kotlin
permalink: /languages/
---

This repo uses four languages on purpose. Each project states its language up front, and the choice is part of the lesson. The rule: **the language is chosen by what the project teaches**, and you should be able to defend every assignment in one sentence.

## The short version

| Language | Role in this repo | Mental model |
|----------|-------------------|--------------|
| **Go** | Control planes, services, and most distributed-systems projects | The default for infrastructure you iterate on fast |
| **Rust** | Data planes, storage engines, and correctness-critical components | When latency variance and memory safety both matter |
| **C++** | Kernel-adjacent and hardware-adjacent work | When you need to see the machine: RDMA, DPDK, caches |
| **Kotlin** | JVM application services and workflow-style systems | Production service ergonomics + the best profiling story |

## Go — the default for distributed systems

**Use it for:** the replicated log, the consistent-hashing KV, the rate limiter, the mini log broker, the inference gateway.

Why:

- MIT 6.824's labs are in Go — the Raft ecosystem, tooling, and reference material assume it.
- Goroutines and channels map directly onto the mental model of distributed systems: many concurrent actors passing messages. You spend your budget on *distributed* bugs, not language fights.
- gRPC, protobufs, and the cloud-native ecosystem (etcd, Kubernetes, NATS, CockroachDB) are Go. Reading real systems after building your toy versions is the fastest way to calibrate.
- Fast compile-test cycles matter when your test suite is a fault-injection harness running thousands of randomized partitions.

Where it loses: GC pauses on the hottest data paths, no control over memory layout, and no metaprogramming worth the name. That's what Rust and C++ are for.

## Rust — the data plane and the storage engine

**Use it for:** the LSM-tree storage engine, CRDTs, the eBPF latency tool (via `aya`), the ANN/vector index.

Why:

- Storage engines live on the p99 path. No GC means your tail latency is yours to control — this is why TiKV, AgateDB, RisingWave, and Redpanda are Rust.
- The ownership model forces you to design ownership of buffers, file handles, and in-flight requests explicitly — which is exactly the discipline a storage engine or a lock-free queue demands.
- `unsafe` is contained and auditable: you write lock-free structures and FFI bindings without the whole codebase becoming a hazard zone.
- CRDTs are pure data-structure design with subtle merge semantics; Rust's type system (and the compiler's refusal to let you be sloppy) makes it the best teacher here.

Where it loses: iteration speed and borrow-checker fights on highly cyclic, message-passing-heavy designs — which is why the consensus projects are Go.

## C++ — where you can see the machine

**Use it for:** kernel networking exercises (sockets, epoll, XDP), RDMA and DPDK work, the benchmark/profiling harness.

Why:

- The kernel APIs are C. The RDMA verbs API is C. DPDK is C. If you want to work at that layer, you write C/C++ — there is no abstraction to hide behind, which is the point.
- Cache-line alignment, false sharing, huge pages, NUMA, `mmap` — C++ makes these concrete because nothing is managed for you. Performance engineering needs a language where the hardware is visible.
- The infra codebases you'll eventually read or work on — ClickHouse, ScyllaDB, TensorFlow, RocksDB — are C++.

Where it loses: everything safety-related. Use sanitizers (ASan/UBSan/TSan) from day one and treat them as part of the build, not a debugging afterthought. Reach for C++ when the hardware is the subject; otherwise Rust gives you most of the control with fewer footguns.

## Kotlin — production service ergonomics on the JVM

**Use it for:** the durable workflow runtime, the streaming consumer/producer services, the feature-serving service, and anything where the JVM's observability story is the lesson.

Why:

- Temporal — the reference system for durable execution — is JVM-first. Building the workflow runtime in Kotlin means you can read Temporal's actual source and compare design decisions directly.
- Coroutines give you structured concurrency with the cleanest syntax of the four languages — a good fit for workflow orchestration and high-fan-out service code.
- The JVM has the best profiling and benchmarking toolchain in existence: async-profiler, JFR, JMH, JOL. The [JVM internals track](../performance-engineering/jvm/) teaches you to use it, and Kotlin is the most pleasant way to live on the JVM.
- Null safety and sealed classes model failure explicitly — aligned with how this repo thinks about errors.

Where it loses: anything allocation-sensitive or kernel-adjacent. GC is a feature until it's the bottleneck; know when to drop to Rust or C++.

## The decision heuristic

When starting a project, ask in order:

1. **Is the hardware the subject?** (kernel, NIC, caches, RDMA) → C++
2. **Is tail latency or memory safety the subject?** (storage, data plane, lock-free) → Rust
3. **Is distributed behavior the subject?** (consensus, sharding, messaging) → Go
4. **Is service ergonomics or JVM observability the subject?** (workflows, consumers, feature services) → Kotlin

If two apply, pick the one higher on the list — proximity to the hardware wins ties, because you can always build a Go service around a Rust core, but not the reverse.

## Learning order per language

You don't need four languages at once. Sequence them:

1. **Go first** — it unblocks the entire distributed-systems ladder, which is the spine of this repo.
2. **Rust second** — it unblocks the storage engine and CRDTs, and it will change how you think about ownership in every other language.
3. **C++ third** — as needed for the networking track; you can go far here with C plus modern C++ hygiene.
4. **Kotlin fourth** — when you reach the workflow runtime and the ML-systems services; the [Kotlin page](kotlin/) and [JVM page](../performance-engineering/jvm/) are ready when you are.

## Reference systems to read, per language

| Language | Codebases worth reading |
|----------|------------------------|
| Go | etcd (Raft in production), NATS, CockroachDB's kv layer, Vitess |
| Rust | TiKV, Redpanda, RisingWave, `tokio` internals, `crossbeam` |
| C++ | RocksDB, ClickHouse, ScyllaDB (Seastar), Folly, Abseil |
| Kotlin/JVM | Temporal Java SDK, Kafka (Java), Netty, JetBrains' kotlinx.coroutines |
