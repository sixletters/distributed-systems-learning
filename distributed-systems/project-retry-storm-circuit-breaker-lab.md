---
layout: page
title: Project — Retry Storm & Circuit Breaker Lab
permalink: /distributed-systems/project-retry-storm-circuit-breaker-lab/
---

A controlled lab to build intuition for **retry amplification**, **p99 latency collapse**, and how **deadlines + backoff+jitter + circuit breakers** prevent cascading failure.

## Learning goals

- Implement configurable retry policies.
- Add deadlines end-to-end.
- Measure impact on throughput, error rates, and tail latency.
- Produce a “policy recommendation” based on data.

## System design

### Components

- **Flaky dependency** service (mock):
  - configurable failure probability
  - configurable latency distribution
  - optional “brownout” mode (slow responses)
- **Client** service: sends requests and retries according to policy.
- Optional: **metrics collector** that logs per-attempt timing.

### Retry policies to implement

For each policy, implement:

- max attempts
- per-attempt timeout (distinct from total deadline)
- exponential backoff
- full jitter vs no jitter
- optional circuit breaker

## Fault injection scenarios

Run the lab under:

1. **Hard failure**: dependency always returns 500.
2. **Timeouts**: dependency responds after the client timeout.
3. **Brownout**: dependency becomes slow (latency shifts right).
4. **Recovery**: dependency recovers after T seconds; observe time-to-normal.

## Milestones

### M0 — Fixed interval retries (the “bad” baseline)

- Implement naive retries without jitter.
- Run multiple concurrent clients.
- **Expected:** retry storms increase load exactly when the dependency is failing.

### M1 — Exponential backoff + full jitter

- Replace fixed interval with exponential backoff + jitter.
- **Expected:** fewer synchronized retry spikes; shorter recovery.

### M2 — Deadlines end-to-end

- Ensure each request carries a deadline through every hop.
- Verify that retries do not outlive the caller deadline.

### M3 — Circuit breaker

- Implement a breaker with thresholds:
  - open after N failures in a window
  - half-open probe policy
  - cooldown duration

### M4 — Load shedding / bulkheads (stretch)

- Add a queue limit + reject policy.
- Add separate worker pools for dependency calls.

## Metrics to plot

- p50/p95/p99 latency per scenario.
- attempts per successful request.
- concurrent in-flight attempts.
- error rate vs time.

## What to write in your README

Answer these explicitly:

- Under which scenario did naive retries fail?
- How did jitter change the shape of retry timing?
- Did your breaker reduce thread-pool exhaustion?
- What policy would you recommend for your real production service and why?

## Resources

- Distributed failure patterns: retry storms and cascading failure.
- Temporal tie-in: activity timeouts + retry policies are how Temporal prevents “retry roulette.”

## Success criteria

- You can reproduce a retry storm with the baseline.
- You can show measurable improvement with backoff+jitter + deadlines.
- The repo contains graphs and a clear writeup.
