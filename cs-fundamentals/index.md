---
layout: page
title: CS Fundamentals
permalink: /cs-fundamentals/
---

Algorithms, data structures, SQL, networking, and software engineering practices. These underpin everything in the roadmap — if they're shaky, the advanced material won't stick.

## Algorithms & Data Structures

| Topic | Why It Matters |
|-------|---------------|
| Big-O notation, time/space complexity | Understand why ANN search is O(log n) vs brute force O(n) |
| Hash maps, trees, heaps, graphs | Used everywhere in pipeline and serving code |
| Sorting and search algorithms | Interview staple; informs understanding of indexing |

## Networking & SQL

| Topic | Why It Matters |
|-------|---------------|
| HTTP, REST, TCP/IP, latency vs throughput | Needed for serving, API design, debugging production issues |
| SQL: joins, aggregations, window functions, indexes, query plans | Feature engineering, debugging pipelines |

## Software Engineering Practices

| Topic | Why It Matters |
|-------|---------------|
| Git: branching, rebasing, conflict resolution, bisect | Daily use; bisect is a superpower for finding regressions |
| Clean code: naming, single responsibility, avoiding premature abstraction | Production ML code is read far more than it's written |
| Testing: unit vs integration vs e2e | Confidence in changes, catching regressions |
| Shell: pipes, grep, awk, sed, find, xargs | Debugging production jobs, data wrangling |

## Resources

- **Grokking Algorithms — Aditya Bhargava** — visual and accessible
- **LeetCode Easy/Medium** — 30-50 problems focused on arrays, hashmaps, trees, sliding window
- **Mode Analytics SQL Tutorial** (free); **SQLZoo** for practice
- **Pro Git — Scott Chacon** (free online)
- **A Philosophy of Software Design — John Ousterhout**
- **Python Testing with pytest — Brian Okken**
- **The Linux Command Line — William Shotts** (free PDF)

## Key Exercises

- LeetCode: 2 Easy + 1 Medium per session for 4-6 weeks — arrays, hashmaps, sliding window patterns
- Write raw SQL for 10 business questions on a dataset you care about, window functions for at least 3
- `EXPLAIN ANALYZE` a slow SQL query in local Postgres, rewrite it to use an index
- Use `git bisect` to find which commit broke something — introduce a bug yourself, then find it
- Shell one-liner: find `.log` files modified in the last 24h, extract ERROR lines, count by type, sort descending

---

## Deep Dive: Building a SQL Engine from Scratch

The best way to truly understand SQL internals — query parsing, execution plans, joins, and indexes — is to build a simple SQL engine yourself. This is my primary vehicle for learning SQL deeply, rather than just writing queries and moving on.

**What building it teaches you:**

| Component | What You Learn |
|-----------|---------------|
| Lexer / tokenizer | How SQL text is broken into tokens — keywords, identifiers, literals, operators |
| Parser → AST | How grammar rules turn tokens into a tree structure your code can traverse |
| Logical planner | How `SELECT`, `FROM`, `WHERE`, `JOIN`, `GROUP BY` map to a plan of operations |
| Optimizer | Filter pushdown, join ordering — why the same query can run 100x faster with a rewrite |
| Executors: scan, filter, project | The inner loop of every query — how rows actually flow through the engine |
| Hash join vs nested loop join | Why hash joins dominate for large tables, when nested loop wins |
| Aggregation (GROUP BY) | How hash aggregation and sort-based aggregation work internally |
| B-tree index | Why index lookups are O(log n), how range scans work, what makes an index selective |
| `EXPLAIN` output | Now you can read it because you built what it's describing |

**Build order:**

1. **Tokenizer** — split a SQL string into a flat list of typed tokens
2. **Parser** — recursive descent parser that produces an AST for `SELECT`, `FROM`, `WHERE`, `JOIN`, `GROUP BY`, `ORDER BY`, `LIMIT`
3. **In-memory table storage** — load CSV files into row-oriented tables
4. **Sequential scan + filter** — execute `SELECT * FROM t WHERE x > 5`
5. **Projection** — evaluate expressions and return only the requested columns
6. **Nested loop join** — get two tables joining correctly before worrying about performance
7. **Hash join** — replace the inner loop with a hash table, measure the speedup
8. **Hash aggregation** — implement `GROUP BY` with `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`
9. **Sort + ORDER BY / LIMIT**
10. **B-tree index** — build an index on a column, add index scan to the planner, measure scan vs index lookup on 1M rows

**Stretch goals:**
- Window functions (`ROW_NUMBER`, `RANK`, `LAG`, `LEAD`) — implement the frame logic
- Basic query optimizer: push filters below joins, choose index scan over full scan based on estimated selectivity
- Write-ahead log (WAL) — understand how durability works at the storage layer

**Resources for building the engine:**

- **Crafting Interpreters — Robert Nystrom** (free online at craftinginterpreters.com) — the best resource for understanding lexers, parsers, and ASTs. Written for a general language but the tokenizer and recursive descent parser chapters translate directly. Read Part II (tree-walk interpreter) first
- **CMU 15-445/645 Database Systems (Andy Pavlo, free on YouTube)** — the definitive university course on database internals. Covers buffer pools, B-trees, hash joins, query execution, and optimisers. Watch the lectures that match what you're currently building
- **Architecture of a Database System — Hellerstein, Stonebraker, Hamilton** (free PDF) — the canonical paper on how relational databases are structured end-to-end. Dense but worth reading alongside the build
- **SQLite source code** — once you have a basic engine working, read SQLite's `vdbe.c` (virtual database engine) and `btree.c`. It's the most readable production database codebase; comments explain why decisions were made
- **Let's Build a Simple Database** (cstack.github.io/db_tutorial) — a step-by-step tutorial building a SQLite clone in C. Follow along in Python instead — the concepts are identical and it covers B-tree pages and WAL
- **Use The Index, Luke (use-the-index-luke.com)** — the best practical resource for understanding B-tree indexes, index selectivity, and why certain queries don't use indexes even when one exists. Read this alongside the index build step

See [Projects](../projects/) for the full project spec.

---

*Notes and exercises will be added below as I work through this section.*
