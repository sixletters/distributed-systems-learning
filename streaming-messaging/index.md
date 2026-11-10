---
layout: page
title: Streaming & Messaging
permalink: /streaming-messaging/
---

Messaging is where distributed-systems theory meets production traffic. Queues and logs are how real systems decouple producers from consumers, absorb load spikes, and recover from failure — and every one of their guarantees (or lack thereof) is a deliberate trade you should be able to name. This section is about building the log, not just using Kafka.

## Topic map

| Topic | What you should be able to explain |
|-------|------------------------------------|
| The log abstraction | why an append-only log is the right core, offsets, retention, segments |
| Delivery semantics | at-most-once, at-least-once, exactly-once *effects* — and why the first two are the only deliveries that exist |
| Consumer groups | partition assignment, rebalancing, the rebalancing storm, static membership |
| Producer semantics | acks=0/1/all, idempotent producers, batching, linger, compression tradeoffs |
| Backpressure | what happens when consumers are slow: buffering, dropping, blocking, load shedding — and who decides |
| Ordering | per-partition ordering guarantees, why global ordering doesn't scale, key-based partitioning |
| Offset management | committing offsets, the commit-then-process vs process-then-commit hazard, replay |
| Queues vs logs | RabbitMQ-style work queues vs Kafka-style logs; visibility timeouts, dead-letter queues, poison messages |
| Stream processing | tumbling/hopping/session windows, watermarks, late data, state and checkpointing, the streaming join |
| Storage layer | segment files, index files, page cache dependence, zero-copy (`sendfile`), why brokers are mostly a filesystem problem |

If you can explain why Kafka consumers get duplicates during a rebalance and what an application must do to stay correct anyway, you're thinking at the right level.

## Project ladder

### 1. Mini log broker — **Go**

A single-node, then multi-partition, Kafka-in-miniature: append-only segment files, offset-indexed reads, producer/consumer protocol over TCP or gRPC.

**Milestones:**

- **M0 — The log.** Append records to segment files on disk; roll segments at a size threshold; read by offset via a sparse index file. Benchmark: sequential write throughput, read-by-offset latency. Compare against the same writes going through your [storage engine](../storage-databases/) and explain the difference.
- **M1 — Producer protocol.** Batching with a linger timeout, configurable acks (leader-only vs flushed-to-disk), per-request idempotency keys. Test: retry a produce after a timeout and prove no duplicate lands.
- **M2 — Consumer protocol + offsets.** Consumers fetch from an offset; offsets committed to an internal `__offsets` partition. Implement the two hazardous orders — commit-then-process (can lose) and process-then-commit (can duplicate) — and write tests that demonstrate each failure. Then implement the correct pattern: process + commit atomically against your own state.
- **M3 — Consumer groups + rebalancing.** N consumers, M partitions; trigger rebalance on consumer join/leave/crash. Test: during a rebalance, prove that your processing logic tolerates duplicates (idempotent sink) or document exactly when it doesn't.
- **M4 (stretch) — Replication.** Replicate partitions to follower brokers with a high-watermark for committed offsets. Now you're rebuilding the [replicated log](../distributed-systems/project-replicated-log-mini-system/) with a real workload on top — and you'll find out what you got wrong the first time.

**Why Go:** this is a control-plane-plus-services problem — many concurrent connections, message-passing internals, fast iteration on protocol design. The reference systems you'd read next (NATS, and the Kafka clients' behavior you emulate) live comfortably in Go's model.

**Success criteria:** a protocol doc, fault-injected integration tests (via your [virtual cluster harness](../distributed-systems/project-fault-injection-virtual-cluster/)), and a benchmark page: produce/consume throughput vs batch size, end-to-end p99, recovery time after broker kill.

### 2. Work-queue service with at-least-once workers — **Kotlin**

Build a RabbitMQ-shaped thing: a queue service with visibility timeouts, redelivery, dead-lettering after N failures, and a worker pool that processes with idempotent handlers. Then deliberately write a poison message and watch it flow to the DLQ. Kafka's log model doesn't teach you visibility-timeout semantics; this project does.

**Why Kotlin:** coroutines map perfectly onto worker pools with bounded concurrency, structured concurrency makes worker lifecycle bugs hard to write, and this is a JVM-ecosystem problem (Spring/rabbit in production) — you'll read real JVM queue clients with full comprehension after.

### 3. Windowed stream processor — **Go** (or Kotlin)

Over your mini broker: implement tumbling and hopping windows with watermarks, event-time processing, and late-data handling. Feed it an out-of-order event stream (your fault harness can generate one deterministically). Output per-window aggregates to a sink, checkpointed so restart resumes without double-counting. This is the smallest system where "exactly-once" stops being a slogan and becomes a concrete design problem in your own code.

### 4. Backpressure lab — **Go**

A load-generator → queue → consumer pipeline where you control consumer slowdown. Implement and compare four strategies: unbounded buffer (watch memory grow), bounded buffer with blocking (watch latency grow), bounded buffer with dropping (watch correctness go), and reactive pull-based consumption (watch throughput adapt). Produce one graph per strategy: throughput, p99, memory, over the same input curve. This lab is small and teaches more production judgment than most large projects.

## Resources

### Reading

- **Designing Data-Intensive Applications — Kleppmann** — chapter 11 (stream processing) is the anchor for this section
- **Kafka: the Definitive Guide** — operational depth on partitions, rebalancing, and producer configs
- **The Log — Jay Kreps** ("The Log: What every software engineer should know about real-time data's unifying abstraction") — the essay that explains why this whole section exists
- **RabbitMQ tutorials** — for the work-queue semantics Kafka deliberately doesn't have

### Papers

- **Kafka: a Distributed Messaging System for Log Processing (2011)** — the original design rationale
- **MillWheel (Google, 2013)** — watermarks and exactly-once in streaming; the clearest treatment of the hard parts
- **Disruptor (LMAX)** — the mechanical-sympathy queue; connects this section to [performance engineering](../performance-engineering/)

### Code to read

- **NATS server (Go)** — readable, production-grade, radically simpler than Kafka
- **Kafka's `Log.scala` and segment/index code** — after your M0; you'll recognize your own design decisions with better answers

## Quick checks

- A consumer processes a message, then crashes before committing its offset. What happens, and is your sink correct anyway?
- Why does Kafka give you ordering within a partition but not across partitions — and what does that force onto your schema design?
- What does `acks=all` actually wait for, and what failure does min-insync-replicas protect against?
- Your consumer lag is growing. Name three distinct root causes and the different fix for each.
- When is a work queue (RabbitMQ) the right answer and a log (Kafka) the wrong one?

## If you only do two things

1. Build the mini broker through M2 — the offset hazard tests are the real lesson.
2. Run the backpressure lab and keep the four graphs.

Together they teach you the two things production messaging incidents are always about: delivery semantics and flow control.
