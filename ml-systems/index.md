---
layout: page
title: ML Systems (the touch of ML)
permalink: /ml-systems/
---

This is deliberately *not* an ML repo — the theory tracks live in the [companion repo](https://github.com/sixletters/ml-engineer-learning). This section treats ML the way infrastructure engineers actually encounter it: as **workloads with latency budgets, fan-out patterns, and state**. Inference serving is a queueing problem. Vector search is a systems problem. Feature pipelines are streaming problems with correctness hazards. If you can serve a model at p99 < 50ms under load with batching, caching, and graceful degradation, you've learned more transferable engineering than another course on backprop.

## Topic map

| Topic | What you should be able to explain |
|-------|------------------------------------|
| Inference serving | request batching, dynamic batching windows, GPU/CPU placement, model warmup, cold start |
| Latency budgets | decomposing a 50ms budget across retrieval, features, model, ranking; what to cut when you exceed it |
| Vector search | ANN vs exact, HNSW vs IVF, recall/latency/memory tradeoffs, index build vs query path |
| Feature pipelines | online vs offline features, point-in-time correctness, training/serving skew, feature freshness |
| Two-stage recommenders | candidate retrieval → ranking; why the split exists, fan-out and latency at each stage |
| Model lifecycle as a systems problem | versioning, rollout/rollback, shadow traffic, A/B at the serving layer |
| Degradation | load shedding, fallback models, cached results, circuit breakers in front of models |
| Distributed training (awareness level) | data/model/pipeline parallelism, AllReduce, why it's a networking problem — depth lives in [Networking](../networking/) phase 4 |

The pattern to notice: none of these are ML problems. They're distributed-systems problems where one of the RPCs happens to be a matrix multiply.

## Project ladder

### 1. Inference gateway with dynamic batching — **Go**

A serving front-end for a model (a small ONNX model or a fake model with a tunable compute delay — the fake model is fine, the systems are the lesson):

- **M0 — Basic serving.** HTTP/gRPC endpoint, per-request inference, measure p50/p99 under load.
- **M1 — Dynamic batching.** Collect requests into batches with a max batch size and a max wait window (e.g. 5ms). Plot throughput vs p99 as you tune the window — this is the fundamental serving tradeoff, in one graph.
- **M2 — Admission control + load shedding.** Bounded queue, fast rejection under overload, per-client fairness. Compare to unbounded queueing and write up what overload actually does to tail latency (see [performance engineering](../performance-engineering/)).
- **M3 — Graceful degradation.** A cheap fallback (cached result or smaller model) behind a circuit breaker; fail over when the real model's p99 exceeds budget.
- **M4 (stretch) — Model versioning + shadow traffic.** Serve v1, shadow v2, compare outputs and latencies, promote by config.

**Why Go:** it's a high-fan-out network service with a scheduler at its core — goroutines, channels, and `time.Timer` batching windows are exactly the right tools, and this is how real serving infrastructure (e.g. much of the model-gateway ecosystem) is written.

### 2. HNSW vector index — **Rust**

Implement Hierarchical Navigable Small World from the paper: layered graph, greedy search, insertion with neighbor selection, `ef`/`M` parameters. Benchmark recall@10 vs latency vs memory against brute-force exact search on a real embedding dataset (SIFT1M is the standard). Then measure what index *build* costs and how concurrent queries behave during inserts — the part tutorials skip.

**Why Rust:** this is a data-plane data structure — pointer-heavy, allocation-sensitive, latency-critical, and a place where a use-after-free is a silent recall bug. Rust's ownership makes the graph's aliasing discipline explicit. (The production references: Qdrant, and Lucene's HNSW, are Rust and Java respectively.)

### 3. Feature-serving service — **Kotlin**

An online feature store in miniature: a gRPC/REST service that serves per-entity features with a freshness guarantee, backed by your [storage engine](../storage-databases/) or Redis, fed by a stream from your [mini broker](../streaming-messaging/):

- Write path: stream of feature updates → materialized per-entity latest values.
- Read path: `getFeatures(entity_id)` at p99 < 10ms.
- The actual lesson: **point-in-time joins**. Given (entity, timestamp), serve features *as of* that timestamp — and understand why getting this wrong produces training/serving skew that silently poisons models.

**Why Kotlin:** this is the JVM service archetype — coroutine-based fan-out, connection pooling, cache layering — and it sets you up to read real feature-store and recommendation-serving codebases, which are heavily JVM.

### 4. Two-stage recsys on your own infrastructure (capstone)

Compose the three projects above into one system: candidate retrieval via your HNSW index → feature fetch via your feature service → ranking via your inference gateway → results. Give the whole pipeline a 100ms budget, decompose it, load-test it, and find the bottleneck with tracing. Write the performance review using the [template](../performance-engineering/#the-performance-review-template). This capstone is the interview story: "I built a recommendation serving path end-to-end and here's where the milliseconds go."

## Resources

- **Designing Machine Learning Systems — Chip Huyen** — the systems view of the ML lifecycle; the serving chapters are the relevant ones here
- **The HNSW paper (Malkov & Yashunin, 2016)** — read it alongside the implementation; it's unusually implementable
- **TFX / KServe / vLLM docs** — skim for architecture; vLLM's continuous batching is the grown-up version of your M1
- **"Hidden Technical Debt in Machine Learning Systems" (Sculley et al.)** — the classic; read it as a list of failure modes to design against
- **Eugene Yan's and Chip Huyen's blogs on recsys/serving design** — production-shaped writeups
- The [networking track's phase 4](../networking/) for when you want distributed training depth (collectives, NCCL, RDMA)

## Quick checks

- Your model p99 is 30ms alone, but 400ms in the serving path. Enumerate where the time could be going.
- Why does dynamic batching raise throughput *and* p99 latency, and how do you pick the window?
- Your ANN index returns 92% recall at 1ms. What parameter moves recall to 97%, and what does it cost?
- What is training/serving skew, and which of your components is responsible for preventing it?
- A new model version is 5% better offline and 3x slower. What do you ship, and how do you decide?

## If you only do two things

1. Build the inference gateway through M2 — the batching graph alone is worth it.
2. Do the capstone's budget decomposition, even with fake models.

The deliverable of this section is not ML knowledge. It's the ability to say, precisely, where a model-backed system's milliseconds and failure modes live.
