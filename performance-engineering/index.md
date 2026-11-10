---
layout: page
title: Performance Engineering
permalink: /performance-engineering/
---

Performance engineering is the discipline of making systems fast **for understood reasons**. Not "it felt slow so I added a cache" — measured hypotheses, controlled experiments, and a written explanation of where the time goes. In a distributed-systems career this is the multiplier: the same consensus implementation can be 10x apart depending on whether you understand batching, cache behavior, and what your runtime does under load.

The JVM gets its own deep track here — see [JVM Deep Internals](jvm/) — because it has the best profiling toolchain of any runtime and it's where the [Kotlin service projects](../languages/) live. This page covers the language-agnostic core.

## Topic map

| Topic | What you should be able to explain |
|-------|------------------------------------|
| Measurement discipline | warm-up, steady state, coordinate omission, why you report p50/p99/max and never just the mean |
| Benchmarking correctly | dead-code elimination, constant folding, DCE in every language's toolchain, harness design per language (JMH, Criterion, Google Benchmark, `testing.B`) |
| CPU behavior | caches (L1/L2/LLC), cache lines, false sharing, branch prediction, prefetch, SIMD |
| Memory behavior | allocation cost, object layout, pointer chasing vs data-oriented layout, NUMA basics |
| I/O behavior | syscall overhead, page cache, buffered vs direct I/O, `io_uring` vs `epoll` |
| Profiling | sampling vs instrumentation, safepoint bias, CPU vs wall-clock vs allocation profiling, reading flame graphs |
| Tracing | distributed traces, spans, tail-latency decomposition across services |
| The tail | why p99 dominates user experience, The Tail at Scale, hedged requests, queueing delay vs service time |
| Concurrency costs | context switches, lock contention, Amdahl's law, when parallelism makes it slower |
| Load testing | open vs closed load models, coordinated omission, saturation testing, finding the knee |

If you can take a slow service, produce a flame graph and a latency histogram, state the hypothesis, apply one fix, and show the before/after with the same harness — you have the core skill. Everything else is depth.

## Project ladder

### 1. A benchmarking harness study — **C++**

Build a tiny benchmark harness with Google Benchmark and deliberately get it *wrong* first: benchmark a function whose result is unused (watch the optimizer delete it), a loop with a compile-time-constant input (watch it get constant-folded), an unwarmed benchmark. Then fix each with `benchmark::DoNotOptimize`, opaque inputs, and proper warm-up. Write up every way the naive version lied.

**Why C++:** the compiler's optimizations are most visible and most ruthless here; the lesson transfers to every other language's JIT/AOT behavior.

### 2. Cache and false-sharing lab — **C++**

- Measure strided access over an array at increasing stride; find your cache-line size and cache levels from the latency cliffs.
- Two threads incrementing adjacent counters (one cache line) vs padded counters — quantify false sharing.
- Pointer-chasing linked list vs packed array traversal — quantify data-oriented layout.

Deliverable: three graphs and one paragraph each. This is mechanical sympathy, made concrete, in one afternoon per experiment.

### 3. Profiling a real service end-to-end — **Kotlin/JVM**

Take one of your service projects (the [workflow runtime](../distributed-systems/project-scriptable-workflow-runtime/) or the feature service from [ML Systems](../ml-systems/)) and run the full loop: async-profiler CPU flame graph → find the hot path → fix → re-profile → wall-clock profile for lock contention → allocation profile for GC pressure. The exercises and toolchain are all in [JVM Deep Internals](jvm/). The deliverable is a written performance review of your own system, in the format you'd post to an engineering blog.

### 4. Tail-latency shootout — **Go**

Build a trivial HTTP service and load-test it properly: closed-model load (wrk2, not wrk — understand why), latency histograms (use HdrHistogram), then introduce one GC-heavy code path and one lock-contended code path. Measure p50/p99/p99.9 before and after each. Then mitigate: object pooling, sharded locks, hedged requests. The point is learning that p99 is a different system from p50 — it has different bottlenecks and different fixes.

**Why Go:** the GC and goroutine scheduler give you real, tunable tail-latency phenomena without kernel-level setup; `testing.B` and the pprof toolchain are excellent.

### 5. `perf` and the kernel's view — **C**

Profile the networking projects from the [networking track](../networking/) with `perf stat` / `perf record`: instructions-per-cycle, cache misses, context switches, syscall counts. Learn to read `perf` output the way you read flame graphs. This is the bridge between application-level and kernel-level performance work.

## The performance review template

Every project above ends in the same document, which is the actual deliverable:

1. **Claim** — what you believe is slow, and why
2. **Measurement** — harness, load model, percentiles, flame graph / profile attached
3. **Hypothesis** — the mechanism you think explains the measurement
4. **Change** — one fix, isolated
5. **Result** — same harness, before/after, honest about regressions elsewhere
6. **Explanation** — why the fix worked, stated in terms of the machine (caches, allocations, syscalls, scheduling)

If you can't write step 6, you got lucky, not fast.

## Resources

### Books

- **Systems Performance — Brendan Gregg** — the methodology book; USE method, the analysis playbook
- **BPF Performance Tools — Gregg** — when the answer is in the kernel
- **Java Performance — Scott Oaks** — for the JVM track
- **What Every Programmer Should Know About Memory — Ulrich Drepper** (free PDF) — still the canonical memory-hierarchy document
- **The Art of Computer Systems Performance Analysis — Jain** — the statistics of measurement; dry but correct

### Papers and talks

- **The Tail at Scale (Dean & Barroso, 2013)** — the reason this section exists
- **Coordinated Omission (Gil Tene's talks + the wrk2 writeup)** — why most load tests lie
- **Mechanical Sympathy (Martin Thompson's blog/talks)** — the mindset behind the C++ labs

### Tools to become fluent in

| Tool | For |
|------|-----|
| `perf` | CPU, cache, syscall analysis on Linux |
| async-profiler | JVM CPU/alloc/wall/lock profiling without safepoint bias |
| pprof | Go CPU/heap/goroutine/block profiling |
| HdrHistogram | honest latency percentiles |
| wrk2 / k6 (constant-arrival) | load generation with an open model |
| flamegraph tooling | reading everything above |

## Quick checks

- Your load test says p99 is 40ms. What was the load model, and would the answer change under open-model load?
- A benchmark got 3x faster after a refactor. Name three ways the benchmark could be lying before you celebrate.
- Why does a mutex protecting a 5ns critical section cost 50ns+ under contention?
- What's the difference between CPU-time and wall-clock profiling, and which one finds lock contention?
- Your service's p50 is 2ms and p99 is 800ms. Where do you look first?

## If you only do two things

1. Run the cache/false-sharing lab — hardware intuition pays off forever.
2. Do one full profiling loop on a real service and write the performance review document.

The labs give you intuition; the review document gives you the habit.
