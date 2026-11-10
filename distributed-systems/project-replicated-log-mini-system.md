---
layout: page
title: Project — Replicated Log Mini-System (Test-First)
permalink: /distributed-systems/project-replicated-log-mini-system/
---

Build a replicated append-only log with quorum commit (a Raft-adjacent subset). The emphasis is on **test-first safety invariants** under partitions.

## Learning goals

- Implement majority commit.
- Understand safety invariants behind consensus.
- Write tests that catch safety bugs.

## System design

### API

- `append(command)` on leader (or a single writer for simplicity)
- `read(index)`

### Replication

- replicate entries to followers
- **commit** once a majority has persisted the entry

## Safety invariants

Write these as testable properties:

1. **Committed entries never disappear.**
2. **Two committed logs must agree on the committed prefix.**
3. **Uncommitted entries may be lost on leader change.** (You must document this.)

## Milestones

### M0 — Single leader log replication

- 3 nodes, one leader.
- No partitions yet.
- Implement: append -> replicate -> commit on majority.

### M1 — Fault injection harness integration

- Use the virtual cluster harness (optional) or implement a minimal one.
- Drop/delay replication messages.

### M2 — Leader change

- After partition heals or leader is killed, elect a new leader (you can start with a simplified election).
- Ensure committed entries remain committed.

### M3 — Linearizability-ish history checker (stretch)

- Record client operations (append/read).
- Check histories against a sequential spec.

## What to write

- A section: “What safety property did we actually test and how?”
- A list of bugs you hit and the invariant that would have caught each earlier.

## Resources

- Raft section in DDIA.
- Jepsen for linearizability testing intuition.

## Success criteria

- A test suite with randomized partitions that never violates your committed-prefix invariant.
- Deterministic seeds for reproducible failures.
