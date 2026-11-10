---
layout: page
title: Storage Engines & Database Internals
permalink: /storage-databases/
---

Distributed systems replicate and coordinate; storage engines are what they replicate and coordinate *over*. If you can't explain how a write becomes durable, how a read finds its data, and why compaction exists, your distributed-systems knowledge floats one layer too high. This section is where you build the layer underneath.

The flagship project is an LSM-tree key-value engine in **Rust** — see the [language strategy](../languages/) for why.

## Topic map

| Topic | What you should be able to explain |
|-------|------------------------------------|
| Durability & the WAL | why every write hits the log first, fsync semantics, group commit, what `fsync` does and does not guarantee |
| LSM trees | memtable, SSTables, levels, compaction (leveled vs tiered), write amplification vs read amplification vs space amplification |
| B-trees / B+ trees | page layout, splits, why B+ trees won for disk, buffer pool interaction |
| Bloom filters | false-positive math, why they make LSM reads viable, per-level filter sizing |
| Buffer pools | page eviction (LRU, clock, LRU-K), dirty pages, checkpointing, the mmap-vs-buffer-pool argument |
| Indexes | primary vs secondary, covering indexes, selectivity, why some queries ignore your index |
| Concurrency control | MVCC, snapshot isolation, locks vs latches, optimistic vs pessimistic |
| Transactions | ACI**D** in practice, isolation anomalies (dirty read, phantom, write skew), how databases actually implement them |
| Query execution | volcano/iterator model, vectorized execution, hash join vs nested loop, pushdown |
| Compression & encoding | prefix compression, delta encoding, columnar vs row layouts and when each wins |

If you can explain the read path and the write path of RocksDB end-to-end — including what happens on crash mid-compaction — you have the core of this section.

## Project ladder

### 1. LSM-tree KV storage engine — **Rust**

The flagship. A mini-RocksDB: WAL → memtable → SSTables → leveled compaction → Bloom filters → snapshots.

**Milestones:**

- **M0 — WAL + memtable.** `put`/`get`/`delete` against an in-memory sorted structure (a `BTreeMap` is fine), with every mutation appended to a write-ahead log first. Crash-recover by replaying the log. Test: kill -9 the process between write and ack, verify recovery.
- **M1 — Flush to SSTable.** When the memtable crosses a size threshold, freeze it and write an immutable sorted file with an index block. Reads check memtable → SSTables newest-first. Test: data survives flush + restart.
- **M2 — Leveled compaction.** Merge SSTables across levels when counts exceed thresholds. Measure write amplification before and after tuning level sizes. Test: compact while serving reads; readers must never see torn or duplicated state.
- **M3 — Bloom filters.** One per SSTable, sized to the expected key count at a target false-positive rate. Benchmark read latency on missing keys with and without filters.
- **M4 — MVCC snapshots.** Sequence-number every key version; snapshot reads pin a sequence number. Test: a snapshot must observe a consistent view while writes and compaction proceed.
- **M5 (stretch) — Column-family-style configuration** and an `sstabledump` debugging tool that prints any SSTable's blocks, keys, and metadata.

**Why Rust:** no GC pauses on the compaction path, explicit ownership of buffers and file handles, and `unsafe` contained to exactly where you need it (mmap, raw page layout). This is the TiKV/AgateDB architecture at toy scale.

**Success criteria:** deterministic crash-recovery tests with fault injection (kill at random points, torn writes via an injectable filesystem layer), a benchmark report (write throughput, read p50/p99, amplification numbers), and a doc explaining every durability decision.

### 2. B-tree index over paged files — **C++** (or Rust if you prefer)

Build a disk-backed B+ tree: 4–8KB pages, a small buffer pool with clock eviction, page splits, range scans. Deliberately benchmark it against your LSM engine on a read-heavy and a write-heavy workload and write up where each wins. This project exists so the LSM-vs-B-tree tradeoff is something you *measured*, not something you read.

### 3. SQL engine from scratch — language of choice (Go suggested)

Ported from the old repo — the full spec lives in [CS Fundamentals](../cs-fundamentals/#deep-dive-building-a-sql-engine-from-scratch). Tokenizer → parser → planner → executors → hash join → aggregation → B-tree index over your own storage. Go is a good fit: fast iteration, and you'll benchmark joins with `testing.B`.

### 4. Buffer pool study — **C++**

Implement a buffer pool with LRU, clock, and LRU-K eviction; drive it with synthetic scan/lookup workloads (loop, zipfian) and plot hit rates. The goal is intuition for why eviction policy is a workload question, not a correctness question.

## Resources

### Books

- **Database Internals — Alex Petrov** — the main reference for this track; storage structures, distributed databases, both halves matter here
- **Designing Data-Intensive Applications — Kleppmann** — chapters 3 (storage and retrieval) is the best short treatment of LSM vs B-tree
- **Architecture of a Database System — Hellerstein, Stonebraker, Hamilton** (free PDF) — how the pieces fit end-to-end
- **CMU 15-445/645 (Andy Pavlo, free on YouTube)** — watch the lectures that match the milestone you're on; the projects (bustub, in C++) are excellent if you want a second build

### Code to read

- **RocksDB wiki** — especially the compaction and WAL pages; the design docs are unusually honest about tradeoffs
- **SQLite source** — `btree.c` and `pager.c` are the most readable production storage code in existence
- **AgateDB / mini-lsm (GitHub)** — Rust LSM implementations at readable scale; read after your M2, not before

### Papers

- **The Log-Structured Merge-Tree (O'Neil et al., 1996)** — the original; short and clear
- **The Bw-Tree (Microsoft, 2013)** — latch-free indexing; read after you understand why latches are the bottleneck
- **WiscKey (FAST 2016)** — key-value separation; explains Badger's design and a real answer to write amplification

## Quick checks

- What exactly happens between `put(k, v)` returning and the data surviving a power loss? Name every fsync.
- Why does an LSM tree need compaction, and what breaks if you turn it off?
- Your read hits three SSTables and a memtable — what's the worst-case I/O count, and what reduces it?
- When does a B-tree beat an LSM tree? When does it lose? Do you have numbers, or just opinions?
- What is write amplification and how would you measure it in your own engine?

## If you only do two things

1. Build the LSM engine through M2 (flush + compaction) with crash-recovery tests.
2. Benchmark it against your B+ tree and write up the result.

The engine teaches you storage; the comparison teaches you judgment.
