---
layout: page
title: Kotlin
permalink: /languages/kotlin/
---

Kotlin compiles to JVM bytecode (and optionally native). Understanding Kotlin means understanding both the language and what it generates — the same discipline as understanding what CPython produces from Python source. Use IntelliJ's "Show Kotlin Bytecode" (Tools → Kotlin → Show Kotlin Bytecode → Decompile) constantly.

---

## 1. The Type System & Null Safety

Kotlin's type system makes null errors compile-time failures instead of runtime crashes. This is not just syntactic sugar — it changes how you design APIs.

### Core rules

```kotlin
var name: String = "Harris"   // non-null — compiler guarantees no null
var name: String? = null      // nullable — must check before use

// Safe call — returns null instead of throwing NPE
val length = name?.length      // Int?

// Elvis operator — provide a default
val length = name?.length ?: 0  // Int

// Not-null assertion — throws if null (use sparingly)
val length = name!!.length      // Int, throws NullPointerException if null

// Smart cast — after a null check, compiler knows the type
if (name != null) {
    println(name.length)   // name is String here, not String?
}
```

### Generics and variance

In Java, generics are invariant — `List<String>` is not a `List<Object>`. Kotlin has declaration-site variance:

```kotlin
// out — covariant (producer): can only produce T, not consume it
// equivalent to Java's List<? extends T>
interface Producer<out T> {
    fun produce(): T
}

// in — contravariant (consumer): can only consume T, not produce it
// equivalent to Java's List<? super T>
interface Consumer<in T> {
    fun consume(item: T)
}
```

### Exercises

**Exercise 1: Type system exploration**

```kotlin
// Predict: which of these compiles? Which throws at runtime?
val a: String? = null
val b: String = a!!         // ?
val c: String = a ?: "default"  // ?
val d: Int = a?.length ?: -1    // ?

// Smart cast puzzle
fun process(s: String?) {
    if (s != null) {
        println(s.length)  // smart cast — compiles?
    }
    val len = s?.let { it.length } ?: 0  // what type is len?
}
```

**Exercise 2: Variance — when does it matter?**

```kotlin
fun printAll(items: List<Any>) {
    items.forEach { println(it) }
}

val strings: List<String> = listOf("a", "b", "c")
printAll(strings)  // Does this work? Why? (hint: List<out T>)

// Now with MutableList
fun addItem(items: MutableList<Any>) {
    items.add(42)
}

val strings2: MutableList<String> = mutableListOf("a", "b")
// addItem(strings2)  // Does this compile? Why not?
```

Explain the difference. Decompile both to Java to see what the compiler generates.

---

## 2. Language Features — What They Compile To

The key discipline: always check the bytecode. Kotlin's conveniences often come free; some don't.

### Data classes

```kotlin
data class User(val id: Long, val name: String, val email: String)
```

Generates: `equals`, `hashCode`, `toString`, `copy`, `componentN` functions. All free — the compiler writes the boilerplate. Decompile to Java and read each generated method.

### Sealed classes

```kotlin
sealed class Result<out T> {
    data class Success<T>(val value: T) : Result<T>()
    data class Failure(val error: Throwable) : Result<Nothing>()
    object Loading : Result<Nothing>()
}
```

When used with `when`, the compiler enforces exhaustiveness — you can't forget a case. No `else` branch needed.

```kotlin
fun handle(result: Result<String>): String = when (result) {
    is Result.Success -> result.value
    is Result.Failure -> "Error: ${result.error.message}"
    Result.Loading -> "Loading..."
    // no else — compiler verifies all cases are covered
}
```

### Extension functions

```kotlin
fun String.isPalindrome(): Boolean = this == this.reversed()
fun List<Int>.secondOrNull(): Int? = if (size >= 2) this[1] else null
```

Extension functions are **static methods** with the receiver as the first parameter. They don't modify the class. They can't access private members. Decompile to Java: `isPalindrome(String $this$isPalindrome)`.

### Inline functions

```kotlin
inline fun <T> measureTime(block: () -> T): Pair<T, Long> {
    val start = System.nanoTime()
    val result = block()
    return result to (System.nanoTime() - start)
}
```

`inline` copies the function body at every call site. This eliminates: the function call overhead, and the lambda allocation (the `block: () -> T` lambda becomes inline code, not an object). Use for higher-order functions on hot paths.

`reified` type parameters only work with `inline`:

```kotlin
inline fun <reified T> isInstanceOf(value: Any): Boolean = value is T
// Without inline + reified: can't use `is T` — T is erased at runtime
```

### Exercises

**Exercise 1: Decompile everything**

Write each of the following, then use IntelliJ's bytecode decompiler to read the generated Java:

1. `data class Point(val x: Int, val y: Int)` — find `equals`, `hashCode`, `copy`
2. An `object` (Kotlin singleton) — how is the singleton enforced?
3. An `extension function` on `String` — find the static method signature
4. An `inline` higher-order function — confirm the body is inlined at the call site
5. A `sealed class` with a `when` expression — find the exhaustiveness check

For each: write one sentence explaining what the compilation strategy tells you about performance or API design.

**Exercise 2: Value classes — zero-cost wrappers**

```kotlin
@JvmInline
value class UserId(val id: Long)
@JvmInline
value class Email(val value: String)

data class User(val id: UserId, val name: String, val email: Email)
```

Value classes wrap a single property and are represented as the underlying type at runtime (no allocation). They add type safety at zero cost. Verify with JOL or async-profiler allocation profiling — `UserId` should not appear as a heap object.

**Exercise 3: Build a typed DSL**

```kotlin
@DslMarker
annotation class QueryDsl

@QueryDsl
class QueryBuilder {
    private var table: String = ""
    private val conditions = mutableListOf<String>()
    private val columns = mutableListOf<String>()

    fun from(table: String) { this.table = table }
    fun where(condition: String) { conditions.add(condition) }
    fun select(vararg cols: String) { columns.addAll(cols) }

    fun build(): String {
        val cols = if (columns.isEmpty()) "*" else columns.joinToString(", ")
        val where = if (conditions.isEmpty()) "" else " WHERE ${conditions.joinToString(" AND ")}"
        return "SELECT $cols FROM $table$where"
    }
}

fun query(block: QueryBuilder.() -> Unit): String =
    QueryBuilder().apply(block).build()

// Usage
val sql = query {
    from("users")
    select("id", "name", "email")
    where("age > 18")
    where("active = true")
}
// SELECT id, name, email FROM users WHERE age > 18 AND active = true
```

Extend it: add `limit`, `orderBy`, and type-safe column references. The `@DslMarker` annotation prevents calling outer DSL methods inside an inner DSL block.

---

## 3. Coroutines

Kotlin coroutines are the production concurrency model for Kotlin services. Understanding them means understanding both the API and what the compiler generates.

### The `suspend` keyword

A `suspend` function can pause execution without blocking a thread. It's a regular function that the compiler transforms into a **state machine** — each suspension point becomes a state.

```kotlin
suspend fun fetchUser(id: Long): User {
    val data = httpClient.get("/users/$id")  // suspend here
    return parseUser(data)                   // resume here
}
```

The compiler generates: a `Continuation<User>` parameter (the "callback"), a `label` field tracking which state we're in, and a `when(label)` dispatch. Decompile `fetchUser` to Java — you'll see exactly this.

### Structured concurrency

Coroutines must run in a `CoroutineScope`. When the scope is cancelled, all children are cancelled. This prevents leaks — you can't accidentally lose a coroutine the way you can lose a thread.

```kotlin
// Every coroutine launched here is a child of this scope
coroutineScope {
    val a = async { fetchUser(1) }
    val b = async { fetchUser(2) }
    // Both cancelled if either throws
    println(a.await() to b.await())
}  // all children complete before this returns
```

### Dispatchers

```kotlin
Dispatchers.Default    // CPU-bound work — thread pool sized to CPU cores
Dispatchers.IO         // I/O-bound work — larger thread pool (up to 64 threads)
Dispatchers.Main       // UI thread (Android, JavaFX)
Dispatchers.Unconfined // runs in calling thread until first suspension — use rarely
```

### `Flow` — cold, lazy streams

`Flow<T>` is a cold stream — it doesn't produce values until collected. Like Kotlin sequences but with suspension support.

```kotlin
fun numberStream(): Flow<Int> = flow {
    for (i in 1..1000) {
        delay(10)  // suspend
        emit(i)    // produce a value
    }
}

// Consume
numberStream()
    .filter { it % 2 == 0 }
    .map { it * it }
    .take(5)
    .collect { println(it) }
```

### `SharedFlow` / `StateFlow` — hot streams

`SharedFlow` is hot — values are emitted regardless of collectors. `StateFlow` is a `SharedFlow` that always holds the latest value (like a reactive state variable). Use for event broadcasting and state management.

### Exercises

**Exercise 1: Read the state machine**

Write a simple suspend function with two suspension points:

```kotlin
suspend fun twoSteps(): String {
    val a = suspendCoroutine<String> { it.resume("hello") }
    val b = suspendCoroutine<String> { it.resume("world") }
    return "$a $b"
}
```

Decompile to Java. Find: the `label` field, the `switch(label)` dispatch, the `Continuation.resumeWith()` calls. Draw the state machine as a diagram with 3 states (0 = start, 1 = after first suspend, 2 = done).

**Exercise 2: Structured cancellation — verify it**

```kotlin
suspend fun main() = coroutineScope {
    val job = launch {
        try {
            repeat(100) { i ->
                println("Child working $i")
                delay(100)
            }
        } finally {
            println("Child cancelled — cleaning up")
        }
    }

    delay(350)  // let it run for 3 iterations
    job.cancel()
    job.join()  // wait for cancellation to complete
    println("Done")
}
```

Verify: "Child cancelled — cleaning up" is printed. This proves `finally` runs on cancellation, which is how you release resources safely.

**Exercise 3: `Flow` backpressure — slow consumer**

```kotlin
val fastProducer = flow {
    repeat(1000) { i ->
        emit(i)
        // no delay — produces as fast as possible
    }
}

fastProducer
    .buffer(capacity = 10)         // buffer up to 10 items
    .collect { item ->
        delay(10)                  // slow consumer
        println("Consumed $item")
    }
```

Without `buffer()`: what happens? (The producer suspends at each `emit` until the consumer is ready — full backpressure.) With `buffer(10)`: the producer can run ahead by 10 items. Measure how long each takes. Then try `conflate()` — drops intermediate values when the consumer is slow.

**Exercise 4: `Flow` vs `Channel` — hot vs cold**

```kotlin
// Cold: Flow re-runs for each collector
val coldFlow = flow {
    println("Flow started")
    emit(1); emit(2); emit(3)
}

// Hot: Channel produces regardless of consumers
val channel = Channel<Int>(capacity = 3)

runBlocking {
    // Collect cold flow twice — "Flow started" prints twice
    coldFlow.collect { print("$it ") }
    coldFlow.collect { print("$it ") }

    // Send to channel — values sit in buffer
    channel.send(1); channel.send(2); channel.send(3)
    // Receive
    repeat(3) { print("${channel.receive()} ") }
}
```

Write up: when would you use `Flow` vs `Channel` in a real service? (Hint: Flow for data transformation pipelines; Channel for communication between producers and consumers that run concurrently.)

**Exercise 5: `Dispatcher` performance comparison**

```kotlin
suspend fun cpuWork(): Long {
    var sum = 0L
    for (i in 0 until 10_000_000) sum += i
    return sum
}

suspend fun main() {
    // CPU-bound: Default vs IO
    withContext(Dispatchers.Default) {
        val t0 = System.currentTimeMillis()
        (1..4).map { async { cpuWork() } }.awaitAll()
        println("Default: ${System.currentTimeMillis() - t0}ms")
    }

    withContext(Dispatchers.IO) {
        val t0 = System.currentTimeMillis()
        (1..4).map { async { cpuWork() } }.awaitAll()
        println("IO: ${System.currentTimeMillis() - t0}ms")
    }
}
```

Explain the results: `Dispatchers.IO` has more threads but they're still constrained by the JVM GIL... wait, there's no GIL in the JVM. So why might `Default` be better for CPU work? (Thread count — Default = CPU cores, IO = up to 64. More IO threads → more context switching overhead for CPU work.)

---

## 4. Performance Patterns

### Avoid unnecessary object creation on hot paths

```kotlin
// Bad: creates a lambda object on every call
fun process(items: List<Int>) = items.filter { it > 0 }.map { it * 2 }

// Better: use sequences to avoid intermediate list allocations
fun process(items: List<Int>) = items.asSequence().filter { it > 0 }.map { it * 2 }.toList()

// For very tight loops: use arrays and primitive types
fun processArray(items: IntArray): IntArray {
    var count = 0
    val result = IntArray(items.size)
    for (item in items) {
        if (item > 0) result[count++] = item * 2
    }
    return result.copyOf(count)
}
```

### `buildList` / `buildMap` — avoid temporary copies

```kotlin
// Bad: creates an intermediate mutable list, then an immutable copy
val result = mutableListOf<String>()
items.forEach { result.add(it.toString()) }
val immutable = result.toList()  // extra copy

// Better: buildList is the canonical way
val result = buildList {
    items.forEach { add(it.toString()) }
}
```

### Exercises

**Exercise: Sequence vs eager — benchmark intermediate allocations**

```kotlin
val data = (1..100_000).toList()

// Version A: eager (creates intermediate lists)
fun eager(data: List<Int>): List<Int> =
    data.filter { it % 2 == 0 }.map { it * it }.take(1000)

// Version B: lazy (no intermediate allocations)
fun lazy(data: List<Int>): List<Int> =
    data.asSequence().filter { it % 2 == 0 }.map { it * it }.take(1000).toList()
```

JMH benchmark both. Use async-profiler allocation mode to verify: `eager` shows `Integer[]` allocations for each intermediate list; `lazy` shows near-zero.

---

## Resources

- **Kotlin in Action, 2nd ed. — Jemerov & Isakova** — the authoritative book
- **Kotlin Coroutines Deep Dive — Marcin Moskala** — internals and patterns for coroutines
- [Roman Elizarov's blog](https://elizarov.medium.com/) — Roman designed coroutines; these posts explain the design decisions
- [Kotlin documentation — Coroutines guide](https://kotlinlang.org/docs/coroutines-guide.html)
- IntelliJ: Tools → Kotlin → Show Kotlin Bytecode → Decompile — use this constantly

---

*Notes and exercise solutions added below as completed.*
