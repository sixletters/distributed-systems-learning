---
layout: page
title: JVM Deep Internals
permalink: /performance-engineering/jvm/
---

The JVM is one of the most sophisticated runtimes ever built. Understanding it means you can reason about Kotlin and Java performance the way Python internals let you reason about CPython — not guessing, but knowing.

Five areas, in order of dependency.

---

## 1. Memory Layout

The JVM heap is not a flat array. It's divided into regions that map to object lifetime, and the GC uses this structure to avoid scanning everything on every collection.

### The regions

**Young generation** — where all new objects are allocated. Divided into: Eden (where objects are born), and two Survivor spaces (S0 and S1). Most objects die here without ever reaching old gen — the "generational hypothesis."

**Old generation (Tenured)** — objects that survive enough minor GCs get promoted here. Collected less frequently but pauses are longer.

**Metaspace** (replaces PermGen since Java 8) — class metadata: loaded class definitions, method bytecode, interned strings. Not in the heap — grows in native memory.

**Code cache** — JIT-compiled native code lives here. If it fills up, the JVM stops compiling and falls back to interpreted mode.

**Thread stacks** — each thread has its own stack for stack frames (local variables, operand stack, return addresses). Outside the heap.

### Key tools

```bash
# Heap dump — snapshot of all objects
jmap -dump:format=b,file=heap.hprof <pid>

# Object histogram — what's taking up space, without a full heap dump
jmap -histo:live <pid> | head -30

# Thread stacks
jstack <pid>

# JVM flags in use
jcmd <pid> VM.flags

# GC log (add to JVM startup)
-Xlog:gc*:file=gc.log:time,uptime:filecount=5,filesize=20m
```

### Exercises

**Exercise 1: Map heap regions with GC logging**

Start any JVM application with GC logging enabled:

```bash
java -Xlog:gc*,gc+heap=debug:file=gc.log:time,uptime \
     -Xms256m -Xmx512m \
     -jar your-app.jar
```

Find in the log: (1) the Eden size at startup, (2) the first minor GC — what was live before vs after, (3) the first object promotion to old gen, (4) the first major GC if it happens. Write down the lifecycle of an object in this run.

**Exercise 2: Object histogram for memory diagnosis**

Run a program that creates many objects of known types. Use `jmap -histo:live <pid>` to list what's in the heap. Find your objects. Then add a memory leak (retain objects in a growing list), re-run, and find where the extra memory went. This is how you diagnose memory leaks in production — you look at the histogram first.

**Exercise 3: `sys.getsizeof` equivalent — measure JVM object size**

The JVM doesn't expose object sizes directly, but the `Instrumentation` API does. Use [Java Object Layout (JOL)](https://openjdk.org/projects/code-tools/jol/) to print the layout of any object:

```kotlin
// build.gradle.kts: testImplementation("org.openjdk.jol:jol-core:0.17")
import org.openjdk.jol.info.ClassLayout

println(ClassLayout.parseClass(HashMap::class.java).toPrintable())
println(ClassLayout.parseClass(ArrayList::class.java).toPrintable())
```

For each: note the header size (12 bytes on compressed OOPs), field layout, any padding. Compare an empty `ArrayList` vs one with 10 elements — does the object size change? (The array backing it does, but the `ArrayList` object itself doesn't.)

---

## 2. Garbage Collectors

There is no single "the GC." CPython has one GC strategy. The JVM ships four production-ready GC algorithms with different latency/throughput tradeoffs.

### G1GC (default since Java 9)

Divides the heap into equal-sized regions (1MB–32MB). Regions can play any role (Eden, Survivor, Old, Humongous). G1 collects the regions with the most garbage first — "Garbage First."

- Target: configurable pause goals (`-XX:MaxGCPauseMillis=200`)
- Concurrent: most marking happens while the application runs
- Stop-the-world: short evacuation pauses
- Good for: general-purpose, heap sizes 4GB–64GB

### ZGC (production since Java 15)

Fully concurrent — GC work runs concurrently with the application almost all the time. Uses colored pointers (stores GC metadata in pointer bits) for concurrent relocation.

- Pause times: sub-millisecond regardless of heap size
- Throughput: slightly lower than G1 due to concurrent overhead
- Heap sizes: 8MB to 16TB
- Good for: latency-critical services, very large heaps

### Shenandoah

Similar goals to ZGC but uses a different technique (Brooks forwarding pointers for concurrent compaction). OpenJDK project from Red Hat.

- Also sub-millisecond pauses
- Slightly different throughput/latency tradeoff than ZGC
- Good for: same use cases as ZGC, may perform differently depending on workload

### Serial / Parallel GC

Old algorithms — stop-the-world, all threads pause for collection. Parallel GC uses multiple threads for the GC work itself. Still useful for batch jobs where throughput matters more than pauses.

### Exercises

**Exercise 1: GC log analysis**

Run this allocation workload under G1GC and ZGC:

```kotlin
fun main() {
    val results = mutableListOf<ByteArray>()
    repeat(10_000) {
        // short-lived allocations
        val trash = ByteArray(10_000)
        // keep every 100th object — simulates mixed survival
        if (it % 100 == 0) results.add(trash)
    }
    println("Done, retained: ${results.size} objects")
}
```

With G1GC:
```bash
java -Xms256m -Xmx512m -XX:+UseG1GC \
     -Xlog:gc*:file=g1.log:time,uptime \
     -jar app.jar
```

With ZGC:
```bash
java -Xms256m -Xmx512m -XX:+UseZGC \
     -Xlog:gc*:file=zgc.log:time,uptime \
     -jar app.jar
```

Compare: maximum pause time, total GC overhead (time spent in GC / total time), number of GC events. Upload to [GCEasy.io](https://gceasy.io/) for a visual breakdown.

**Exercise 2: Tune G1GC pause target**

Run a latency-sensitive workload (e.g. a server that handles 1000 requests/sec). With `-XX:MaxGCPauseMillis=50`, measure p99 latency. Then change to `200`, then `500`. Does the pause target actually get met? What happens to throughput as you tighten the target? Write up the tradeoff.

**Exercise 3: Force and observe a full GC**

```kotlin
fun main() {
    val held = mutableListOf<ByteArray>()
    repeat(500) { held.add(ByteArray(1_000_000)) }  // fill heap
    System.gc()  // request a full GC
    Thread.sleep(5000)
}
```

Run with `-verbose:gc` and observe: does a full GC happen? How long does it pause? What's the heap before and after?

---

## 3. JIT Compilation

The JVM starts by interpreting bytecode, then progressively compiles hot code to native machine code. Understanding this is essential for writing JMH benchmarks correctly and for understanding why JVM apps have a "warm-up" period.

### Tiered compilation

| Tier | Mode | When |
|------|------|------|
| 0 | Interpreter | Always starts here |
| 1 | C1, no profiling | After ~1000 invocations |
| 2 | C1, limited profiling | Transitional |
| 3 | C1, full profiling | Gathering data for C2 |
| 4 | C2 (server compiler) | After ~10,000 invocations — full optimisation |

C2 is aggressive. It inlines methods, eliminates dead code, unrolls loops, vectorises, and applies escape analysis to stack-allocate short-lived objects.

### Key optimisations

**Inlining** — the JIT copies the body of a called method into the caller, eliminating the method call overhead and enabling further optimisations. Methods stay inlinable if they're small enough (`-XX:MaxInlineSize=35` bytes of bytecode by default).

**Escape analysis** — if an object is created in a method and never passed outside it, the JIT can allocate it on the stack (zero GC pressure) or even eliminate it entirely (scalar replacement).

**Loop unrolling** — the JIT replaces a loop with multiple copies of the body, reducing branch overhead. Enables SIMD auto-vectorisation.

**Dead code elimination** — if the JIT proves a branch is never taken, it removes it. This is why naive JMH benchmarks that don't consume results produce misleadingly fast numbers — the JIT eliminates the computation entirely.

### Tools

```bash
# See JIT compilation events
-XX:+PrintCompilation

# See inlining decisions (verbose)
-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining

# Disable tiered compilation (interpret only, for comparison)
-Xint

# Force C2 only
-XX:-TieredCompilation
```

### Exercises

**Exercise 1: Warm-up effect — measure it**

```kotlin
fun compute(n: Int): Long {
    var sum = 0L
    for (i in 0 until n) sum += i
    return sum
}

fun main() {
    // Run 10,000 times and measure each call
    repeat(10_000) { iteration ->
        val t0 = System.nanoTime()
        compute(1_000)
        val elapsed = System.nanoTime() - t0
        if (iteration % 500 == 0) println("Iter $iteration: ${elapsed}ns")
    }
}
```

You should see latency drop dramatically around iteration 1000 (C1) and again around iteration 10000 (C2). This is why you must warm up a JVM before benchmarking.

**Exercise 2: Write a JMH benchmark correctly**

```kotlin
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(1)
open class StringBenchmark {

    private val str = "Hello, World! This is a test string."

    @Benchmark
    fun withFormat(bh: Blackhole) {
        bh.consume(String.format("Value: %s", str))
    }

    @Benchmark
    fun withConcat(bh: Blackhole) {
        bh.consume("Value: " + str)
    }

    @Benchmark
    fun withTemplate(bh: Blackhole) {
        bh.consume("Value: $str")
    }
}
```

Run it. Compare the three. Write up: which is fastest, why (check the bytecode for each), and what the `Blackhole` is preventing.

**Exercise 3: Escape analysis — verify with allocation profiling**

```kotlin
data class Point(val x: Double, val y: Double)

fun distance(x1: Double, y1: Double, x2: Double, y2: Double): Double {
    val p1 = Point(x1, y1)  // does this allocate on heap?
    val p2 = Point(x2, y2)  // or on stack via escape analysis?
    return Math.sqrt((p2.x - p1.x).pow(2) + (p2.y - p1.y).pow(2))
}
```

Use async-profiler in allocation profiling mode (`-e alloc`) to check if `Point` objects appear in the allocation profile after warm-up. If escape analysis worked, they should be stack-allocated and invisible. If not, you'll see them clearly.

---

## 4. Profiling — async-profiler and JFR

### async-profiler

The best JVM profiler. Works by sending OS signals to threads at a fixed interval and capturing the call stack — no JVM instrumentation, no safepoint bias, minimal overhead.

Modes:
- `cpu` — CPU-time profiling (where is time spent?)
- `alloc` — allocation profiling (who is allocating?)
- `wall` — wall-clock profiling (what is each thread doing, including blocked time?)
- `lock` — lock contention profiling

```bash
# CPU profiling for 30 seconds, output a flame graph
./profiler.sh -d 30 -f flame.html <pid>

# Allocation profiling
./profiler.sh -e alloc -d 30 -f alloc.html <pid>

# Wall-clock profiling (shows blocked threads too)
./profiler.sh -e wall -d 30 -f wall.html <pid>
```

### JFR (Java Flight Recorder)

Built into the JVM since Java 11 (free). Records a continuous stream of events: GC, JIT compilation, thread activity, exceptions, I/O. Low overhead (< 1% typically).

```bash
# Start recording
jcmd <pid> JFR.start duration=60s filename=recording.jfr

# Dump a running recording
jcmd <pid> JFR.dump filename=recording.jfr

# Open in JDK Mission Control
jmc recording.jfr
```

### Reading a flame graph

The x-axis is the population of samples (width = time fraction, NOT execution order). The y-axis is the call stack (bottom = entry point, top = where time is actually spent). Wide bars at the top of the flame are the hot code.

Look for: wide bars high in the flame (hot code), GC threads taking large fractions, blocked threads in a wall-clock flame.

### Exercises

**Exercise 1: Profile a CPU-bound function — find and fix the bottleneck**

Write a deliberately inefficient function:

```kotlin
fun processData(data: List<Int>): List<Int> {
    val result = mutableListOf<Int>()
    for (item in data) {
        // Inefficient: sorts on every iteration
        val sorted = data.sortedDescending()
        result.add(sorted.indexOf(item))
    }
    return result
}
```

Run it on `(1..10_000).toList()`. Profile with async-profiler (`-e cpu`). Find `sortedDescending` and `indexOf` dominating the flame graph. Fix it (sort once outside the loop, use a map for lookups). Re-profile to verify.

**Exercise 2: Allocation profiling — find what's creating GC pressure**

```kotlin
fun buildMessages(n: Int): List<String> {
    return (1..n).map { i ->
        "Processing item number $i with result ${i * 2}"  // string interpolation = String allocation
    }
}
```

Profile with `-e alloc`. Find `String` or `StringBuilder` objects dominating the allocation profile. Consider: is this allocation necessary? What would you change if this were on the hot path of a latency-sensitive service?

**Exercise 3: Wall-clock profiling — find blocked threads**

Write a server that handles 100 concurrent requests but uses a shared synchronized resource:

```kotlin
val lock = Object()
var counter = 0

fun handleRequest() {
    Thread.sleep(10)  // simulate I/O
    synchronized(lock) {
        Thread.sleep(50)  // simulate slow critical section
        counter++
    }
}
```

Run 50 threads calling `handleRequest`. Profile with `-e wall`. You should see a large fraction of samples in `BLOCKED` state waiting for the lock. This is lock contention — the wall-clock flame graph shows it immediately.

---

## 5. JMH — Benchmarking Correctly

JMH (Java Microbenchmark Harness) is the only correct way to benchmark JVM code. Naive timing loops produce garbage results because of JIT warm-up, dead code elimination, and CPU frequency scaling.

### Why naive benchmarks lie

```kotlin
// DO NOT do this — results are meaningless
fun naiveBenchmark() {
    val start = System.nanoTime()
    repeat(1_000_000) {
        "hello" + "world"  // JIT eliminates this — result is never used
    }
    println("${System.nanoTime() - start}ns")  // measures near-zero
}
```

The JIT sees that the string concatenation result is discarded and eliminates the whole loop. Your "benchmark" measures nothing.

JMH prevents this by: forcing warm-up phases, consuming results via `Blackhole`, forking fresh JVM processes for each benchmark, and handling JIT compilation state properly.

### Setup (Kotlin + Gradle)

```kotlin
// build.gradle.kts
plugins {
    kotlin("jvm") version "2.0.0"
    id("me.champeau.jmh") version "0.7.2"
}

dependencies {
    jmh("org.openjdk.jmh:jmh-core:1.37")
    jmh("org.openjdk.jmh:jmh-generator-annprocess:1.37")
}
```

### Exercises

**Exercise 1: Benchmark collection operations**

```kotlin
@State(Scope.Benchmark)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 5)
@Measurement(iterations = 5)
@Fork(1)
open class CollectionBenchmark {

    private val list = (1..1000).toList()
    private val set = list.toHashSet()

    @Benchmark fun listContains(bh: Blackhole) = bh.consume(500 in list)
    @Benchmark fun setContains(bh: Blackhole) = bh.consume(500 in set)

    @Benchmark fun listIteration(bh: Blackhole) {
        var sum = 0L
        for (x in list) sum += x
        bh.consume(sum)
    }

    @Benchmark fun sequenceIteration(bh: Blackhole) {
        bh.consume(list.asSequence().sumOf { it.toLong() })
    }
}
```

Run and interpret: which operations are O(1), which are O(n)? Does `asSequence()` help or hurt for this size?

**Exercise 2: The cost of boxing**

```kotlin
@Benchmark fun intList(bh: Blackhole) {
    val list: List<Int> = (1..1000).toList()  // boxes each Int
    bh.consume(list.sum())
}

@Benchmark fun intArray(bh: Blackhole) {
    val arr = IntArray(1000) { it }  // primitive int[]
    bh.consume(arr.sum())
}
```

Measure the difference. Then verify with allocation profiling: `IntArray` should show near-zero allocations per iteration; `List<Int>` shows many `Integer` object allocations.

---

## Resources

- **Java Performance — Scott Oaks** — the definitive JVM performance reference
- **Optimizing Java — Benjamin Evans et al.** — more applied, GC-heavy
- [JVM Anatomy Quarks — Aleksey Shipilёv](https://shipilev.net/jvm/anatomy-quarks/) — required reading. Short, precise, correct.
- [async-profiler](https://github.com/async-profiler/async-profiler) — profiler
- [JMH samples](https://github.com/openjdk/jmh/tree/master/jmh-samples) — official benchmark examples
- [GCEasy](https://gceasy.io/) — GC log analysis
- [JDK Mission Control](https://adoptium.net/jmc/) — JFR viewer

---

*Notes and exercise solutions added below as completed.*
