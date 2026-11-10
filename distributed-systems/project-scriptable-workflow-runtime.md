---
layout: page
title: "Project: Scriptable Workflow Runtime (Rust + Python DSL on Temporal)"
permalink: /distributed-systems/project-scriptable-workflow-runtime/
---

A flagship project: build a Temporal worker where **workflows are written as Python scripts**, executed by an embedded interpreter inside a **Rust** worker. This is the Python/Rust sequel to the goja-based JS DSL runtime — and architecturally it's the same shape as the official TypeScript SDK (`sdk-core` + V8 isolates), just with different parts swapped in.

This is deliberately ambitious: you're not embedding a scripting language in a worker, you're **writing a new Temporal SDK**. That's the point — it forces you to understand sdk-core's lang bridge, determinism, replay, and command generation at the level the official SDK authors had to.

## Background and motivation

Previous version: Go worker + goja (pure-Go JS interpreter) running JS workflow scripts inside workflow code. Temporal acted as the durable runtime layer; users wrote orchestration logic in plain JS files.

This version: Rust worker + RustPython (Python interpreter written in Rust) running Python workflow scripts. Same idea, harder constraints, and a direct window into how sdk-typescript is actually built.

Why this is worth doing:

- sdk-typescript is `sdk-core` (Rust) + V8 + a thin lang bridge. Building the same thing with RustPython teaches you the lang-bridge contract that all SDKs share.
- Every hard problem in Temporal SDK design — determinism sandboxing, replay, versioning, history budgeting — appears here, and none of them can be dodged.
- It's a natural ML-platform artifact: data scientists write orchestration in Python; the platform team keeps control of determinism and side effects.

## Scope decision (make this first)

**The scripts are a DSL that uses Python syntax — not general-purpose Python.**

- No imports (or a tiny allowlist), no file/network I/O, no threads
- A controlled global namespace with a `temporal.*` bridge API
- Scripts must be cheap: pure orchestration glue, no heavy compute

This decision is what makes the project tractable. "Run any Python file as a workflow" is a research project; "run a restricted Python-syntax DSL as a workflow" is an engineering project.

## The replay model decision (make this second)

Workflow code is re-executed during replay. Two designs:

1. **Re-execution from the top (choose this one).** On every workflow task, re-run the whole script from the beginning. Bridge calls check history: if a result is recorded, return it immediately; if not, schedule the command and throw an internal `Yield` exception that unwinds the interpreter. Simple, robust, and — critically — does **not** require working async/await support in RustPython, which is its weakest area. Same model the goja version used.
2. **Coroutine suspension.** Scripts genuinely suspend at blocking calls. This is what real SDKs do (Python SDK uses asyncio, TS uses V8 async). Faster and more elegant, but requires mature async support — avoid for v1.

Known cost of re-execution: O(n) work per workflow task. Fine for DSL-scale scripts. Measure it and document the limit honestly.

## Architecture

```
Temporal Server
      │
      ▼
 sdk-core (Rust)          ← handles server comms, history, task polling
      │  lang bridge (workflow activations in, commands out)
      ▼
 Your runtime (Rust)      ← this project
      │  one interpreter instance per workflow execution
      ▼
 RustPython VM            ← restricted builtins, temporal.* bridge API
      │
      ▼
 User's .py workflow script
```

Per workflow execution, you create a fresh RustPython VM — the equivalent of a V8 isolate. Nothing is shared between executions.

## Milestones

### Milestone 0 — The lang bridge skeleton

- Stand up a worker on sdk-core that polls workflow tasks and can respond with commands
- Hardcode one workflow type that immediately completes with a fixed value
- **Test:** start the workflow from a client, see it complete in the Temporal UI. You now have the skeleton every SDK has.

### Milestone 1 — RustPython embedded

- Embed RustPython; run a hardcoded script string on each workflow activation
- Script result becomes the workflow result
- **Test:** `result = 1 + 1` completes the workflow with `2`.

### Milestone 2 — The determinism sandbox

This is the heart of the project. Before any user script runs, construct a VM with:

- `time.time()` / `datetime.now()` replaced with **workflow time** (from the workflow-task metadata, not the wall clock)
- `random` replaced with a **seeded PRNG** (seed derived from workflow ID + run ID)
- All I/O builtins removed: `open`, `os`, `socket`, `input`, `print` (or route `print` to workflow logs)
- `__import__` removed or allowlisted

**Test — the hostile-script suite.** Each of these must be caught or made deterministic:
- a script calling `time.time()` — must return identical values across replays
- a script using `random.random()` — same sequence on replay
- a script iterating a set and branching on order — forbidden or made deterministic
- a script attempting `open("/etc/passwd")` — hard error, surfaced cleanly
- float-heavy script — verify IEEE behavior matches across runs

### Milestone 3 — The activity bridge

- Expose `temporal.execute_activity(name, *args)` into the VM
- Implementation: look up the next expected command in history → if the activity already completed, return the recorded result; if not, emit a `ScheduleActivity` command and throw the internal `Yield` exception that unwinds the interpreter
- **Test:** a script with two sequential activities; kill the worker between them; restart; the second activity runs and the first is **not** re-executed (result came from history).

### Milestone 4 — The replay correctness suite

The single most valuable artifact of this project:

- Record real event histories from the server
- Replay them through your runtime with **mutated scripts** (reordered lines, changed literals) and verify you detect non-determinism the same way official SDKs do
- Replay the same history 100× and assert byte-identical command sequences
- This suite is what you'd contribute upstream / write about

### Milestone 5 — Timers

- `temporal.sleep(seconds)` → `StartTimer` command, same yield mechanism as activities
- **Test:** a script that sleeps an hour; verify it survives worker restarts and that no wall-clock sleeping happens anywhere.

### Milestone 6 — Signals and queries

- `temporal.wait_for_signal(name)` and query handlers exposed into the VM
- Signal handling under re-execution: signal events appear in history; script must observe them in history order
- **Test:** signal arrives while the script is between activities; replay stays correct.

### Milestone 7 — Versioning

- Implement a `temporal.get_version(change_id, min, max)` equivalent for interpreter/DSL changes
- Use it yourself: change the bridge semantics deliberately, patch with a version gate, prove old histories still replay
- **Test:** v1 history + v2 interpreter = correct replay; without the gate = loud non-determinism error.

### Milestone 8 — History budgeting and continue-as-new

- A loop calling activities 10k times will hit history limits — handle it in the runtime, not the user's script
- Expose `temporal.continue_as_new(state)`; runtime enforces history-size budgets and suggests/auto-invokes it
- **Test:** the 10k-activity script completes via continue-as-new without user intervention.

### Milestone 9 — Debugging experience

- Map history event N ↔ script line M; on replay failure, report the *script line* that diverged, not an opaque command index
- Even a crude version of this is a huge DX win and a differentiator vs. early-stage SDKs

### Milestone 10 — Chaos and limits

- Kill -9 the worker mid-script at random points, thousands of runs, assert correctness
- Duplicate deliveries, delayed activations
- Document measured limits: max script steps/sec, memory per execution, re-execution cost curve

## Hard parts / known traps

- **RustPython maturity**: stdlib gaps are fine (you're restricting anyway), but verify float behavior, string/unicode edge cases, and exception semantics early — your determinism guarantees ride on them
- **Determinism is a property of the sandbox, not the interpreter** — a perfect interpreter with `time.time()` exposed is non-deterministic; a modest interpreter with a locked-down namespace is correct
- **The Yield-exception trick** must not be catchable by user code — or if it is, catching it must be harmless
- **One VM per execution, always** — sharing interpreter state across executions is a replay-correctness time bomb
- **Memory per VM**: measure it on day one; it determines your max concurrent workflow capacity

## Alternative paths (and why this one)

| Path | Effort | Production-viable | Learning value |
|------|--------|-------------------|----------------|
| Python SDK + script-runner workflow | Low | Yes | Low–medium |
| Rust worker + CPython via PyO3 sub-interpreters (PEP 684) | High | Yes | High |
| **Rust worker + RustPython (this project)** | High | Experimental | **Highest** |

If the goal ever shifts from learning to shipping, the PyO3 path is the serious alternative: per-interpreter GIL gives you the isolate model with a mature runtime — at the cost of sandboxing CPython's enormous builtins surface.

## Resources

- [temporalio/sdk-typescript](https://github.com/temporalio/sdk-typescript) — read `packages/worker` and `packages/workflow`: this is your architectural reference
- [temporalio/sdk-core](https://github.com/temporalio/sdk-core) — the lang-bridge contract you implement against
- [temporalio/sdk-python](https://github.com/temporalio/sdk-python) — read the sandbox implementation (`workflow` module import restrictions) and the replayer
- [RustPython](https://github.com/RustPython/RustPython) — embedding docs, `vm` crate, frozen-module support
- Temporal docs: **workflow determinism**, **replay**, **versioning (`GetVersion`)**, **continue-as-new** — read these as an SDK author, not a user

## Success criteria

- A user writes a `.py` file with activities, timers, and signals; it runs durably on your Rust worker
- The hostile-script suite passes: no determinism escape hatch survives
- Killing the worker at any point never re-executes a completed activity and never loses state
- A recorded history replays byte-identically across 100 runs
- You can explain every line of the lang-bridge contract you implemented
