# Full-Stack Interview Answers: 5–6 Years Experience

*Target stack: Java, Spring Boot, Node.js, React, SQL/NoSQL, system design, DevOps, security, and behavioral leadership*

This guide answers all 100 questions in [`full_stack_interview_questions.md`](full_stack_interview_questions.md). It is written as a preparation companion rather than a list of definitions: each answer explains the model to keep in your head, the trade-offs an experienced engineer should mention, and a practical example where one improves understanding.

## Coverage map

| Questions | Topic |
|---:|---|
| 1–20 | Java, concurrency, Spring Boot, and microservices |
| 21–33 | Node.js, Express, and NestJS |
| 34–46 | React, micro-frontends, and web performance |
| 47–56 | SOLID, design patterns, and machine coding |
| 57–74 | Distributed systems, messaging, transactions, and HLD |
| 75–83 | SQL, MongoDB, and Redis |
| 84–96 | Containers, Kubernetes, delivery, observability, and security |
| 97–100 | Behavioral and leadership answer frameworks |

## How to use this guide

1. Read the **interview-ready answer** first and practice saying it in 60–90 seconds.
2. Use the deeper sections to prepare for follow-up questions.
3. Re-type important code rather than memorizing it. Be ready to explain correctness, failure modes, and alternatives.
4. For design questions, state assumptions and requirements before proposing architecture. Interviewers care as much about reasoning as the final diagram.
5. Personalize the behavioral templates with your own metrics, decisions, mistakes, and lessons. Never present an illustrative story as your experience.

## Version and accuracy notes

- Runtime and framework behavior changes over time. Version-sensitive answers identify their scope where it matters.
- “Exactly once,” “zero data loss,” “non-blocking,” and “thread safe” are conditional claims. The relevant answer spells out the conditions rather than treating them as magic guarantees.
- Samples emphasize the core concept. Production code still needs organization-specific validation, observability, security review, load testing, and failure testing.
- References favor official specifications and project documentation. External links were selected for further study, not as substitutes for understanding the answer.

---

## Part 1: Java Core & Concurrency

### 1. Explain the internal working of `ConcurrentHashMap` in Java 8+. How does it achieve thread safety without locking the entire map?

**Interview-ready summary.** Java 8+ `ConcurrentHashMap` combines volatile-style visibility, compare-and-set (CAS), and very small critical sections. A read normally performs no locking. A write CASes directly into an empty bucket; when a bucket already contains entries, the writer synchronizes on that bucket's current head node rather than on the whole table. Heavy-collision buckets may become balanced trees, resizing is cooperative, and striped counters reduce contention when maintaining the approximate size.

**How it works.** The table is an array of bins. A bin is initially `null`, then usually a linked list of `Node<K,V>`, and may become a tree-backed `TreeBin` after enough collisions (subject to a minimum table capacity). Hash spreading uses high hash bits as well as low ones when selecting `(tableLength - 1) & hash`.

- `get(key)` reads the table/bin using memory-safe volatile access, compares the first node, and traverses a list or tree. It does not acquire a map-wide or bin lock. A completed update for a key *happens-before* a later non-null retrieval that observes it.
- `put` into an empty bin uses CAS. If another writer wins, the loser retries. For an occupied ordinary bin, the writer enters `synchronized (firstNode)`, rechecks that it is still the bin head, then changes only that bin. Thus writers to unrelated bins can proceed concurrently.
- A badly collided list can be treeified (commonly remembered thresholds: treeify at 8 nodes, but resize instead while the table is below 64 entries). A `TreeBin` provides its own coordination for tree updates.
- During resize, a bin is replaced by a forwarding marker. Threads encountering it can help transfer ranges of bins to the new table; lookups can follow the forwarding node. No single resizer must stop all access.
- The count uses a base plus striped counter cells, conceptually similar to `LongAdder`, so every update does not contend on one counter. Consequently, aggregate observations such as `size()` are not an atomic snapshot during concurrent mutation.

Compound actions must use the atomic map APIs. `if (!map.containsKey(k)) map.put(k, v)` is a race; `putIfAbsent`, `compute`, and `merge` express one per-key atomic operation. Also remember that callbacks passed to `compute*` should be short and must not recursively modify the same key.

```java
ConcurrentHashMap<String, LongAdder> counts = new ConcurrentHashMap<>();

void record(String route) {
    // Per-key initialization is atomic; LongAdder scales repeated increments.
    counts.computeIfAbsent(route, ignored -> new LongAdder()).increment();
}

long count(String route) {
    LongAdder adder = counts.get(route); // normally lock-free retrieval
    return adder == null ? 0 : adder.sum();
}
```

**Tradeoffs.** It is ideal for shared registries, caches, and counters with many reads and independent-key writes. It does not make the objects stored inside it thread-safe, does not provide a transaction across multiple keys, rejects `null` keys/values, and its iterators and aggregate methods provide concurrent views rather than a globally frozen snapshot.

**References:** [Java SE 25 `ConcurrentHashMap` API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html), [Java concurrent package memory-consistency guarantees](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/package-summary.html)

### 2. What is the difference between `reentrantLock` and a standard `synchronized` block? When would you prefer the former?

**Interview-ready summary.** The actual class is `ReentrantLock` (capitalized). Both it and `synchronized` are reentrant mutual-exclusion mechanisms with equivalent fundamental memory-visibility guarantees. Prefer `synchronized` for simple lexical critical sections because release is automatic. Prefer `ReentrantLock` when the algorithm genuinely needs timed/non-blocking acquisition, interruptible acquisition, fairness selection, multiple condition queues, lock-state instrumentation, or a lock whose lifetime cannot be expressed as one nested block.

With `synchronized`, a thread acquires an object's intrinsic monitor when entering the method/block and the JVM releases it on every exit, including exceptions. `wait`, `notify`, and `notifyAll` operate on the monitor's one wait set. The syntax is hard to misuse and modern JVMs optimize uncontended monitors well.

`ReentrantLock` is an explicit object. It offers:

- `tryLock()` and timed `tryLock(...)`, useful for avoiding indefinite waits or implementing backoff;
- `lockInterruptibly()`, so cancellation can interrupt a thread waiting for the lock;
- an optional approximately FIFO fairness policy (`new ReentrantLock(true)`), usually with lower throughput;
- multiple `Condition` objects, allowing separate queues such as `notEmpty` and `notFull`;
- diagnostic methods such as `isLocked()` and `getQueueLength()`; and
- the ability to acquire in one method and release in another—powerful, but easier to get wrong.

```java
final class Inventory {
    private final ReentrantLock lock = new ReentrantLock();
    private int available;

    boolean reserve(int quantity, Duration budget) throws InterruptedException {
        if (!lock.tryLock(budget.toMillis(), TimeUnit.MILLISECONDS)) {
            return false; // caller can degrade gracefully instead of hanging
        }
        try {
            if (available < quantity) return false;
            available -= quantity;
            return true;
        } finally {
            lock.unlock(); // always pair explicit acquisition with finally
        }
    }
}
```

Do not choose `ReentrantLock` merely because it sounds faster; benchmark the actual contention pattern. Never return or throw between `lock()` and the `try` block. Conditions, like monitor waits, must be checked in a `while` loop because wakeups do not prove the predicate is true.

**Version nuance.** Older Loom discussions said `synchronized` could pin a virtual thread to its carrier. JDK 24 delivered JEP 491, which changed HotSpot so ordinary monitor blocking no longer pins in the usual case. Native/foreign calls can still pin. Therefore choosing `ReentrantLock` solely to avoid `synchronized` pinning is outdated for JDK 24+, though it can matter on JDK 21–23.

**References:** [Java concurrency guide: locks versus intrinsic synchronization](https://docs.oracle.com/en/java/javase/25/core/concurrency.html), [Java SE 25 `ReentrantLock` API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/ReentrantLock.html), [JEP 491: Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491)

### 3. Explain the Java Memory Model (JMM). What are the roles of the Stack, Heap, Metaspace, and the volatile keyword in preventing instruction reordering?

**Interview-ready summary.** This question mixes two layers. The JMM is the language-level contract for visibility, ordering, atomicity, data races, and *happens-before*; stack, heap, and Metaspace are runtime storage regions. Those regions do not themselves prevent reordering. Synchronization actions—monitor unlock/lock, volatile write/read, thread start/join, and safe publication—establish the ordering and visibility guarantees.

**JMM.** Compilers, the JIT, and CPUs may reorder operations as long as single-thread behavior is preserved. When two threads perform conflicting accesses without a happens-before edge, the program has a data race and another thread may see stale or surprising values. Important edges include:

- earlier actions in one thread happen-before later actions in that thread;
- monitor unlock happens-before a subsequent lock of the same monitor;
- a write to a `volatile` variable happens-before every subsequent read of that same variable;
- actions before `Thread.start()` are visible to the started thread, and a thread's actions are visible after a successful `join()`;
- properly constructed `final` fields receive special safe-initialization guarantees if `this` does not escape during construction.

**Storage regions.** A Java thread's stack contains frames: local variables, operand state, and return information. Stacks are thread-confined, although a local variable can hold a reference to a shared heap object. The heap contains objects and arrays shared by references and managed by GC. Metaspace, in native memory since Java 8, primarily holds class metadata; repeated undeployments can exhaust it if class loaders remain reachable. JIT code cache, thread stacks, direct buffers, and native allocations are additional regions. These implementation areas help diagnose memory, but they are not the JMM.

**`volatile`.** A volatile read observes a suitably ordered volatile write and acts as an acquire; a write acts as a release. This restricts reorderings across that access and provides visibility. It does **not** turn a multi-step operation into one atomic transaction: `count++` is still read–modify–write and races. Use `AtomicInteger`, a lock, or confinement for that.

```java
final class SettingsHolder {
    private static volatile Settings instance;

    static Settings get() {
        Settings local = instance;
        if (local == null) {
            synchronized (SettingsHolder.class) {
                local = instance;
                if (local == null) {
                    local = new Settings();
                    instance = local; // volatile release: safely publishes fields
                }
            }
        }
        return local; // volatile acquire on the read path
    }
}
```

Without `volatile`, double-checked locking is invalid: publication may become observable before construction effects are safely visible. A simpler production choice is often an initialization-on-demand holder or enum singleton.

**References:** [JLS §17: Threads and Locks](https://docs.oracle.com/javase/specs/jls/se25/html/jls-17.html), [Java concurrent package happens-before summary](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/package-summary.html), [JVM Specification §2.5 runtime data areas](https://docs.oracle.com/javase/specs/jvms/se25/html/jvms-2.html#jvms-2.5)

### 4. How do Virtual Threads (Project Loom) differ fundamentally from platform threads regarding memory footprint and OS thread blocking?

**Interview-ready summary.** A platform thread is generally a thin Java wrapper over an OS thread and carries an OS-managed stack, so a very large thread-per-request population is expensive. A virtual thread is scheduled by the JVM, has stack chunks that grow/shrink in the heap, and is mounted on one of a much smaller set of platform *carrier* threads. When supported blocking I/O parks a virtual thread, the JVM normally unmounts it, freeing the carrier to run another virtual thread.

Virtual threads became final in Java 21. They preserve the familiar sequential style—blocking calls, exceptions, stack traces, and thread-local APIs—while making high-concurrency, mostly-waiting workloads practical. They do not make code execute faster and do not add CPU cores. One million CPU-bound virtual threads only create scheduling overhead; CPU parallelism remains bounded by available processors.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<Response>> calls = requests.stream()
        .map(request -> executor.submit(() -> blockingHttpCall(request)))
        .toList();

    for (Future<Response> call : calls) {
        consume(call.get()); // structured lifecycle; executor closes after tasks
    }
}
```

**Blocking behavior.** Network I/O and many JDK blocking operations can park/unmount a virtual thread. The carrier is not the identity of the virtual thread; thread-local state belongs to the virtual thread. A virtual thread can be *pinned* while executing a native method or foreign function, which prevents carrier reuse during that interval. On JDK 21–23, blocking while holding an intrinsic monitor was another important pinning case; JDK 24's JEP 491 removed that usual `synchronized` limitation. Use JFR's `jdk.VirtualThreadPinned` event to find remaining long pins.

**Practical implications.** Use virtual threads for request-per-task services with blocking JDBC/HTTP/file calls, especially when concurrency is high and wait time dominates. Do not pool virtual threads; create one per task and constrain the scarce downstream resource instead—for example, a semaphore or a correctly sized database connection pool. Avoid storing large objects in `ThreadLocal` when creating huge numbers of virtual threads. Reactive APIs may still win when end-to-end backpressure, streaming composition, or a mature reactive ecosystem is central; virtual threads win on simpler imperative control flow.

Virtual threads are daemon threads, so they do not by themselves keep the JVM alive. Also audit libraries that assume a small fixed population of threads or use thread identity as a resource-pool key.

**References:** [Oracle Java 25 virtual-thread guide](https://docs.oracle.com/en/java/javase/25/core/virtual-threads.html), [Java SE 25 `Thread` API: platform and virtual threads](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/lang/Thread.html), [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)

### 5. How would you diagnose a memory leak in a production Java application? Which tools (e.g., JProfiler, VisualVM, Eclipse MAT) and JVM flags would you use?

**Interview-ready summary.** First prove which memory is growing—Java heap, Metaspace/class loaders, direct/native memory, thread stacks, or the container's page cache. Then capture low-overhead time-series evidence, take a heap dump or native-memory baseline at the right moment, identify the GC root retaining the growth, reproduce if possible, fix ownership/lifecycle, and verify that the post-GC baseline stops rising.

**Production workflow.** Start with application/container RSS, `jvm.memory.*`, allocation rate, old-generation occupancy *after full/concurrent collection*, GC pause/frequency, class count, thread count, and direct-buffer metrics. A sawtooth heap is normal; a leak is suggested when the trough after collection rises under comparable load. Check recent releases and traffic changes before forcing GC or attaching a heavy profiler.

Useful startup options on a supported HotSpot release include:

```text
-Xms2g -Xmx2g
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/crash/my-service
-Xlog:gc*,safepoint:file=/var/log/my-service/gc.log:time,uptime,level,tags
-XX:StartFlightRecording=filename=/var/log/my-service/app.jfr,settings=profile,maxage=2h,maxsize=512m
-XX:NativeMemoryTracking=summary
```

Ensure dump paths have enough disk and restricted permissions: heap dumps contain credentials and customer data. Capturing a live dump can pause the application and may temporarily require memory comparable to the heap, so use a canary/replica or controlled traffic drain when possible.

At runtime:

```bash
jcmd <pid> GC.class_histogram
jcmd <pid> GC.heap_dump /secure/path/heap-1.hprof
jcmd <pid> JFR.dump filename=/secure/path/incident.jfr path-to-gc-roots=true
jcmd <pid> VM.native_memory baseline
# Later:
jcmd <pid> VM.native_memory summary.diff
```

Open `.hprof` in Eclipse MAT and inspect the leak-suspect report, dominator tree, retained (not merely shallow) size, and paths to GC roots. Compare two snapshots: collections whose retained population grows are more persuasive than one large object. Typical roots are static maps/caches, listeners never removed, `ThreadLocal` values on pooled threads, pending futures/queues, class loaders retained after redeploy, and unclosed native/direct resources. VisualVM is useful for monitoring, histograms, sampling, and local/test snapshots; JProfiler adds polished allocation/call tracking and heap analysis. JFR/JDK Mission Control is generally a strong first production profiler because overhead is designed to be low.

If RSS grows while Java heap is flat, inspect NMT categories, direct buffers, thread count/stack sizing, JNI libraries, memory-mapped files, and allocator fragmentation. NMT tracks HotSpot allocations, not all third-party native allocations, so OS tools may still be required.

Finally, write a load/regression test, bound caches/queues, close or deregister resources, clear thread locals in `finally`, repeat snapshots under the same workload, and verify both retained objects and post-GC baseline stabilize.

**References:** [Oracle: Troubleshoot Memory Leaks](https://docs.oracle.com/en/java/javase/25/troubleshoot/troubleshooting-memory-leaks.html), [Oracle diagnostic tools and `jcmd`](https://docs.oracle.com/en/java/javase/25/troubleshoot/diagnostic-tools.html), [Oracle Native Memory Tracking](https://docs.oracle.com/en/java/javase/25/vm/native-memory-tracking.html)

### 6. Explain the difference between G1 GC and ZGC. In what scenarios would you optimize for low latency versus high throughput?

**Interview-ready summary.** G1 is the default mostly concurrent, region-based collector on most HotSpot configurations and aims for a strong latency/throughput balance with configurable pause goals. ZGC performs almost all expensive work concurrently and is designed for extremely short pauses even on very large heaps, at the cost of additional CPU/memory overhead and sometimes lower peak throughput. Measure service-level objectives rather than choosing from heap size alone.

**G1.** It divides the heap into equal regions, tracks cross-region references, performs concurrent marking, and evacuates selected regions during stop-the-world young and mixed collections. “Garbage First” means it preferentially collects regions expected to reclaim the most space. `-XX:MaxGCPauseMillis=200` is a goal, not a guarantee. G1 is often a good default for ordinary APIs, batch systems with some latency sensitivity, moderate-to-large heaps, and environments where CPU efficiency matters.

**ZGC.** Modern ZGC uses generations (non-generational ZGC was removed in JDK 24), colored/metadata-bearing references and load barriers so marking, relocation, and reference processing can occur mostly concurrently. Pause time is intended to remain very small and largely independent of heap size or live-set size. It needs enough CPU headroom for concurrent GC and enough heap headroom so allocation does not outrun reclamation.

```bash
# Start with an explicit heap and otherwise modest tuning.
java -Xms8g -Xmx8g -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=100 -Xlog:gc* -jar app.jar

# JDK 24+: ZGC is generational; no separate generational flag is needed.
java -Xms8g -Xmx8g -XX:+UseZGC \
     -Xlog:gc* -jar app.jar
```

Choose **low latency** for interactive trading, ad bidding, games, or request services whose p99/p99.9 response budget cannot tolerate tens-of-milliseconds GC pauses. ZGC is a candidate if the host has resource headroom. Choose **throughput** for offline ETL, report generation, compilation, or cost-sensitive batch processing where completing total work fastest matters more than an individual pause; G1 may be preferable, and `ParallelGC` should also be benchmarked for a pure throughput target.

Compare representative load tests with allocation rate, live set, p50/p99/p99.9 latency, GC CPU, pause distribution, reclaimed bytes, and container memory. Do not hide a memory leak by increasing `-Xmx`. Avoid copying folklore flags across JDK versions: collectors and defaults evolve, and excessive tuning can fight collector ergonomics.

**References:** [Oracle Java 25 available collectors](https://docs.oracle.com/en/java/javase/25/gctuning/available-collectors.html), [Oracle G1 collector guide](https://docs.oracle.com/en/java/javase/25/gctuning/garbage-first-g1-garbage-collector1.html), [JEP 474: ZGC Generational Mode by Default](https://openjdk.org/jeps/474)

### 7. How does CompletableFuture handle asynchronous exception chaining? Walk through `exceptionally()`, `handle()`, and `whenComplete()`.

**Interview-ready summary.** An exception in a stage normally makes dependent stages complete exceptionally, usually surfaced from `join()` as a `CompletionException`. `exceptionally` is catch-and-recover only on failure; `handle` runs on success or failure and maps either outcome to a new value; `whenComplete` observes either outcome for side effects and normally preserves it. The placement of recovery in the chain changes which later stages see a value versus an exception.

```java
CompletableFuture<Order> result = CompletableFuture
    .supplyAsync(() -> loadOrder("o-42"), ioExecutor)
    .thenApply(this::validate) // skipped if loadOrder failed
    .whenComplete((order, error) -> { // observe; do not recover
        if (error == null) metrics.success();
        else metrics.failure(unwrap(error));
    })
    .exceptionally(error -> { // runs only if any preceding stage failed
        logger.warn("Returning fallback", unwrap(error));
        return Order.unavailable("o-42"); // recovered normal result
    });

static Throwable unwrap(Throwable error) {
    return error instanceof CompletionException && error.getCause() != null
        ? error.getCause() : error;
}
```

- `exceptionally(Function<Throwable, ? extends T>)` is analogous to `catch`: it is skipped on success and must return the same result type. If its function throws, the returned stage remains exceptional. Newer JDKs also have `exceptionallyCompose` for an asynchronous fallback without creating a nested future.
- `handle(BiFunction<T, Throwable, U>)` always runs. Exactly one of value/error is normally non-null (a successful value itself may legally be `null`). Its return value completes the next stage, so it can recover, normalize success, or change type. If it throws, the new stage fails.
- `whenComplete(BiConsumer<T, Throwable>)` always runs but is designed for logging, metrics, tracing, and cleanup. It returns no replacement value. If the source succeeded and the observer throws, the returned stage fails with the observer's error. If the source already failed and the observer also throws, the original exceptional completion takes precedence under `CompletionStage` rules.

Non-`Async` callbacks can execute on the thread that completes the previous stage (or a caller that notices completion). `*Async` without an executor normally uses `ForkJoinPool.commonPool()`, which is a poor place for unbounded blocking I/O. Pass a named, bounded executor and propagate tracing/security context intentionally.

Cancellation is represented as exceptional completion (`CancellationException`). `get()` wraps failures in checked `ExecutionException`; `join()` uses unchecked `CompletionException`. Test the entire graph, including timeouts and fallbacks, and avoid a final `exceptionally` that converts programming bugs into plausible business responses.

**References:** [Java SE 25 `CompletionStage` exception rules](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/CompletionStage.html), [Java SE 25 `CompletableFuture` API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/CompletableFuture.html)

### 8. What is the difference between `fail-fast` and `fail-safe` iterators? Provide internal structural examples.

**Interview-ready summary.** A fail-fast iterator detects certain structural changes to a normal collection and throws `ConcurrentModificationException` on a best-effort basis. “Fail-safe” is not an official Java Collections term; interviewers usually mean a snapshot iterator such as `CopyOnWriteArrayList` or a weakly consistent iterator such as `ConcurrentHashMap`, neither of which throws merely because the collection changes concurrently.

An `ArrayList`/`HashMap` maintains a structural modification count (`modCount`). Its iterator captures `expectedModCount`. On operations such as `next()` or iterator removal, a mismatch can trigger `ConcurrentModificationException`. Structural means adding/removing entries or otherwise changing iteration shape; replacing an existing value may not count. The iterator's own supported `remove()` updates its expected count. This mechanism detects bugs—it is not synchronization, is not guaranteed to detect every race, and must never be used for correctness.

```java
List<String> names = new ArrayList<>(List.of("a", "b", "c"));
for (String name : names) {
    // names.remove(name); // wrong: likely ConcurrentModificationException
}

for (Iterator<String> it = names.iterator(); it.hasNext();) {
    if (it.next().equals("b")) it.remove(); // supported iterator mutation
}
```

`CopyOnWriteArrayList` publishes a fresh backing array for every mutation. An iterator keeps the array reference captured when it was created, so it sees an immutable snapshot and never sees later changes. Iteration is stable and lock-free, but writes copy the entire array, iterator mutation is unsupported, and old arrays remain reachable while old iterators live. It fits small, read-mostly listener/configuration lists—not high-write data.

`ConcurrentHashMap` iterators are *weakly consistent*: they tolerate concurrent updates, never throw `ConcurrentModificationException` for them, and may reflect some updates that happened after iterator creation but need not reflect all. They do not duplicate an element and are designed for useful concurrent traversal, not a point-in-time report.

```java
var listeners = new CopyOnWriteArrayList<String>(); // snapshot iteration
var sessions = new ConcurrentHashMap<String, Session>(); // weakly consistent

sessions.forEach((id, session) -> sample(session));
// Safe during updates, but not an atomic snapshot across all sessions.
```

If business logic needs a coherent snapshot, take an application-level lock, copy under the required consistency boundary, or query a datastore with appropriate transaction isolation. Merely switching to a concurrent collection prevents structural corruption; it does not invent cross-element invariants.

**References:** [Java SE 25 `ConcurrentModificationException` contract](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/ConcurrentModificationException.html), [Java SE 25 `CopyOnWriteArrayList` snapshot semantics](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/CopyOnWriteArrayList.html), [Java SE 25 `ConcurrentHashMap` iterator semantics](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)

### 9. How do you implement a custom ThreadPoolExecutor? How do you choose the ideal core pool size, max pool size, and queue capacity for an I/O-bound vs. CPU-bound application?

**Interview-ready summary.** Construct `ThreadPoolExecutor` explicitly with a bounded queue, named threads, an intentional rejection policy, metrics, and graceful shutdown. Know its admission order: create threads up to `corePoolSize`; then queue; only when the queue is full create up to `maximumPoolSize`; then reject. Therefore an unbounded queue makes `maximumPoolSize` effectively irrelevant.

```java
int cores = Runtime.getRuntime().availableProcessors();
AtomicInteger sequence = new AtomicInteger();

ThreadFactory factory = task -> {
    Thread t = new Thread(task, "pricing-" + sequence.incrementAndGet());
    t.setUncaughtExceptionHandler((thread, error) -> log.error("worker failed", error));
    return t;
};

ThreadPoolExecutor pool = new ThreadPoolExecutor(
    cores,                         // core workers
    cores,                         // CPU work: keep near available cores
    30, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(500), // explicit memory/latency bound
    factory,
    new ThreadPoolExecutor.CallerRunsPolicy() // slows producer under overload
);
pool.prestartAllCoreThreads();

// Observe pool.getActiveCount(), getQueue().size(), completed/rejected tasks.
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    pool.shutdown();
    try {
        if (!pool.awaitTermination(20, TimeUnit.SECONDS)) pool.shutdownNow();
    } catch (InterruptedException e) {
        pool.shutdownNow();
        Thread.currentThread().interrupt();
    }
}));
```

For CPU-bound independent tasks, start near `Ncpu` (sometimes `Ncpu + 1` if occasional stalls occur) and benchmark. More runnable threads than cores add context switching and cache contention. For a mixed workload, a useful starting model is `Nthreads = Ncpu × targetUtilization × (1 + waitTime / computeTime)`. For blocking I/O this can be much larger, but must be capped by downstream connection pools, rate limits, memory, and latency objectives—not chosen from CPU count alone. In Java 21+, one virtual thread per blocking task can replace a large I/O worker pool; still bound the database/API with a semaphore or connection pool.

Queue capacity is a latency and memory budget. Estimate `acceptableQueueWait × sustainableCompletionRate`, validate with load tests, and account for bytes retained by each queued task. A huge queue converts overload into timeouts and heap pressure. A tiny queue with a larger max pool favors thread growth; a `SynchronousQueue` hands work directly to threads and can grow aggressively. Rejection should be observable: `AbortPolicy` fails fast; `CallerRunsPolicy` provides local feedback but must not block an event-loop/request thread unexpectedly; discard policies are only acceptable when loss is explicit.

Separate pools for materially different workloads so slow reports cannot starve login work. Apply deadlines/cancellation, never hold locks while submitting to a pool that might use caller-runs, and expose active threads, queue depth/age, task duration, rejection count, and downstream saturation.

**References:** [Java SE 25 `ThreadPoolExecutor` queuing and sizing rules](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html), [Java SE 25 executor APIs](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/Executors.html)

### 10. Explain the structural behavior of ThreadLocal variables. How can they introduce memory leaks in an application server like Tomcat?

**Interview-ready summary.** A `ThreadLocal` is not a map owned by the `ThreadLocal`; each `Thread` owns a `ThreadLocalMap`. Entries weakly reference the `ThreadLocal` key but strongly reference its value. If the key is collected without `remove()`, the value can remain until that long-lived thread's map happens to expunge the stale entry. In a Tomcat worker pool, the thread can outlive the request and even a web-app redeployment, retaining request data or the old application class loader.

Conceptually, `threadLocal.set(value)` finds the current thread, then inserts `(weak key, strong value)` into that thread's private map. Isolation avoids synchronization, but lifecycle follows the thread unless application code removes the value. A weak key does **not** make the value weak. Cleanup is opportunistic during later map operations; an idle pooled thread may retain it for a long time.

Leaks have two forms:

1. **Per-request contamination:** request A sets a user/tenant and forgets cleanup; request B reusing the same worker can observe it—a security bug as well as retained memory.
2. **Class-loader retention:** the value, key class, or object graph was loaded by a deployed web application's class loader. Tomcat keeps the worker thread, which keeps the value, which keeps the old class loader and all its classes after redeploy. Repeating redeploys can exhaust Metaspace/heap.

```java
final class RequestContext {
    private static final ThreadLocal<Context> CURRENT = new ThreadLocal<>();

    static void runWith(Context context, Runnable action) {
        CURRENT.set(context);
        try {
            action.run();
        } finally {
            CURRENT.remove(); // do not merely set(null); remove the entry
        }
    }
}

// A servlet Filter should use the same try/finally around chain.doFilter(...).
```

Clear *every* library context you establish (logging MDC, security, tracing, tenant, locale) on all outcomes. Prefer explicit parameters where practical. Do not assume `CompletableFuture` work sees the request thread's locals; context must be captured/propagated deliberately, then cleared on the executor thread. `InheritableThreadLocal` is usually more dangerous with pools because inheritance happens when the thread is created, not per submitted task.

Tomcat includes a `ThreadLocalLeakPreventionListener` that renews executor threads as a context stops, but it is defense in depth, not a substitute for correct application cleanup. Virtual threads reduce cross-request reuse when there is one thread per request, yet retaining large per-thread values across millions of virtual threads can still consume substantial memory.

**References:** [Java SE 25 `ThreadLocal` API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/lang/ThreadLocal.html), [Oracle virtual-thread guide: thread-local lifetime](https://docs.oracle.com/en/java/javase/25/core/virtual-threads.html), [Tomcat 11 ThreadLocal Leak Prevention Listener](https://tomcat.apache.org/tomcat-11.0-doc/config/listeners.html#ThreadLocal_Leak_Prevention_Listener_-_org.apache.catalina.core.ThreadLocalLeakPreventionListener)

## Part 1: Spring Boot & Microservices

### 11. Explain the Lifecycle of a Spring Bean from instantiation to destruction. Where do `BeanPostProcessor` and `@PostConstruct` fit?

**Interview-ready summary.** For an ordinary singleton, Spring reads/merges the bean definition, instantiates the object, injects dependencies, invokes awareness callbacks, passes it through `BeanPostProcessor` callbacks around initialization, and finally exposes the ready bean—often as a proxy. At context shutdown it invokes destruction-aware post-processors, `@PreDestroy`, `DisposableBean.destroy()`, and configured destroy methods. `@PostConstruct` is an initialization callback invoked by a post-processor before `postProcessAfterInitialization`.

A useful detailed sequence is:

1. **Definition and instantiation:** `BeanFactoryPostProcessor`s may first change bean *definitions*. The container selects a constructor or factory method and creates the raw instance. `InstantiationAwareBeanPostProcessor` hooks can participate even around this stage.
2. **Dependency population:** properties/fields/setters are resolved and injected. Constructor dependencies were necessarily resolved before construction.
3. **Aware callbacks:** applicable interfaces receive infrastructure, for example `BeanNameAware`, `BeanClassLoaderAware`, `BeanFactoryAware`, and application-context awareness through corresponding processors.
4. **Before initialization:** every registered `BeanPostProcessor.postProcessBeforeInitialization` runs in order. Spring's annotation processor invokes `@PostConstruct` here.
5. **Initialization callbacks:** for distinct method names the conventional order is `@PostConstruct`, `InitializingBean.afterPropertiesSet()`, then a configured custom `initMethod`. Prefer `@PostConstruct` or a custom method to avoid coupling domain code to Spring.
6. **After initialization:** `postProcessAfterInitialization` runs. Auto-proxy creators commonly return an AOP proxy here, so callers may receive a different object reference that delegates to the target.
7. **Use and destruction:** singleton beans remain managed until context shutdown. Prototype creation gets initialization callbacks, but Spring does not automatically manage a prototype's full destruction lifecycle.

```java
@Component
final class PriceClient {
    private final HttpClient http;

    PriceClient(HttpClient http) { this.http = http; } // instantiate + inject

    @PostConstruct
    void validateConfiguration() { /* fail startup if configuration is invalid */ }

    @PreDestroy
    void close() { /* close resources owned by this bean */ }
}
```

Do not perform slow remote calls casually in `@PostConstruct`: they lengthen or destabilize startup. Also do not expect proxy-based advice such as `@Transactional` to work reliably from initialization callbacks—the proxy may not yet be the active call boundary. Destruction callbacks are guaranteed for an orderly context close, not for `kill -9` or machine loss, so externally durable correctness cannot depend solely on them.

**References:** [Spring bean lifecycle callbacks](https://docs.spring.io/spring-framework/reference/core/beans/factory-nature.html), [Spring `BeanPostProcessor` extension point](https://docs.spring.io/spring-framework/reference/core/beans/factory-extension.html)

### 12. What is Bean Circular Dependency in Spring? How does Spring resolve it using three-level caches, and when does it fail?

**Interview-ready summary.** A circular dependency is a graph cycle such as A → B → A. Spring Framework can sometimes resolve a cycle between singleton beans when at least one dependency is injected after construction, by exposing an early reference through three singleton caches. It cannot construct a pure constructor cycle, and modern Spring Boot disables circular references by default. The best fix is normally to redesign the ownership boundary rather than enable them.

The often-discussed three levels in `DefaultSingletonBeanRegistry` are:

1. `singletonObjects`: fully initialized singleton instances;
2. `earlySingletonObjects`: materialized early references to beans still being created; and
3. `singletonFactories`: `ObjectFactory<?>` instances capable of creating an early reference, including an early AOP proxy when necessary.

For a setter/field cycle, Spring can instantiate A, register A's singleton factory, begin populating A, and discover B. It instantiates B and, while injecting B's A dependency, asks for A. Since A is “currently in creation,” Spring obtains the early reference from level 3, moves it to level 2, completes B, completes A, and finally promotes A to level 1 while clearing its early entries. The third level matters because it gives proxy-producing post-processors a chance to expose the same proxy identity early instead of injecting a raw target that later becomes a proxy.

```java
// Better: make the cycle explicit and lazy only as a migration step.
@Service
final class ReportService {
    private final ObjectProvider<NotificationService> notifications;

    ReportService(ObjectProvider<NotificationService> notifications) {
        this.notifications = notifications;
    }

    void complete() {
        // Resolved only when the operation actually needs it.
        notifications.getObject().sendReportReady();
    }
}
```

Resolution fails for two constructors that require each other because neither raw object exists to expose. It can also fail for prototype cycles, factory-method cycles, disabled circular-reference support, or proxy/initialization combinations where an injected raw early reference would not match the final wrapped bean. `@Lazy` or `ObjectProvider` can break construction timing, but may merely hide a flawed design.

Prefer one-direction dependencies, extract the shared responsibility into a third service, publish an application event for a genuinely asynchronous relationship, or pass data rather than calling back into the origin service. Since Spring Boot 2.6, `spring.main.allow-circular-references` defaults to `false`; turning it on is a temporary compatibility choice and still cannot solve every cycle. Constructor injection is valuable partly because it makes invalid dependency graphs fail clearly at startup.

**References:** [Spring dependency-injection documentation on circular dependencies](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html), [Spring Framework `DefaultSingletonBeanRegistry` source](https://github.com/spring-projects/spring-framework/blob/main/spring-beans/src/main/java/org/springframework/beans/factory/support/DefaultSingletonBeanRegistry.java)

### 13. Explain `@Transactional` propagation behaviors. What happens when a method with `REQUIRED` calls a method with `REQUIRES_NEW` or `NESTED`?

**Interview-ready summary.** `REQUIRED` joins an existing transaction or creates one. `REQUIRES_NEW` always starts an independent physical transaction, suspending the outer one. `NESTED` normally uses a savepoint inside the same physical transaction, so the inner scope can roll back to that savepoint but the outer transaction ultimately controls the physical commit. These semantics apply only when the inner call is intercepted by Spring's transactional infrastructure.

With outer `REQUIRED` → inner `REQUIRED`, both scopes share resources and commit together. If the inner scope marks the transaction rollback-only and the outer catches the exception and tries to commit, Spring raises `UnexpectedRollbackException`; it will not pretend that a commit occurred.

With outer `REQUIRED` → inner `REQUIRES_NEW`, Spring suspends the outer transaction's resources, obtains a new transaction (often another JDBC connection), runs and commits/rolls it back independently, then resumes the outer transaction. An audit record committed in the inner transaction can therefore survive a later outer rollback. Size the connection pool carefully: concurrent outer transactions can each hold one connection while waiting for another, causing pool exhaustion. The Spring documentation recommends exceeding concurrent outer threads by at least one connection when using this pattern, but real sizing must cover all nested concurrency.

With outer `REQUIRED` → inner `NESTED`, Spring creates a JDBC savepoint. Inner failure can roll back work after that savepoint, and the outer can continue. But if the outer later rolls back, nested work rolls back too. `NESTED` is typically supported by `DataSourceTransactionManager` with savepoint-capable JDBC; it is not universally equivalent in JPA/JTA transaction managers. With no existing transaction it behaves like `REQUIRED`.

```java
@Service
final class CheckoutService {
    private final AuditService audit; // separate bean ensures a proxy boundary

    @Transactional(propagation = Propagation.REQUIRED)
    public void checkout(Order order) {
        reserveStock(order);
        audit.recordAttempt(order.id()); // independent transaction
        charge(order);
    }
}

@Service
final class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void recordAttempt(UUID orderId) { /* insert audit row */ }
}
```

Also discuss rollback rules in interviews: by default Spring rolls back on unchecked `RuntimeException`/`Error`, not every checked exception; configure `rollbackFor` when the business contract requires it. Avoid `REQUIRES_NEW` as a way to paper over unclear consistency. For cross-service work, these are local transaction semantics—not distributed transactions.

**References:** [Spring transaction propagation: REQUIRED, REQUIRES_NEW, NESTED](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html), [Spring `@Transactional` settings](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)

### 14. How does transactional proxying fail due to self-invocation within the same Spring Bean? How do you fix it?

**Interview-ready summary.** In Spring's default proxy mode, a caller must invoke the bean *through its proxy* for `TransactionInterceptor` to open/commit/roll back a transaction. A call such as `this.saveOne()` is a direct Java call on the target and never crosses the proxy, so annotations on `saveOne` are ignored for that call.

```java
@Service
class ImportService {
    public void importAll(List<Item> items) {
        for (Item item : items) {
            saveOne(item); // self-invocation: no REQUIRES_NEW interception
        }
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveOne(Item item) { /* ... */ }
}
```

The proxy wraps `ImportService`; an external `proxy.saveOne()` enters advice and then calls `target.saveOne()`. Once code is already inside the target, `this` is the target, not a magic re-entry through the proxy. The same trap affects other proxy-based annotations such as `@Async`, `@Cacheable`, and method security. A private method cannot be overridden by a class proxy; interface proxies require interface-visible public methods. Spring 6 can advise protected/package-visible methods with class-based proxies, but only an external proxy call is intercepted.

**Preferred fix—refactor the boundary:** move the transactional operation into another bean whose public method is called through injection.

```java
@Service
final class ItemWriter {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveOne(Item item) { /* repository.save(item) */ }
}

@Service
final class ImportService {
    private final ItemWriter writer;
    ImportService(ItemWriter writer) { this.writer = writer; }

    public void importAll(List<Item> items) {
        items.forEach(writer::saveOne); // crosses ItemWriter proxy
    }
}
```

Alternatives are `TransactionTemplate` for an explicit programmatic boundary, AspectJ compile/load-time weaving (which modifies bytecode and handles self-invocation), or injecting a lazy self-proxy. Calling `AopContext.currentProxy()` requires proxy exposure, couples the class to AOP, and is usually the least maintainable option. Self-injection can also reintroduce circular dependencies.

Verify behavior with an integration test that asserts commit/rollback, not merely by checking that an annotation exists. Remember that calling an advised method from a constructor or `@PostConstruct` is too early to rely on a fully initialized proxy.

**References:** [Spring `@Transactional` proxy-mode and self-invocation rules](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html), [How Spring declarative transaction proxies work](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-decl-explained.html)

### 15. Explain the dynamic architectural differences between Spring Cloud Gateway and Netflix Zuul.

**Interview-ready summary.** Version scope is essential. The common comparison is Spring Cloud Gateway's WebFlux/Netty architecture versus **Zuul 1** in the historical Spring Cloud Netflix stack, which used a Servlet/blocking request model. Netflix's **Zuul 2** is itself Netty/event-loop based and non-blocking, so saying “Zuul is blocking” without a version is inaccurate. Also, current Spring Cloud Gateway 5 provides both WebFlux and Web MVC server variants.

Spring Cloud Gateway is integrated with Spring configuration, Reactor, Spring Security, Actuator/Micrometer, route predicates, discovery, and ordered pre/post filter chains. In the WebFlux variant, a small number of event-loop threads can multiplex many connections; filters return reactive publishers and must not perform blocking JDBC/HTTP work on those threads. Its MVC variant is appropriate when the application intentionally uses Servlet/blocking libraries.

Historical Spring Cloud Netflix Zuul wrapped Zuul 1 and ran servlet filters such as pre, route, post, and error filters. A request occupied a container thread while blocking downstream I/O, which is straightforward but consumes more threads under slow fan-out. That Spring Cloud integration is maintenance-era/legacy for new architecture decisions.

Zuul 2, maintained by Netflix, runs a Netty server/client with inbound, endpoint, and outbound filters. Its own documentation explicitly warns against blocking the event loop and provides async-filter patterns. It is not simply a drop-in continuation of Spring Cloud Netflix Zuul 1 and has a different integration/ecosystem story.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: catalog
          uri: lb://catalog-service
          predicates:
            - Path=/catalog/**
          filters:
            - StripPrefix=1
            - name: CircuitBreaker
              args:
                name: catalog
                fallbackUri: forward:/fallback/catalog
```

For a new Spring estate, Gateway is usually the natural choice because configuration, security, metrics, discovery, and resilience share the Spring programming model. Choose the WebFlux variant only if the filter chain and clients remain non-blocking; one blocking authentication lookup can stall many requests. Evaluate streaming/WebSocket support, route-update mechanism, retry semantics, observability, memory, p99 latency, team expertise, and project maintenance status—not just requests per second.

**References:** [Current Spring Cloud Gateway overview and variants](https://docs.spring.io/spring-cloud-gateway/reference/), [Spring Cloud Gateway WebFlux filter-chain behavior](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/), [Netflix Zuul 2 architecture](https://github.com/Netflix/zuul/wiki/How-It-Works-2.0)

### 16. How do you configure a resilient Circuit Breaker using Resilience4j? Explain the transitions between Closed, Open, and Half-Open states.

**Interview-ready summary.** A circuit breaker measures recent call outcomes. In **CLOSED**, calls flow and outcomes fill a count- or time-based window. Once the minimum sample is met and failure-rate or slow-call-rate crosses its threshold, it moves to **OPEN** and fails fast. After the open wait duration, **HALF_OPEN** admits a limited number of probes; enough successful probes close it, while a failed/slow threshold opens it again.

```yaml
resilience4j:
  circuitbreaker:
    instances:
      inventory:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 50
        minimumNumberOfCalls: 20
        failureRateThreshold: 50
        slowCallDurationThreshold: 800ms
        slowCallRateThreshold: 60
        waitDurationInOpenState: 15s
        permittedNumberOfCallsInHalfOpenState: 5
        automaticTransitionFromOpenToHalfOpenEnabled: true
        recordExceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
        ignoreExceptions:
          - com.example.catalog.ProductNotFoundException
  timelimiter:
    instances:
      inventory:
        timeoutDuration: 1s
```

```java
@Service
class InventoryClient {
    @CircuitBreaker(name = "inventory", fallbackMethod = "fallback")
    public Stock getStock(String sku) {
        return remoteClient.fetch(sku);
    }

    // Same business arguments followed by Throwable; compatible return type.
    private Stock fallback(String sku, Throwable cause) {
        return Stock.unknown(sku); // only if this is a truthful business response
    }
}
```

A circuit breaker alone does not stop a hung socket promptly; combine it with strict connection/read deadlines or a `TimeLimiter`. A bulkhead limits concurrent pressure, a rate limiter controls admission, and a retry handles carefully selected transient failures. Order matters: retries can multiply downstream load and distort breaker metrics. Retry only idempotent operations unless an idempotency key makes repetition safe; add jitter and a small maximum attempt count.

Tune from SLOs and measured baseline. A small window opens on noise; a very large window reacts too slowly. Exclude expected 4xx/domain misses from failure metrics. A fallback should not turn missing money/stock into fabricated success; sometimes the correct fallback is a clear `503` with `Retry-After`. Export state, buffered calls, failure/slow rates, rejected calls, fallback counts, and downstream latency through Micrometer/Actuator, and alert on sustained open state rather than every transition.

Resilience4j also has `METRICS_ONLY`, `DISABLED`, and `FORCED_OPEN` special states. These are useful operationally but are not part of the normal closed/open/half-open loop.

**References:** [Resilience4j circuit-breaker state machine and configuration](https://resilience4j.readme.io/docs/circuitbreaker), [Spring Cloud CircuitBreaker Resilience4j configuration](https://docs.spring.io/spring-cloud-circuitbreaker/reference/spring-cloud-circuitbreaker-resilience4j.html), [Spring Cloud CircuitBreaker metrics](https://docs.spring.io/spring-cloud-circuitbreaker/reference/spring-cloud-circuitbreaker-resilience4j/collecting-metrics.html)

### 17. How does Spring Boot Auto-configuration work under the hood? Explain `@ConditionalOnClass`, `@ConditionalOnMissingBean`, and `spring.factories`.

**Interview-ready summary.** `@SpringBootApplication` includes `@EnableAutoConfiguration`. Boot imports a catalog of `@AutoConfiguration` classes and evaluates conditions against the classpath, environment, application type, and bean definitions. `@ConditionalOnClass` activates configuration only when a library is present; `@ConditionalOnMissingBean` supplies a default only if the user has not supplied the relevant bean. In Boot 3+, auto-configuration candidates live in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`; describing `spring.factories` as the current registration mechanism is legacy.

Auto-configuration is ordinary configuration plus carefully ordered conditions. Boot first processes user definitions, then backs off where the application has made an explicit choice. A “starter” normally contributes dependencies; the corresponding auto-configure jar contributes conditional bean definitions. Conditions do not repeatedly decide on every request—they control definition registration during context construction.

```java
@AutoConfiguration
@ConditionalOnClass(HttpClient.class)
@ConditionalOnProperty(prefix = "acme.weather", name = "enabled",
                       havingValue = "true", matchIfMissing = true)
public class WeatherAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    WeatherClient weatherClient(WeatherProperties properties) {
        return new DefaultWeatherClient(properties.baseUrl());
    }
}
```

```text
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.acme.weather.WeatherAutoConfiguration
```

`@ConditionalOnClass` uses annotation metadata/ASM so a class-level condition can be checked without eagerly loading the optional type. Be careful placing it directly on a `@Bean` method whose return type is absent—the JVM may resolve that signature before the condition; isolate optional types in a nested configuration. `@ConditionalOnMissingBean` searches bean definitions by type/name/annotation according to its attributes. It is sensitive to processing order, which is why it belongs in auto-configuration that runs after user configuration.

Historically, Boot discovered auto-configurations from the `EnableAutoConfiguration` key in `META-INF/spring.factories`. Boot 2.7 introduced the imports file; Boot 3 removed `spring.factories` support for registering auto-configurations. `spring.factories` still has other framework/bootstrap uses, so it did not disappear entirely.

Debug unwanted behavior with `--debug`, the condition evaluation report, Actuator's `conditions` endpoint, and `ApplicationContextRunner` tests. Do not apply component scanning to an auto-configuration package; list candidates explicitly, use `@AutoConfiguration(before/after=...)` only for real ordering dependencies, and make every default overridable.

**References:** [Spring Boot auto-configuration behavior](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html), [Spring Boot: creating and registering auto-configuration](https://docs.spring.io/spring-boot/reference/features/developing-auto-configuration.html)

### 18. How do you secure microservices using OAuth2 and JWT with Spring Security? How do you manage stateless session validation at the gateway level?

**Interview-ready summary.** Let an OAuth 2.0/OIDC authorization server authenticate users/clients and issue short-lived, audience-bound access tokens. Configure the gateway as a Spring Security resource server: extract the Bearer token, verify its signature against trusted JWKs, validate `iss`, `aud`, `exp`, and `nbf`, map scopes/claims to authorities, authorize the route, and avoid an HTTP session. Downstream services should usually validate the token too, because gateway-only validation fails if a service is reachable by another path.

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://identity.example.com/realms/company
          audiences: https://api.example.com
```

```java
@Bean
SecurityWebFilterChain gatewaySecurity(ServerHttpSecurity http) {
    return http
        .csrf(ServerHttpSecurity.CsrfSpec::disable) // appropriate for header-only bearer API
        .securityContextRepository(NoOpServerSecurityContextRepository.getInstance())
        .authorizeExchange(auth -> auth
            .pathMatchers("/actuator/health", "/public/**").permitAll()
            .pathMatchers(HttpMethod.POST, "/payments/**").hasAuthority("SCOPE_payments.write")
            .anyExchange().authenticated())
        .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
        .build();
}
```

The JWT is self-contained, so “stateless” means the server does not persist a login session between calls; it does **not** mean “decode without state or checks.” `JwtDecoder`/`ReactiveJwtDecoder` verifies the JWS and validators. Cache the authorization server's JWK set with bounded refresh behavior and support key rotation via `kid`; never copy a shared signing secret among many services when asymmetric keys can separate signing from verification.

The gateway can preserve/relay the bearer token to a service. Spring Cloud Gateway's `TokenRelay` is useful when the gateway is an OAuth2 client and holds an authorized client token. On the service network, enforce that only the gateway/service identities can connect using network policy and preferably mTLS, but still authorize resource ownership in the service. Do not trust `X-User` or `X-Tenant` supplied by the public client; strip such headers and derive trusted context from validated claims.

JWT tradeoff: revocation is not immediate. Use short access-token lifetimes, refresh-token rotation at the authorization server, signing-key rotation for emergencies, and possibly a denylist or opaque-token introspection for high-risk immediate revocation. Avoid putting secrets/PII in a JWT—signing does not encrypt its payload. If authentication is cookie-based, CSRF protection is still needed; disabling it is justified only when credentials are not automatically attached by the browser.

**References:** [Spring Security reactive JWT resource server](https://docs.spring.io/spring-security/reference/reactive/oauth2/resource-server/jwt.html), [Spring Cloud Gateway Spring Security integration](https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/spring-security.html), [Spring Security servlet JWT validation](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html)

### 19. What is the difference between `@ControllerAdvice` and local `@ExceptionHandler`? How do you structure a global error response layout?

**Interview-ready summary.** A local `@ExceptionHandler` handles exceptions for its controller/class hierarchy and is best when the response is controller-specific. Methods in `@ControllerAdvice` apply across controllers; `@RestControllerAdvice` adds response-body semantics. Spring resolves applicable local handlers before global advice. For modern APIs, standardize on RFC 9457 `ProblemDetail` plus stable domain extensions such as an error code, trace ID, and validation violations.

```java
@RestControllerAdvice
final class ApiErrorAdvice {

    @ExceptionHandler(OrderNotFoundException.class)
    ProblemDetail notFound(OrderNotFoundException ex, HttpServletRequest request) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, "The requested order does not exist");
        problem.setTitle("Order not found");
        problem.setType(URI.create("https://api.example.com/problems/order-not-found"));
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("code", "ORDER_NOT_FOUND");
        problem.setProperty("traceId", MDC.get("traceId"));
        return problem;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    ResponseEntity<ProblemDetail> invalid(MethodArgumentNotValidException ex) {
        ProblemDetail p = ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_REQUEST, "One or more fields are invalid");
        p.setProperty("code", "VALIDATION_FAILED");
        p.setProperty("violations", ex.getBindingResult().getFieldErrors().stream()
            .map(e -> Map.of("field", e.getField(), "message", e.getDefaultMessage()))
            .toList());
        return ResponseEntity.badRequest().body(p);
    }
}
```

The core layout should be predictable: `type` (documented problem category URI), `title`, HTTP `status`, safe `detail`, request `instance`, stable machine-readable `code`, `traceId`/correlation ID, and—for 400 responses—a list of safe violations. Timestamp is optional; the trace system already has one. Keep status/code semantics in a versioned API contract.

Use multiple advice classes when domains need separation, with explicit `@Order` only where necessary. An advice can extend `ResponseEntityExceptionHandler` to customize Spring MVC's built-in exceptions. Log unexpected errors once at the boundary with the exception and trace ID, return a generic 500, and never leak stack traces, SQL, class names, tokens, or internal hostnames. Map domain conflicts deliberately (`409`, `422`, etc.) rather than converting every exception to `500` or `200`.

`@ControllerAdvice` is also capable of global `@InitBinder` and `@ModelAttribute`; it is not exception-only. In WebFlux the analogous model and `ProblemDetail` support exist, but use reactive request types.

**References:** [Spring MVC controller advice scope and precedence](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-advice.html), [Spring MVC `@ExceptionHandler`](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-exceptionhandler.html), [Spring RFC 9457 error responses and `ProblemDetail`](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-rest-exceptions.html)

### 20. Explain the architectural setup and benefits of Spring WebFlux over standard Spring MVC for high-throughput, non-blocking I/O.

**Interview-ready summary.** Spring MVC is built around the Servlet model and commonly assigns a request to a container thread while application code blocks on I/O. WebFlux is a reactive stack using `Mono`/`Flux`, non-blocking server adapters such as Reactor Netty, and Reactive Streams backpressure. It can sustain many concurrently waiting requests with a small event-loop population—but only when the entire hot path avoids blocking.

In WebFlux, a request enters an `HttpHandler`/`WebHandler` chain and `DispatcherHandler`, then an annotated controller or functional route returns a publisher. No thread waits for the value; readiness events resume small callbacks, and demand signals control how much a publisher emits. This reduces thread stacks and context switching for high fan-out APIs, streaming, WebSockets/SSE, and slow remote calls.

```java
@RestController
class ProductController {
    private final WebClient catalog;
    private final ReactiveProductRepository products; // e.g. R2DBC

    @GetMapping("/products/{id}")
    Mono<ProductView> get(@PathVariable UUID id) {
        return Mono.zip(
                products.findById(id),
                catalog.get().uri("/stock/{id}", id)
                       .retrieve().bodyToMono(Stock.class))
            .map(tuple -> ProductView.of(tuple.getT1(), tuple.getT2()))
            .timeout(Duration.ofSeconds(1));
    }
}
```

The benefits appear when concurrency × wait time is large. WebFlux does not make CPU-heavy mapping faster, and it can be worse for ordinary CRUD if the team and dependencies are imperative. Calling blocking JDBC/JPA, `RestTemplate`, filesystem code, or `Thread.sleep` on a Netty event loop stalls every request assigned to that loop. During migration, isolate unavoidable blocking calls on a bounded scheduler (`boundedElastic`) and measure it, but an end-to-end reactive driver such as R2DBC is the scalable architecture.

Reactive tradeoffs include a steeper debugging/stack-trace model, operator misuse, explicit context propagation (request state is not safely modeled by a simple `ThreadLocal` as execution moves), cancellation semantics, and the need to reason about backpressure. Spring MVC can itself return reactive types and use `WebClient`, and Java virtual threads make imperative high-concurrency designs another strong option. Choose with workload tests: p99 latency, event-loop utilization, allocation, connection-pool saturation, and developer operability matter more than a “reactive” label.

**References:** [Spring WebFlux overview](https://docs.spring.io/spring-framework/reference/web/webflux.html), [Spring WebFlux architecture and applicability guidance](https://docs.spring.io/spring-framework/reference/web/webflux/new-framework.html), [Spring reactive core and backpressure adapters](https://docs.spring.io/spring-framework/reference/web/webflux/reactive-spring.html)

## Part 1: Node.js Event Loop & Runtime Diagnostics

### 21. Walk through the 6 phases of the Node.js Event Loop in exact sequential execution order. Where do `process.nextTick()` and microtasks run?

**Interview-ready summary.** The traditional six named libuv phases are **timers → pending callbacks → idle/prepare → poll → check → close callbacks**. However, “exact sequential order” needs a version caveat: from libuv 1.45/Node 20, timers are processed after `poll` in each regular iteration (with an initial pre-loop timers run retained for compatibility). Current Node documentation therefore depicts an initial timers step, then pending → idle/prepare → poll → check → close → timers. `process.nextTick` and V8 microtasks are not libuv phases; Node drains them at JavaScript callback/operation boundaries before continuing the loop.

What each phase does:

1. **Timers:** runs eligible `setTimeout`/`setInterval` callbacks. A delay is a minimum threshold, not an appointment.
2. **Pending callbacks:** runs selected I/O callbacks deferred from a previous iteration, such as certain TCP errors.
3. **Idle, prepare:** internal libuv/Node work; application code does not schedule callbacks here.
4. **Poll:** obtains new I/O readiness and executes most I/O callbacks. It may wait when there is no immediate work.
5. **Check:** runs `setImmediate` callbacks. An immediate scheduled inside an I/O callback normally beats a zero-delay timer.
6. **Close callbacks:** runs callbacks such as `socket.on('close', ...)` for abruptly closed handles.

After the current JavaScript operation/callback unwinds, Node drains the **next-tick queue**, then the V8 microtask queue containing Promise reactions and `queueMicrotask` callbacks, before progressing. A callback can enqueue more work, so phases are not one callback each and a recursive `process.nextTick` chain can starve I/O. In CommonJS top-level code, `nextTick` normally precedes Promise microtasks; ES module evaluation already runs in a microtask, so top-level ordering can differ. Current Node marks `process.nextTick()` as legacy and recommends `queueMicrotask()` for most portable deferral.

```js
import fs from 'node:fs';

fs.readFile(new URL(import.meta.url), () => {
  process.nextTick(() => console.log('nextTick'));
  Promise.resolve().then(() => console.log('promise microtask'));
  setImmediate(() => console.log('check'));
  setTimeout(() => console.log('timer'), 0);
});

// In this I/O callback: nextTick, promise, check, then timer.
```

Do not memorize the diagram as a guarantee that every timeout precedes every immediate; context, readiness, OS differences, and the Node 20 timer change matter. The reliable design rule is to keep every callback small and never depend on incidental inter-phase timing for correctness.

**References:** [Node.js event-loop phases and Node 20 timer change](https://nodejs.org/learn/asynchronous-work/event-loop-timers-and-nexttick), [Node.js `process.nextTick` and microtask ordering](https://nodejs.org/api/process.html#when-to-use-queuemicrotask-vs-processnexttick)

### 22. How does Node.js handle heavy CPU-bound computational tasks without blocking the main event loop? Compare Worker Threads vs. Clustering vs. Child Processes.

**Interview-ready summary.** Node cannot make a long synchronous JavaScript calculation non-blocking merely by wrapping it in a Promise. Partition tiny work across turns or, for real CPU work, send it to a persistent Worker Thread pool or an external service. Cluster and child processes also provide parallelism, but their isolation, memory sharing, and operational purposes differ.

- **Worker Threads:** separate V8 isolates and JavaScript threads inside one process. They can transfer `ArrayBuffer`s without copying and intentionally share `SharedArrayBuffer` memory. Startup/message overhead is lower than a process but each worker still has an isolate/heap. Best for image transforms, parsing, compression implemented in JS, simulation, and cryptography not already asynchronous. Reuse a bounded pool; never spawn one worker per request.
- **Cluster:** forks multiple Node processes that can share a listening port. Each process has its own heap/event loop and can serve HTTP independently. It scales many ordinary requests across cores and gives process isolation, but it does not by itself split one expensive calculation; that job must still be sent over IPC. In Kubernetes, multiple one-process pods often replace in-process cluster management.
- **Child Processes:** `spawn`, `exec`, or `fork` starts another process/program. This gives the strongest memory/failure isolation and supports non-Node executables. IPC/serialization and startup cost are higher; `exec` buffers output and is dangerous for large/untrusted output, while `spawn` streams. It is useful for sandboxed tools, legacy binaries, or independently scalable compute services.

```js
// cpu-worker.mjs
import { parentPort } from 'node:worker_threads';
parentPort.on('message', ({ id, input }) => {
  try {
    parentPort.postMessage({ id, value: expensivePureFunction(input) });
  } catch (error) {
    parentPort.postMessage({ id, error: { message: error.message } });
  }
});

// In production, create N long-lived workers (about available CPU cores),
// queue bounded jobs, transfer large ArrayBuffers, and reject on saturation.
```

The main event loop should validate/admit work, enqueue it, and await the result; apply a queue limit, deadline, cancellation strategy, per-tenant quota, and metrics for queue age/utilization. Large structured-clone payloads can erase parallelism gains, so transfer buffers or keep data close to the worker. Shared memory demands atomic synchronization and can introduce races.

Also distinguish this from libuv's built-in worker pool: asynchronous `fs`, selected `crypto`, `zlib`, and DNS APIs already use native/libuv work. Worker Threads are for application JavaScript CPU work. Benchmark pool size—more workers than effective CPU quota can worsen throughput and tail latency.

**References:** [Node.js Worker Threads guidance](https://nodejs.org/api/worker_threads.html), [Node.js Cluster API](https://nodejs.org/api/cluster.html), [Node.js Child Process API](https://nodejs.org/api/child_process.html), [Node.js: Do not block the event loop or worker pool](https://nodejs.org/learn/asynchronous-work/dont-block-the-event-loop)

### 23. Explain how the `cluster` module utilizes round-robin load balancing across multiple CPU cores.

**Interview-ready summary.** The cluster primary creates workers using `child_process.fork()`. Under `cluster.SCHED_RR`, the primary owns/accepts on the listening socket and distributes accepted connections among worker processes in round-robin order, with some overload avoidance. Each worker has an independent V8 heap and event loop, so separate processes can execute JavaScript on separate CPU cores.

```js
import cluster from 'node:cluster';
import http from 'node:http';
import os from 'node:os';

if (cluster.isPrimary) {
  cluster.schedulingPolicy = cluster.SCHED_RR; // set before forking/setup
  for (let i = 0; i < os.availableParallelism(); i++) cluster.fork();

  cluster.on('exit', (worker, code, signal) => {
    console.error(`worker ${worker.process.pid} exited`, { code, signal });
    // Add rate-limited restart/backoff in production to avoid a crash loop.
    cluster.fork();
  });
} else {
  http.createServer((req, res) => {
    res.end(`served by ${process.pid}\n`);
  }).listen(8080); // workers logically share this port
}
```

There are two scheduling modes. `SCHED_RR` is the default on Unix-like platforms; Node's documentation still notes a Windows exception. With `SCHED_NONE`, the primary shares the listening handle and the operating system decides which worker accepts a connection. OS scheduling can be substantially uneven, which motivated Node's primary-distributed round robin.

The unit is usually a **connection**, not an HTTP request. HTTP keep-alive and WebSocket connections remain attached to one worker, so equal connection counts can still produce unequal work. Cluster does not provide application session sharing: store sessions, rate-limit counters, caches that require coherence, and jobs in external systems, or use load-balancer affinity only when unavoidable. Workers communicate through IPC; ordinary globals are not shared.

Production concerns include graceful worker replacement (stop accepting, drain existing connections, then exit), crash-loop backoff, readiness per worker, signal forwarding, and aggregate observability. Be aware of container CPU quotas: host CPU count can exceed the cores granted to the container. In an orchestrated platform, a simpler pattern is often one Node process per container and round-robin load balancing at the Service/ingress layer, gaining independent rollout and failure isolation.

**References:** [Node.js Cluster “How it works” and scheduling policy](https://nodejs.org/api/cluster.html#how-it-works), [Node.js `os.availableParallelism()`](https://nodejs.org/api/os.html#osavailableparallelism)

### 24. Explain V8 Engine Memory Management. What are the limits, how does the Scavenge/Mark-Sweep GC operate, and how do you profile leaks using heap snapshots?

**Interview-ready summary.** V8 uses a generational heap because most objects die young. New objects enter young-generation semi-spaces; a minor Scavenge/evacuation copies survivors and eventually promotes them. Major collection traces the whole heap and combines concurrent/incremental marking, sweeping, and selective compaction. The heap limit is platform/version/container dependent—query it or set it—while process RSS also includes Buffers, native allocations, code, stacks, and libraries.

```js
import v8 from 'node:v8';

console.table({
  ...process.memoryUsage(),          // rss, heapTotal, heapUsed, external, arrayBuffers
  heapLimit: v8.getHeapStatistics().heap_size_limit,
});
```

The young generation is divided into from/to semi-spaces. Allocation is cheap pointer bumping. During Scavenge, roots and remembered old-to-young references are traced; live objects are evacuated to the other space and pointers updated. Objects surviving enough minor collections are promoted to old space. This reclaims dead young objects without scanning the entire old heap.

For old space, modern “Mark-Sweep” is more accurately V8's Major Mark-Compact pipeline: concurrent helpers trace and mark reachable objects while JavaScript runs, a short finalization pause closes the marking set, unreachable regions are swept into free lists, and selected fragmented pages are compacted. Details evolve with V8, so interview answers should explain the model rather than claim every phase is stop-the-world.

`--max-old-space-size=4096` sets old-space size in MiB, but it is not a total process-memory cap. `Buffer` backing stores commonly appear under `external`/`arrayBuffers`; RSS can grow while `heapUsed` remains stable. Raising the limit can delay symptoms and create longer failure recovery; respect the container limit and leave native headroom.

For a suspected leak, graph `heapUsed` after GC and allocation rate, then compare snapshots under equivalent load. Create them with the inspector/Chrome DevTools, `v8.writeHeapSnapshot()`, or operational snapshot options/signals. In DevTools inspect comparison deltas, dominators, retained size, and paths to GC roots—large shallow size is less informative than what an object retains.

```bash
node --max-old-space-size=2048 --heap-prof app.mjs
# For development/staging diagnostics:
node --inspect app.mjs
```

Heap snapshots stop the isolate and can require roughly another heap's worth of memory, potentially crashing a production instance. Capture on a drained replica or use sampling heap profiles/diagnostic reports first; protect artifacts as sensitive data. Remember that a main-thread snapshot does not include Worker isolates.

**References:** [V8 generational and major GC internals](https://v8.dev/blog/trash-talk), [Node.js V8 heap statistics and snapshot APIs](https://nodejs.org/api/v8.html), [Node.js heap-snapshot diagnostics guide](https://nodejs.org/en/learn/diagnostics/memory/using-heap-snapshot)

### 25. What is the difference between `Stream.pipe()` and manual event handling (`on('data')`)? How do you prevent backpressure issues in streams?

**Interview-ready summary.** Adding a `data` listener puts a readable into flowing mode and gives the application responsibility for pacing, errors, completion, and teardown. `readable.pipe(writable)` connects those mechanics and automatically pauses/resumes according to the destination's backpressure signal. For production multi-stage streams, prefer `stream.pipeline()` because it propagates errors and destroys/cleans up the chain.

```js
import { createReadStream, createWriteStream } from 'node:fs';
import { pipeline } from 'node:stream/promises';
import { createGzip } from 'node:zlib';

await pipeline(
  createReadStream('huge.csv', { highWaterMark: 64 * 1024 }),
  createGzip(),
  createWriteStream('huge.csv.gz'),
); // rejects on an error in any stage and cleans up streams
```

A writable's `write(chunk)` returns `false` when its internal buffer has reached its `highWaterMark`. That is a signal to stop producing until `drain`; it is not an error and `highWaterMark` is a threshold, not a hard global memory ceiling.

```js
import { once } from 'node:events';

async function writeAll(writable, chunks) {
  for await (const chunk of chunks) {
    if (!writable.write(chunk)) {
      // events.once also rejects if the emitter produces an 'error' event.
      await once(writable, 'drain');
    }
  }
  writable.end();
}
```

With manual `on('data')`, calling a slow async function does not make EventEmitter wait. Hundreds of handlers can be outstanding and retain chunks. Either `pause()` before async work and `resume()` afterward, correctly honor `write()`/`drain`, or—usually more clearly—consume with `for await...of`, which naturally awaits each iteration. Limit transform concurrency explicitly if ordering is not required.

Plain `pipe()` manages flow but historically does not make all error/cleanup behavior as robust as `pipeline`; every stream still needs correct lifecycle handling. Do not mix `data`, `readable`, `pipe`, and async-iterator consumption on the same stream without understanding flowing state. Bound object-mode buffers by object counts and the actual size of each object, set upload/decompression limits, abort the pipeline when the client disconnects, and monitor writable length plus queue age. Ignoring backpressure can produce unbounded RAM growth and, for sockets whose peer never reads, a remotely triggerable denial of service.

**References:** [Node.js Streams API, `write()`/`drain`, `pipe`, and `pipeline`](https://nodejs.org/api/stream.html), [Node.js backpressure guide](https://nodejs.org/en/learn/modules/backpressuring-in-streams)

### 26. How do you handle unhandled promise rejections and uncaught exceptions safely in a production Node.js environment without leaving the app corrupted?

**Interview-ready summary.** Handle expected failures at their ownership boundary. Treat an actually unhandled rejection or uncaught exception as fatal: record enough diagnostics, do only safe cleanup, terminate, and let an external supervisor start a clean process. Continuing after an uncaught exception is unsafe because partially completed mutations and violated invariants may have left the process in an undefined state.

Modern Node's default unhandled-rejection mode is `throw`: if a rejection remains unhandled, it is raised as an uncaught exception. Make every detached Promise deliberately observed and install framework boundary handling.

```js
import { writeSync } from 'node:fs';

// Express 5 automatically forwards rejected async handlers to next(error).
app.get('/orders/:id', async (req, res) => {
  const order = await orders.find(req.params.id); // rejection reaches error middleware
  res.json(order);
});

app.use((err, req, res, next) => {
  logger.error({ err, requestId: req.id }, 'request failed');
  if (res.headersSent) return next(err);
  res.status(500).json({ code: 'INTERNAL_ERROR', requestId: req.id });
});

// Observe fatal errors without suppressing Node's normal crash behavior.
process.on('uncaughtExceptionMonitor', (err, origin) => {
  // A synchronous write is suitable in this last-resort path.
  writeSync(process.stderr.fd, `${origin}: ${err.stack ?? err}\n`);
});
```

`uncaughtExceptionMonitor` observes without changing the exit behavior. By contrast, adding an `uncaughtException` listener prevents the default exit, so it must explicitly terminate and must not resume serving. Node's own guidance limits this handler to synchronous cleanup. For normal planned shutdown (`SIGTERM`), it is appropriate to mark readiness false, call `server.close()` to stop new connections, drain within a hard deadline, close pools, then exit. Do not assume that lengthy asynchronous graceful shutdown is safe after arbitrary memory corruption/invariant failure.

Run under systemd, Kubernetes, a process manager, or another supervisor with restart backoff and crash-loop alerting. Use multiple replicas so one fatal process does not remove capacity. Capture structured logs, a diagnostic report/core dump where policy permits, the release version, request correlation, and health metrics. Never put secrets into crash output.

Prevent the fatal path with `await`/`try-catch`, `.catch()` on fire-and-forget work, `Promise.allSettled` where partial failure is intentional, deadlines/abort signals, validation, and tests that force dependency failures. Do not swallow rejections globally or convert all errors to success; that hides defects and lets corrupted state persist.

**References:** [Node.js process events and correct `uncaughtException` handling](https://nodejs.org/api/process.html#event-uncaughtexception), [Node.js `unhandledRejection` event](https://nodejs.org/api/process.html#event-unhandledrejection), [Express error handling](https://expressjs.com/en/guide/error-handling.html)

### 27. Compare the performance, security, and scoping behaviors of CommonJS (`require`) versus ES Modules (`import/export`).

**Interview-ready summary.** CommonJS (CJS) is Node's original synchronous, function-wrapped module system with `require` and mutable `module.exports`. ES Modules (ESM) are the JavaScript standard: statically analyzable imports/exports, live bindings, strict mode, URL-based resolution, asynchronous graph support, and top-level `await`. Neither is a security sandbox, and neither is universally faster; choose for ecosystem/interoperability and measure startup-sensitive workloads.

**Loading/performance.** CJS resolves and executes a dependency synchronously on first `require`, then returns its cached `module.exports`. ESM constructs, links, and evaluates a module graph; static imports are known before execution and exports are live bindings. Static structure helps bundlers tree-shake browser/serverless bundles, but Node itself does not magically tree-shake an application at runtime. Dynamic `import()` is asynchronous and works from both systems. Current Node versions can `require()` synchronous ESM graphs in defined cases, but a graph containing top-level `await` must be loaded with `import()`; interoperability behavior has changed across Node releases.

**Scope and semantics.** Both have module-local scope. CJS achieves it with a wrapper function that supplies `exports`, `require`, `module`, `__filename`, and `__dirname`; exported objects can be mutated/reassigned. ESM is strict by default, has no CJS wrapper globals, uses `import.meta.url`/modern `import.meta` helpers, and imported bindings cannot be reassigned by the importer even though referenced objects may be mutable. Circular dependencies expose partially initialized CJS exports, while ESM cycles use linked live bindings and temporal-dead-zone rules; both require careful design.

```json
{ "type": "module" }
```

```js
// ESM
import legacy from './legacy.cjs';
export const answer = 42;

// CJS loading ESM portably, including async graphs:
async function loadModernModule() {
  return import('./modern.mjs');
}
```

`.mjs` and `.cjs` are explicit; the nearest `package.json` `type` governs `.js`. Use package `exports` to define supported entry points and avoid accidental deep imports.

**Security.** Both execute dependencies with the process's OS privileges. ESM's static grammar improves policy/tooling analysis and avoids some monkey-patching patterns, but a malicious imported package can still read files, environment variables, or network. Defend with dependency review/lockfiles, minimal OS/container permissions, secret isolation, and Node's policy/permission mechanisms where suitable—not with module syntax alone. Avoid constructing `require`/import specifiers directly from untrusted input.

**References:** [Node.js CommonJS modules](https://nodejs.org/api/modules.html), [Node.js ECMAScript modules](https://nodejs.org/api/esm.html), [Node.js package type and exports rules](https://nodejs.org/api/packages.html)

### 28. Explain the difference between `Buffer.alloc()` and `Buffer.allocUnsafe()`. What are the hidden security risks of using the unsafe method?

**Interview-ready summary.** `Buffer.alloc(size)` returns initialized memory, zero-filled unless another fill is supplied. `Buffer.allocUnsafe(size)` returns uninitialized memory and is faster because it skips zeroing; bytes may contain old process data. Reading, serializing, hashing, or sending any byte before fully overwriting it can leak tokens, keys, customer data, or other remnants.

```js
// Safe default for a buffer whose whole contents may be observed.
const frame = Buffer.alloc(1024);

// Acceptable only when every byte is guaranteed to be overwritten first.
const encoded = Buffer.allocUnsafe(payloadLength);
let offset = 0;
offset += encoded.writeUInt32BE(messageType, offset);
offset += payload.copy(encoded, offset);
if (offset !== encoded.length) throw new Error('partial buffer initialization');
socket.write(encoded); // no uninitialized suffix can escape
```

The hidden bug is often an exceptional or conditional path: code allocates 4 KiB, writes only the real 600-byte payload, then sends all 4 KiB; or validation throws halfway through filling and a logging path prints the buffer. Small unsafe buffers may be slices of Node's shared internal slab, making stale bytes particularly plausible. `allocUnsafeSlow` avoids the shared pool but is still uninitialized.

Only use unsafe allocation in a measured hot path with a simple, reviewable proof that all bytes are written before every possible read. `buf.fill(0)` makes it initialized, though `Buffer.alloc()` expresses the intent better. The `--zero-fill-buffers` process option forces unsafe allocations to be zeroed at a performance cost and can be defense in depth for a sensitive service.

Also validate attacker-controlled sizes before *either* allocation. Zero-filling prevents disclosure, not memory-exhaustion DoS. Buffers use backing memory outside the ordinary V8 object heap; watch `process.memoryUsage().external` and `.arrayBuffers`, not just `heapUsed`. Retaining a small slice can retain a larger underlying allocation, so copy long-lived tiny data when retention matters.

The deprecated `new Buffer(...)` API should not be used because its behavior historically depended on the argument type and made validation mistakes especially dangerous. Use `Buffer.from(data)` for existing content, `alloc` for initialized capacity, and `allocUnsafe` only under the strict full-overwrite contract.

**References:** [Node.js Buffer allocation and unsafe-memory warning](https://nodejs.org/api/buffer.html#static-method-bufferallocunsafesize), [Node.js `Buffer.from`, `alloc`, and `allocUnsafe` comparison](https://nodejs.org/api/buffer.html#bufferfrom-bufferalloc-and-bufferallocunsafe)

### 29. How do Libuv thread pools manage asynchronous file system operations when OS kernels don't provide native async I/O drivers?

**Interview-ready summary.** libuv implements asynchronous filesystem APIs by running blocking filesystem syscalls on worker-pool threads. The event-loop thread submits a work request and returns; a worker performs the blocking call, then posts completion back to the loop, which invokes the JavaScript callback/resolves the Promise. Network sockets normally use OS readiness/completion facilities such as epoll, kqueue, or IOCP instead of this pool.

Node's callback and Promise filesystem APIs (except watchers and documented exceptions) share libuv's global worker pool with `dns.lookup`, selected crypto operations, and zlib. The default pool size is 4 and `UV_THREADPOOL_SIZE` can be set at process startup (libuv currently allows up to 1024). It is global across event loops, including workers in the process, so four slow filesystem calls can delay an unrelated DNS/crypto operation.

```bash
# Set before Node initializes/uses the pool; benchmark this value.
UV_THREADPOOL_SIZE=16 node server.mjs
```

```js
import { readFile } from 'node:fs/promises';

// JavaScript yields; a libuv worker performs blocking filesystem work.
const config = JSON.parse(await readFile('/etc/my-app/config.json', 'utf8'));
```

Increasing the pool can improve throughput when independent operations wait on genuinely parallel storage, but it can also increase memory, context switching, and disk contention. Modern libuv worker threads reserve sizeable stacks, and a single disk/device may simply queue more work. Tune using filesystem latency, thread-pool-dependent operation latency, CPU/iowait, storage queue depth, and the process/container memory budget. Application-level concurrency should remain bounded; `Promise.all` over a million paths first allocates a million tasks even though only a few can execute.

File operations are asynchronous from JavaScript's perspective, not automatically transactional or synchronized. Concurrent modifications to the same file can race. Prefer streams for large data so completion of one giant `readFile` does not create a huge Buffer and downstream memory spike.

libuv briefly had broader Linux `io_uring` filesystem behavior in some releases, but current documentation notes the default reverted to the thread pool, with optional configuration for suitable cases. Therefore “filesystem always uses kernel native async I/O” is not a portable Node assumption.

**References:** [libuv thread-pool scheduling and size](https://docs.libuv.org/en/v1.x/threadpool.html), [libuv filesystem design](https://docs.libuv.org/en/v1.x/guide/filesystem.html), [Node.js filesystem thread-pool usage](https://nodejs.org/api/fs.html#threadpool-usage)

### 30. How would you optimize a Node.js API that processes massive multiline CSV file uploads to keep memory consumption under 50MB?

**Interview-ready summary.** Stream bytes from the socket through an incremental CSV parser, validate one record at a time, write small bounded batches, and await each downstream write so backpressure reaches the client. Never buffer the upload, call `readFile`, use a multipart memory-storage engine, collect all rows, or launch an unbounded Promise per row. Clarify the SLO: keeping *incremental upload memory* below 50 MB is controllable; total RSS below 50 MB may be unrealistic for a given Node/V8 build and dependency set.

```js
import express from 'express';
import { parse } from 'csv-parse';

const app = express();

app.post('/imports', async (req, res, next) => {
  if (!req.is('text/csv')) return res.status(415).end();

  const parser = req.pipe(parse({
    columns: true,
    bom: true,
    skip_empty_lines: true,
    max_record_size: 64 * 1024, // blocks a single malicious giant record
  }));

  try {
    let batch = [];
    let rows = 0;
    for await (const raw of parser) { // awaiting DB writes backpressures parser/socket
      batch.push(validateAndNormalize(raw));
      rows++;

      if (batch.length === 250) {
        await stagingRepository.insertBatch(batch); // bounded in-flight work
        batch = [];
      }
    }
    if (batch.length) await stagingRepository.insertBatch(batch);
    res.status(202).json({ rows, status: 'staged' });
  } catch (error) {
    next(error);
  }
});
```

For multipart uploads, use a streaming parser such as Busboy and pipe the emitted file stream directly into the CSV parser; configure file-size, file-count, part-count, header, and timeout limits. Do not place `express.raw()`/JSON parsing or Multer `memoryStorage` before this route. Bound `highWaterMark`, maximum record/field length, column count, batch rows, number of simultaneous imports, and queued jobs. Estimate worst-case retained bytes from these bounds and leave room for V8, native buffers, libraries, and database drivers.

Massive imports should usually return `202`, store the upload in object storage or a staging table, and process via a durable queue with per-tenant concurrency limits. Use a staging/import ID and idempotency key; validate/merge in chunks, then atomically mark the job complete. A single transaction spanning hours retains locks/WAL and makes retries painful. Reject CSV formula injection if records are later exported to spreadsheets, validate encoding, and never trust the filename.

Move genuinely CPU-heavy transforms to a bounded Worker pool, transferring chunks rather than cloning entire files. Track `rss`, `heapUsed`, `external`, `arrayBuffers`, parser/writer buffer lengths, batch latency, and event-loop delay under concurrent worst-case uploads. Add a load test that samples peak RSS; architecture alone does not prove the 50 MB target.

**References:** [Node.js stream backpressure and `pipeline`](https://nodejs.org/api/stream.html), [CSV Parse stream API](https://csv.js.org/parse/api/stream/), [Busboy streaming upload parser and limits](https://github.com/mscdex/busboy)

## Part 1: Express / NestJS Framework Architecture

### 31. How does middleware chaining work in Express under the hood? Explain the mechanical design of the `next()` function call.

**Interview-ready summary.** An Express application/router maintains an ordered stack of layers. Each layer contains a path/method matcher and one or more handlers. Dispatch creates a request-specific `next` closure holding the current stack index and routing state. Calling `next()` advances that index until it finds the next matching normal layer; ending the response without calling it terminates the chain.

Conceptually—not exact Express source—the dispatcher behaves like this:

```js
function dispatch(stack, req, res, done) {
  let index = 0;
  function next(error) {
    const layer = stack[index++];
    if (!layer) return done(error);
    if (!layer.matches(req)) return next(error);

    const isErrorHandler = layer.handler.length === 4;
    if (error && !isErrorHandler) return next(error);
    if (!error && isErrorHandler) return next();
    return error
      ? layer.handler(error, req, res, next)
      : layer.handler(req, res, next);
  }
  next();
}
```

`next()` continues normally. `next(error)` (anything except the special strings) skips normal handlers until a four-argument error handler. `next('route')` skips the rest of the current route's callbacks and searches the next route; `next('router')` exits the current router. Mounted routers form nested stacks and temporarily adjust path/base URL state.

```js
app.use((req, res, next) => {
  const started = performance.now();
  res.on('finish', () => metrics.observe(performance.now() - started));
  return next(); // return avoids accidental fall-through after delegation
});

app.get('/orders/:id', authenticate, authorize, async (req, res) => {
  res.json(await orders.find(req.params.id));
});

app.use((err, req, res, next) => { // must have four parameters
  if (res.headersSent) return next(err);
  res.status(500).json({ code: 'INTERNAL_ERROR' });
});
```

Calling `next()` does not mean “return from my function”; code after it can run, sometimes after a synchronous downstream handler, and may cause double sends. Use `return next()` unless deliberate post-processing is clearly managed. Do not call `next()` after sending a response. In Express 5, a middleware/handler that returns a rejected Promise or throws asynchronously through an `async` function automatically invokes `next(rejection)`; older Express versions need a wrapper or explicit catch.

Order is architecture: parsers before handlers that need bodies, authentication before authorization, routes before the final 404, and error middleware last. A middleware that neither responds nor calls `next` leaves the request hanging.

**References:** [Express middleware model](https://expressjs.com/en/guide/using-middleware.html), [Express writing middleware and Express 5 Promise behavior](https://expressjs.com/en/guide/writing-middleware/), [Express error handling](https://expressjs.com/en/guide/error-handling.html)

### 32. Explain the execution context sequence of Interceptors, Guards, Pipes, and Exception Filters in a NestJS request lifecycle.

**Interview-ready summary.** The inbound order is **middleware → guards → interceptor pre-handling → pipes → controller/service**. The normal response unwinds through interceptors in reverse order. An uncaught error jumps to exception filters, resolved from the most specific (route) to controller to global. In scope ordering, guards/interceptors/pipes generally enter global → controller → route.

The responsibilities explain the order:

- **Guards** run after middleware and before interceptors/pipes. They use `ExecutionContext` and route metadata to answer whether the handler may execute—ideal for authentication/authorization. Global guards run before controller and route guards, in binding order.
- **Interceptors** wrap `next.handle()` as an RxJS `Observable`. Inbound execution is global → controller → route; response transformation/finalization unwinds route → controller → global (first in, last out). An interceptor can time/cache/transform or catch an error before filters see it.
- **Pipes** validate/transform values just before handler invocation. Global → controller → route → route-parameter pipes apply; parameter-level ordering has a Nest-specific right-to-left nuance when multiple decorated parameters are present. If a pipe throws, the controller is never called.
- **Exception filters** run only for uncaught exceptions. Resolution begins route → controller → global, unlike the global-first inbound enhancers. A filter that handles an exception ends filter resolution; exceptions are not passed from one filter to the next like Express error middleware.

```ts
@UseGuards(RolesGuard)           // controller-level guard
@UseInterceptors(LoggingInterceptor)
@Controller('orders')
export class OrdersController {
  @Get(':id')
  @UseInterceptors(CacheInterceptor) // innermost on inbound, first on return
  @UseFilters(OrderExceptionFilter)
  find(@Param('id', ParseUUIDPipe) id: string) {
    return this.orders.find(id);
  }
}
```

Full normal sequence: global/module middleware; global/controller/route guards; global/controller/route interceptor “before” logic; global/controller/route/parameter pipes; controller and service; route/controller/global interceptor “after” logic; response. If a pipe/controller/service/interceptor throws and no interceptor converts the error, route/controller/global filters are considered, followed by Nest's built-in exception layer.

`ExecutionContext` extends `ArgumentsHost` and exposes `getClass()`/`getHandler()`, allowing guards and interceptors to read decorator metadata while still adapting to HTTP, RPC, GraphQL, or WebSockets. Integration-test order when multiple global providers are involved; registration method and Nest major-version middleware changes can affect ordering details.

**References:** [NestJS request lifecycle and complete ordering](https://docs.nestjs.com/faq/request-lifecycle), [NestJS execution context](https://docs.nestjs.com/fundamentals/execution-context), [NestJS exception filters](https://docs.nestjs.com/exception-filters)

### 33. How do you build a robust, scalable multi-tenant architectural filter in Express/NestJS to isolate database connections dynamically based on request headers?

**Interview-ready summary.** Treat the header only as a *tenant selector*, never as proof of access. Authenticate first, verify that the selected tenant is in the caller's trusted claims/membership, create immutable request context, and route all repositories through a controlled tenant-aware data-access layer. Reuse bounded pools rather than opening a connection per request, and add database-enforced isolation such as PostgreSQL row-level security (RLS) so one missed application predicate cannot leak another tenant's rows.

Choose an isolation model intentionally:

- **Database per tenant:** strongest operational isolation and per-tenant restore, but migrations and one pool per tenant become expensive. A registry must resolve an allow-listed tenant to secrets from a control plane—not turn a raw header into a database name/URL. Cache a bounded number of pools with idle eviction and connection budgets.
- **Schema per tenant:** intermediate isolation but dynamic identifiers are difficult to parameterize and schema/migration counts grow.
- **Shared schema with `tenant_id`:** most pool-efficient for many tenants; every key/index/query includes the tenant and RLS provides defense in depth. Noisy-neighbor quotas and tenant-aware partitioning remain necessary.

```js
import { AsyncLocalStorage } from 'node:async_hooks';
const tenantContext = new AsyncLocalStorage();

// Runs after JWT authentication has populated req.user.
app.use((req, res, next) => {
  const tenantId = req.get('X-Tenant-Id');
  if (!isUuid(tenantId) || !req.user.tenantIds.includes(tenantId)) {
    return res.status(403).json({ code: 'TENANT_FORBIDDEN' });
  }
  tenantContext.run(Object.freeze({ tenantId, subject: req.user.sub }), next);
});

async function inTenantTransaction(work) {
  const context = tenantContext.getStore();
  if (!context) throw new Error('missing tenant context');

  const client = await sharedPool.connect();
  try {
    await client.query('BEGIN');
    // true makes the setting transaction-local, so pooled connections do not leak it.
    await client.query("SELECT set_config('app.tenant_id', $1, true)",
                       [context.tenantId]);
    const result = await work(client);
    await client.query('COMMIT');
    return result;
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

```sql
ALTER TABLE invoice ENABLE ROW LEVEL SECURITY;
ALTER TABLE invoice FORCE ROW LEVEL SECURITY;
CREATE POLICY invoice_tenant ON invoice
  USING (tenant_id = current_setting('app.tenant_id')::uuid)
  WITH CHECK (tenant_id = current_setting('app.tenant_id')::uuid);
```

Run the application as a non-owner role without `BYPASSRLS`; owners/superusers can bypass ordinary RLS. Include `tenant_id` in unique constraints, cache keys, object-storage paths, message payloads, search filters, metrics, rate limits, and audit events. Strip public `X-Tenant-Id` at a trusted gateway if an internal signed header/claim is used.

In NestJS, a guard can authenticate/authorize the tenant and middleware/`AsyncLocalStorage` can propagate it. Request-scoped data-source providers work, and Nest documents durable per-tenant DI subtrees, but request scope adds object-creation overhead and durable subtrees are unsuitable for unbounded tenant cardinality. Prefer a singleton pool registry plus request context. Test with many concurrent interleaved tenant A/B requests, background jobs, errors, and pool reuse; isolation bugs often appear only after asynchronous reuse. Apply maximum tenants/pools, per-tenant quotas, secret rotation, migration orchestration, and audit logs before calling the design scalable.

**References:** [NestJS injection scopes and multi-tenant durable providers](https://docs.nestjs.com/fundamentals/injection-scopes), [Node.js `AsyncLocalStorage`](https://nodejs.org/api/async_context.html#class-asynclocalstorage), [PostgreSQL row-level security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html), [NestJS AsyncLocalStorage recipe](https://docs.nestjs.com/recipes/async-local-storage)

---

## Part 1: Frontend Architecture (React)

### 34. Explain the React Fiber architecture. How does it break down the reconciliation phase into incremental, interruptible chunks?

**Interview-ready answer.** Fiber is React's reimplementation of reconciliation as an explicit, persistent work graph. A *fiber* is a JavaScript object representing one component/host element and one unit of work. Instead of relying on an ordinary recursive call stack that must finish in one pass, React links fibers through `child`, `sibling`, and `return` pointers and advances an explicit work loop one fiber at a time. On a concurrent root, the scheduler can yield between units, resume later, abandon obsolete work, and prioritize urgent updates over transitions. The eventual DOM commit is still synchronous and non-interruptible.

Important fields conceptually include:

- `type` and `key`: identity used during reconciliation.
- `pendingProps`, `memoizedProps`, `memoizedState`: incoming and last-completed inputs/state.
- `child`, `sibling`, `return`: a traversal-friendly linked tree.
- `alternate`: links the currently committed fiber to its reusable work-in-progress counterpart (double buffering).
- lanes/flags: update priorities and effects that the commit phase must perform.

The render/reconciliation loop resembles this simplified pseudocode:

```js
function workLoop(deadline) {
  while (nextUnitOfWork && !deadline.shouldYield()) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
  }
  if (nextUnitOfWork) scheduleContinuation(workLoop);
  else commitRoot(finishedWork); // atomic DOM/effect commit
}

function performUnitOfWork(fiber) {
  const firstChild = beginWork(fiber); // call component, reconcile children
  if (firstChild) return firstChild;

  let node = fiber;
  while (node) {
    completeWork(node);               // bubble flags/build host result
    if (node.sibling) return node.sibling;
    node = node.return;
  }
  return null;
}
```

React actually schedules by *lanes* and host scheduling primitives rather than literally using this snippet. A click can be assigned urgent work while an update wrapped in `startTransition` is lower priority. If the transition render is interrupted, React keeps the current tree visible and may resume or restart the work-in-progress tree. This is why render logic must be pure: React may invoke it without committing the result.

**Trade-offs and pitfalls.** Fiber makes concurrency possible; it does not mean every update is time-sliced. Legacy roots, `flushSync`, and some urgent work run synchronously. React can yield only between React work units, not in the middle of an expensive user function; a component doing 100 ms of computation still blocks the main thread. Fiber also does not make the commit interruptible, because users must not see a half-mutated DOM. Use transitions, component boundaries, memoization proven by profiling, list virtualization, or a Web Worker for genuinely heavy CPU work.

**References:** [React Fiber architecture notes by React core contributor Andrew Clark](https://github.com/acdlite/react-fiber-architecture), [React 18 concurrent rendering overview](https://react.dev/blog/2022/03/29/react-v18), [React source: reconciler package](https://github.com/facebook/react/tree/main/packages/react-reconciler).

### 35. What is the exact difference between the Render Phase and the Commit Phase in React? Which lifecycle methods or hooks execute in each?

**Interview-ready answer.** The **render phase** computes what the UI should become; the **commit phase** applies the finished result to the host environment. Render may be paused, restarted, or discarded in concurrent rendering and must be pure. Commit operates on an already completed fiber tree, performs mutations/effect lifecycles, and is synchronous for that root.

| Phase | Representative work and APIs |
|---|---|
| Render/reconciliation | Function component bodies; hook state/reducer processing; `useMemo` calculations; class `constructor` on mount, `static getDerivedStateFromProps`, `shouldComponentUpdate`, and `render`; child diffing and effect-flag construction |
| Commit: before/mutation/layout | `getSnapshotBeforeUpdate`; DOM insertion/update/removal; ref detach/attach; class `componentDidMount`, `componentDidUpdate`, `componentWillUnmount`; `useLayoutEffect` cleanup/setup; insertion-effect work; error-boundary `componentDidCatch` |
| Passive effects after the commit | `useEffect` cleanup for the previous dependencies, then its new setup; React normally schedules this without blocking paint |

The public mental model is:

```text
trigger update
  -> render: call components and build work-in-progress tree (no visible DOM change)
  -> commit: mutate DOM, update refs, run layout lifecycles/effects
  -> browser paint (usual case)
  -> flush passive useEffect cleanups/setups
```

Some exact timing depends on why an effect was scheduled and React/browser scheduling, so do not build correctness around “`useEffect` always runs after paint.” `useLayoutEffect` is the API for DOM measurement or a visual correction that must occur before paint. `useInsertionEffect` exists mainly for CSS-in-JS libraries and runs early enough for style insertion; application code should rarely need it.

Example of the appropriate separation:

```tsx
function Tooltip({ anchor }: { anchor: HTMLElement }) {
  const ref = React.useRef<HTMLDivElement>(null);

  // Render: calculate JSX only—no DOM writes, subscriptions, or requests.
  React.useLayoutEffect(() => {
    const box = anchor.getBoundingClientRect();
    ref.current!.style.transform = `translate(${box.left}px, ${box.bottom}px)`;
  }, [anchor]);

  React.useEffect(() => {
    const onResize = () => console.log('reposition on next state update');
    window.addEventListener('resize', onResize);
    return () => window.removeEventListener('resize', onResize);
  }, []);

  return <div ref={ref}>Help</div>;
}
```

**Trade-offs and pitfalls.** Side effects in a component body or `useMemo` may execute multiple times, execute for an abandoned render, or be exposed by development Strict Mode. Never fetch imperatively, mutate shared objects, start timers, or manipulate DOM during render. A state update in a layout effect forces more work before paint and can cause jank; prefer passive effects unless visual correctness requires layout timing. Legacy `UNSAFE_componentWill*` methods are render-phase lifecycles and inherit these risks.

**References:** [React: Render and Commit](https://react.dev/learn/render-and-commit), [React `useEffect` reference](https://react.dev/reference/react/useEffect), [React `useLayoutEffect` reference](https://react.dev/reference/react/useLayoutEffect).

### 36. Deep dive into React's Diffing Algorithm. How do component keys affect state preservation, unmounting, and element DOM updates?

**Interview-ready answer.** General tree edit distance is expensive, so React uses an O(n)-style heuristic based mainly on two assumptions: different element types represent different subtrees, and developers identify stable siblings with keys. Reconciliation compares each new element with the old fiber at the corresponding parent position using `(type, key)` identity.

- **Same type and same key:** React reuses the fiber and component state. For a host node such as `div`, it reuses the DOM node and patches changed properties/styles/listeners; then it reconciles children.
- **Different type:** React unmounts the old subtree (cleanup lifecycles/effects and refs), discards its state, creates a new subtree, and replaces the relevant DOM.
- **Same type but different key:** React deliberately treats it as a different instance. This is a useful way to reset a form or restart a subtree.
- **Arrays:** React first follows position/key matches, then uses a key map to locate moved/reused children. Keys need be unique only among siblings, but must be stable across renders.

```tsx
// Stable identity: state follows each todo even after sort/filter.
{todos.map(todo => <TodoRow key={todo.id} todo={todo} />)}

// Deliberate reset when selecting a different user.
<ProfileEditor key={selectedUserId} userId={selectedUserId} />
```

With index keys, inserting at the start turns old position 0 into the identity of the new first item. React may retain an input's local state, focus, or uncontrolled DOM value in the wrong logical row:

```tsx
// Unsafe if items can be inserted, deleted, sorted, or filtered.
{todos.map((todo, index) => <TodoRow key={index} todo={todo} />)}
```

Keys are not passed as a normal prop; pass `id={todo.id}` separately if the child needs it. A key belongs where the array is created, not hidden inside `TodoRow`. Random keys (`key={Math.random()}`) force a complete remount every render, losing state and DOM focus and repeating effect cleanup/setup.

**Trade-offs and pitfalls.** React does not search globally for a component that moved to a different parent; identity is local to its position in a parent's child set. Duplicate keys make matching ambiguous. An index is acceptable for a truly static, never-reordered presentation list, but real database IDs are preferable. Also distinguish “component rendered again” from “DOM updated”: React may call the component, diff equal host output, and commit no DOM mutation.

**References:** [React: Preserving and Resetting State](https://react.dev/learn/preserving-and-resetting-state), [React: Rendering Lists and choosing keys](https://react.dev/learn/rendering-lists), [React legacy reconciliation algorithm explanation](https://legacy.reactjs.org/docs/reconciliation.html).

### 37. How does React handle Synthetic Event pooling and event delegation at the root container level in React 17+?

**Interview-ready answer.** A React event handler receives a `SyntheticEvent`: a cross-browser wrapper exposing a consistent interface such as `target`, `currentTarget`, `preventDefault()`, and `stopPropagation()`, while retaining the native event under `nativeEvent`.

Two React 17 changes are the key interview points:

1. **Pooling was removed on the web.** React 16 and earlier reused event objects and nulled their fields after dispatch, so asynchronous code needed `event.persist()` or had to copy values. In React 17+, the event object is not pooled; `persist()` is effectively a no-op and event properties can be retained. It is still good practice to copy the actual value required, because DOM nodes and application state can change asynchronously.
2. **Delegation moved from `document` to each React root container.** For most supported event types, React installs shared native capture/bubble listeners on the root rather than one listener on every rendered element. When a native event reaches the root, React determines the target fiber, builds the React propagation path, and invokes capture handlers top-down and bubble handlers bottom-up.

```tsx
function Search() {
  const onChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.currentTarget.value; // clear, stable data dependency
    queueMicrotask(() => console.log(value)); // no e.persist() in React 17+
  };
  return <input onChange={onChange} />;
}
```

Root-scoped delegation enables two independently managed React trees—or a React tree nested inside a non-React application—to respect propagation boundaries more predictably. `stopPropagation()` within an inner React 17+ tree can prevent the native event from reaching an outer root. React also listens on portal containers so portal events still propagate according to the React tree.

**Trade-offs and pitfalls.** “One root listener handles everything” is an approximation: some events have special handling or do not naturally bubble. React 17 stopped emulating bubbling for `onScroll`; focus handling uses native `focusin`/`focusout`; portals need listeners at their container. `return false` does not cancel a React event—call the explicit methods. Mixing native document listeners and synthetic listeners can produce ordering surprises, especially across capture/bubble phases. Test propagation when embedding multiple roots or legacy code.

**References:** [Official React 17 release post: delegation and no pooling](https://legacy.reactjs.org/blog/2020/08/10/react-v17-rc.html), [React `SyntheticEvent` reference](https://react.dev/reference/react-dom/components/common#react-event-object), [MDN: event bubbling](https://developer.mozilla.org/en-US/docs/Learn_web_development/Core/Scripting/Event_bubbling).

### 38. Explain how `useMemo` and `useCallback` cache values/references internally. How do you avoid performance anti-patterns by overusing them?

**Interview-ready answer.** Hook data is associated with a component's fiber in call order. Conceptually, `useMemo` stores `[calculatedValue, dependencyArray]`; on a later render React compares every dependency with its previous value using `Object.is`. If all match, it returns the stored value; otherwise it calls the calculation during render and stores the new pair. `useCallback(fn, deps)` performs the same dependency comparison but stores/returns `fn` itself—it does not call it. A useful mental equivalence is `useCallback(fn, deps) ≈ useMemo(() => fn, deps)`.

```tsx
const visibleRows = useMemo(
  () => expensiveFilter(rows, query),
  [rows, query]
);

const selectRow = useCallback(
  (id: string) => dispatch({ type: 'selected', id }),
  [dispatch]
);

return <MemoizedGrid rows={visibleRows} onSelect={selectRow} />;
```

This helps only when reference stability is consumed—for example by a `memo` child, another hook dependency, or a genuinely expensive recalculation. Creating a function is normally cheap. Memoization itself costs a hook record, dependency allocation/comparison, retained references, and cognitive complexity. A single always-new object can invalidate the entire optimization.

Apply it empirically:

1. Use React DevTools Profiler and browser performance traces to locate a meaningful bottleneck.
2. Keep state local and rendering pure; avoid effects that repeatedly set state.
3. Memoize the expensive calculation or boundary—not every literal and handler.
4. Make dependencies honest. Prefer an updater (`setTodos(t => ...)`) to avoid capturing state solely to calculate the next state.
5. Measure the production build again.

**Trade-offs and pitfalls.** `useMemo` is a performance optimization, not a semantic guarantee; React may discard caches in supported situations. Code must remain correct if recalculated. Omitting dependencies creates stale closures/results, while adding a freshly created object defeats caching—move that object inside the calculation or memoize it for a real reason. In development Strict Mode React may call calculations twice to expose impurities. Modern React Compiler can automate some memoization, but teams should follow the compiler's supported rules rather than mixing manual memoization indiscriminately.

**References:** [React `useMemo` reference](https://react.dev/reference/react/useMemo), [React `useCallback` reference](https://react.dev/reference/react/useCallback), [React DevTools Profiler](https://react.dev/reference/react/Profiler).

### 39. What are the common causes of React Context performance degradation? How do you optimize large context state trees to prevent global re-renders?

**Interview-ready answer.** When a provider's `value` differs from the previous value by `Object.is`, React schedules every descendant consumer of that context. `React.memo` cannot block a context value observed inside the component. The classic problem is a monolithic provider that constructs a new object and callbacks on every render:

```tsx
// Every provider render creates a new value and invalidates all consumers.
<AppContext.Provider value={{ user, theme, cart, setTheme, addToCart }}>
  {children}
</AppContext.Provider>
```

Common causes are oversized “global state” contexts, inline object/function values, high-frequency data (cursor position, live prices), placing a provider too high, and consumers that read a whole object when they need one field.

Use several complementary fixes:

```tsx
const CartStateContext = createContext<CartState | null>(null);
const CartDispatchContext = createContext<React.Dispatch<Action> | null>(null);

function CartProvider({ children }: React.PropsWithChildren) {
  const [state, dispatch] = React.useReducer(cartReducer, initialCart);
  // dispatch has stable identity; components needing only commands do not
  // subscribe to the changing state context.
  return (
    <CartDispatchContext.Provider value={dispatch}>
      <CartStateContext.Provider value={state}>
        {children}
      </CartStateContext.Provider>
    </CartDispatchContext.Provider>
  );
}
```

- Split contexts by change frequency/domain and separate state from dispatch.
- Memoize a provider object only when it truly must be an object: `useMemo(() => ({user, login}), [user, login])`; stabilize callbacks with honest dependencies.
- Put providers close to the subtree that owns the state and split a large consumer into a thin context-reading wrapper plus memoized children receiving primitive props.
- Normalize data and avoid changing untouched object branches.
- For large/high-frequency state, use a selector-capable external store built on subscriptions/`useSyncExternalStore` (Redux, Zustand, or a context-selector library), so only components whose selected result changed render.

**Trade-offs and pitfalls.** Splitting into dozens of arbitrary contexts can harm discoverability; split by ownership and update frequency. Memoizing the provider value does not help when one of its members changes on every update. Context is excellent for relatively low-frequency dependencies such as theme, locale, authenticated identity, and service objects; it is not automatically a full state-management solution. Profile before migrating.

**References:** [React `useContext` caveats](https://react.dev/reference/react/useContext), [React: Scaling Up with Reducer and Context](https://react.dev/learn/scaling-up-with-reducer-and-context), [React `useSyncExternalStore`](https://react.dev/reference/react/useSyncExternalStore).

### 40. How do you implement code-splitting and lazy loading natively using `React.lazy` and `Suspense`? Explain the bundle compilation strategy behind it.

**Interview-ready answer.** Pass a dynamic `import()` to `lazy`, then render the lazy component below a `Suspense` boundary. The promise must resolve to a module whose `default` export is a component. React caches both the loader promise and its resolution.

```tsx
import { lazy, Suspense } from 'react';

const ReportsPage = lazy(() => import('./pages/ReportsPage'));

export function AppRoutes() {
  return (
    <ErrorBoundary fallback={<p>Could not load reports.</p>}>
      <Suspense fallback={<ReportsSkeleton />}>
        <ReportsPage />
      </Suspense>
    </ErrorBoundary>
  );
}
```

At build time, a bundler such as webpack/Vite sees the static dynamic-import boundary and emits `ReportsPage` plus its non-shared dependencies into a separate chunk with a content hash. Shared modules may be extracted into vendor/common chunks according to the bundler's chunk graph. The initial bundle contains a small runtime mapping module IDs to chunk URLs. When React first renders `ReportsPage`, the loader starts fetching the chunk; `lazy` suspends by throwing/returning the pending thenable to React's Suspense machinery. The nearest boundary shows its fallback. After download, parse, evaluation, and module resolution, React retries the subtree.

Prefer route- and feature-level boundaries: enough to reduce startup JavaScript, but not hundreds of tiny requests. Use nested Suspense boundaries to avoid blanking the whole screen. Preload likely next routes on intent (hover/focus) through the router/bundler's supported API. Lazy-load browser-only features, editors, charts, or administrative modules that most users do not open.

**Trade-offs and pitfalls.** A chunk can fail because of offline/network errors or because a deployment removed an old hashed chunk while a user still has old HTML; Suspense handles waiting, not rejection, so use an error boundary and a reload/retry policy. An overly large fallback causes layout shift; reserve dimensions. Top-level declaration matters—declaring `lazy(...)` inside a component creates a different component and resets state. Native `React.lazy` is component-code loading, not a general-purpose data cache. SSR/streaming behavior depends on the framework and bundler, so use the framework's route and preload primitives in production.

**References:** [React `lazy` reference](https://react.dev/reference/react/lazy), [React `Suspense` reference](https://react.dev/reference/react/Suspense), [webpack code-splitting guide](https://webpack.js.org/guides/code-splitting/).

### 41. Compare React state management architectures: Context API vs. Redux Toolkit vs. Zustand. When does a slice-based approach outperform a single-store proxy?

**Interview-ready answer.** Choose based on update frequency, ownership, debugging requirements, and team scale—not line count alone.

| Option | Model and best fit | Main costs |
|---|---|---|
| Context + reducer | React-native dependency propagation; theme/auth/config or a bounded subtree | All consumers of a changed context are notified; no built-in selectors, middleware, server cache, or time-travel tooling |
| Redux Toolkit (RTK) | One explicit store organized into feature slices; immutable reducer actions, middleware, DevTools, selectors; RTK Query for server state | More conventions and concepts; careless broad selectors still rerender; global client state can be overused |
| Zustand | Small external stores with hook-based selector subscriptions and concise imperative actions | Fewer enforced architectural conventions; teams must define normalization, action/audit, SSR, and persistence rules carefully |

```ts
// Zustand: this component subscribes only to total, not the whole store.
const total = useCartStore(s => s.total);

// RTK: slices organize ownership; selectors determine render granularity.
const total = useAppSelector(s => s.cart.total);
```

The phrase “single-store proxy” needs correction: Zustand's standard store is subscription/selector based, not inherently a JavaScript Proxy. A single Zustand store can be perfectly granular if selectors return stable values. Conversely, splitting Redux into slices is primarily code ownership and reducer composition; slices do not physically create separate stores and do not inherently make reads faster.

A slice-based RTK architecture wins in a large enterprise domain when many teams need explicit event/action history, cross-feature workflows, normalized entities, deterministic reducers, middleware/telemetry, and stable public boundaries. It often outperforms a *coarsely consumed* context or store because components select small values and unchanged slices preserve references. A focused Zustand store often wins for a local interactive tool, transient UI state, or a team that values low ceremony. Keep remote server data in a query cache (RTK Query, framework cache, or another data-fetching library) rather than manually copying every response into client state.

**Trade-offs and pitfalls.** Avoid selecting `state => state` or constructing a new object without shallow/equality comparison. Multiple stores improve isolation but make atomic cross-store updates and debugging harder. Persisted state needs versioned migrations and must not contain secrets. SSR applications must create request-scoped stores to prevent user data leaking between requests.

**References:** [Redux Toolkit: why RTK is the recommended Redux approach](https://redux-toolkit.js.org/introduction/why-rtk-is-redux-today), [Redux Toolkit `createSlice`](https://redux-toolkit.js.org/api/createSlice), [Zustand documentation](https://zustand.docs.pmnd.rs/).

### 42. How does `useEffect` cleanup execution work? Explain the memory footprint lifecycle when dealing with active event listeners or open WebSockets.

**Interview-ready answer.** An effect is a synchronization process with an external system. After a commit that changes a dependency, React first invokes the *previous* cleanup with the old captured values, then invokes the new setup. On unmount it invokes the last cleanup. In development Strict Mode, React intentionally runs an extra setup → cleanup → setup cycle on mount to expose asymmetric cleanup; production does not perform that diagnostic cycle.

```tsx
function LiveOrders({ accountId }: { accountId: string }) {
  React.useEffect(() => {
    const socket = new WebSocket(`/ws/orders?account=${accountId}`);
    const onMessage = (event: MessageEvent) => updateOrders(event.data);
    socket.addEventListener('message', onMessage);

    return () => {
      socket.removeEventListener('message', onMessage);
      socket.close(1000, 'component stopped observing this account');
    };
  }, [accountId]);
  return null;
}
```

The browser/native resource—not the effect function itself—is usually the important footprint. A registered listener retains its callback, whose closure may retain props, state, DOM nodes, and large graphs. A WebSocket retains buffers, handlers, TCP/TLS state, and server-side session resources. A timer, observer, subscription, or unresolved request likewise has an external root that can keep otherwise unreachable objects alive. Cleanup severs these roots and prevents stale callbacks from updating the wrong screen.

For a changing `accountId`, the exact lifecycle is: commit A → open A; commit B → close A using A's closure → open B; unmount → close B. Capture mutable refs used by cleanup into a local variable during setup if the ref might later become `null` or point elsewhere.

**Trade-offs and pitfalls.** Handler removal requires the same function identity and compatible listener options. Do not put a freshly created dependency object in the array; it reconnects every render. Implement WebSocket reconnect with bounded exponential backoff, jitter, heartbeat/idle detection, authentication renewal, and cancellation of pending reconnect timers. Closing a client does not prove the server processed every sent message; application-level acknowledgements may be required. Abort fetches with `AbortController`, but also guard against stale completions because not every async API is abortable.

**References:** [React `useEffect` lifecycle](https://react.dev/reference/react/useEffect), [React: Synchronizing with Effects](https://react.dev/learn/synchronizing-with-effects), [MDN WebSocket `close()`](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket/close).

### 43. How do you write a production-ready custom hook to handle robust API fetching with automated retry mechanisms, caching, and race-condition cancellation?

**Interview-ready answer.** A robust hook needs an explicit cache key, finite freshness, in-flight deduplication, cancellation when the last subscriber leaves, stale-response protection, retry rules, and observable loading/error states. Retries should be limited to transient failures and idempotent operations; never blindly repeat a payment `POST` without an idempotency key.

This compact TypeScript implementation caches successful GETs, shares an in-flight request, reference-counts consumers, aborts when none remain, and uses capped exponential backoff with jitter:

```tsx
import * as React from 'react';

type Entry<T> = { data: T; expiresAt: number };
type Flight<T> = {
  promise: Promise<T>; controller: AbortController; users: number;
};
const cache = new Map<string, Entry<unknown>>();
const flights = new Map<string, Flight<unknown>>();

const sleep = (ms: number, signal: AbortSignal) => new Promise<void>((ok, no) => {
  const id = setTimeout(ok, ms);
  signal.addEventListener('abort', () => {
    clearTimeout(id); no(new DOMException('Aborted', 'AbortError'));
  }, { once: true });
});

async function getJson<T>(url: string, signal: AbortSignal, retries: number): Promise<T> {
  for (let attempt = 0; ; attempt++) {
    try {
      const r = await fetch(url, { signal, headers: { Accept: 'application/json' } });
      if (!r.ok) {
        const retryable = r.status === 408 || r.status === 429 || r.status >= 500;
        throw Object.assign(new Error(`HTTP ${r.status}`), { retryable });
      }
      return await r.json() as T;
    } catch (e) {
      if (signal.aborted || attempt >= retries ||
          (e instanceof Error && 'retryable' in e && !(e as any).retryable)) throw e;
      const cap = Math.min(10_000, 250 * 2 ** attempt);
      await sleep(Math.random() * cap, signal); // full jitter
    }
  }
}

function acquire<T>(key: string, url: string, retries: number) {
  let f = flights.get(key) as Flight<T> | undefined;
  if (!f) {
    const controller = new AbortController();
    let created!: Flight<T>;
    const promise = getJson<T>(url, controller.signal, retries)
      .finally(() => {
        // Do not let an old request delete a newer flight for the same key.
        if (flights.get(key) === created) flights.delete(key);
      });
    created = { promise, controller, users: 0 };
    f = created;
    flights.set(key, f as Flight<unknown>);
  }
  f.users++;
  return {
    promise: f.promise,
    release: () => {
      if (--f!.users === 0) {
        if (flights.get(key) === f) flights.delete(key);
        f!.controller.abort();
      }
    }
  };
}

export function useApi<T>(key: string | null, url: string, {
  staleMs = 30_000, retries = 3
} = {}) {
  const [nonce, retry] = React.useReducer(n => n + 1, 0);
  const [state, setState] = React.useState<{
    data?: T; error?: Error; loading: boolean
  }>({ loading: !!key });

  React.useEffect(() => {
    if (!key) { setState({ loading: false }); return; }
    const hit = cache.get(key) as Entry<T> | undefined;
    if (hit && hit.expiresAt > Date.now()) {
      setState({ data: hit.data, loading: false }); return;
    }

    let active = true; // protects against old key completing after a new key
    setState(s => ({ data: s.data, loading: true }));
    const request = acquire<T>(key, url, retries);
    request.promise.then(data => {
      cache.set(key, { data, expiresAt: Date.now() + staleMs });
      if (active) setState({ data, loading: false });
    }).catch(error => {
      if (active && error?.name !== 'AbortError') setState({ error, loading: false });
    });
    return () => { active = false; request.release(); };
  }, [key, url, staleMs, retries, nonce]);

  return {
    ...state,
    retry,
    invalidate: () => { if (key) { cache.delete(key); retry(); } }
  };
}
```

In a real application also parse `Retry-After`, validate JSON/schema, cap response size, record latency/retry metrics, define stale-while-revalidate behavior, and use an error boundary where appropriate. Cache keys must include every response-shaping input, including tenant/user scope; otherwise data can leak across users. On logout, clear user-scoped entries.

**Trade-offs and pitfalls.** This in-memory cache is per tab and unbounded; add LRU/size eviction or use a proven query library for a large app. Aborting after the server received a write does not undo it. Retrying non-idempotent operations needs a server-enforced idempotency key. Automated retries multiply load during incidents, so use jitter, a retry budget, and perhaps a circuit breaker. Framework data loaders/RTK Query/TanStack Query usually provide better SSR, focus revalidation, garbage collection, and mutation coordination than maintaining this code yourself.

**References:** [React guidance on fetching and race-condition cleanup](https://react.dev/reference/react/useEffect#fetching-data-with-effects), [MDN `AbortController`](https://developer.mozilla.org/en-US/docs/Web/API/AbortController), [HTTP Semantics: idempotent methods and `Retry-After`](https://www.rfc-editor.org/rfc/rfc9110.html).

## Part 1: Micro-Frontends and Performance Optimization

### 44. Explain the architecture of Micro-Frontends using Webpack 5 Module Federation. How do apps share dependencies dynamically at runtime?

**Interview-ready answer.** Module Federation lets separately built and deployed webpack applications resolve modules from one another at runtime. A **remote** publishes a small manifest/runtime called `remoteEntry.js` and exposes named modules; a **host** loads that container and requests an exposed module. Each build still owns its chunk graph, public path, and deployment. It is runtime composition, not source-code copying.

```js
// catalog/webpack.config.js (remote)
new ModuleFederationPlugin({
  name: 'catalog',
  filename: 'remoteEntry.js',
  exposes: { './ProductCard': './src/ProductCard' },
  shared: {
    react: { singleton: true, requiredVersion: '^19.0.0' },
    'react-dom': { singleton: true, requiredVersion: '^19.0.0' }
  }
});

// shell/webpack.config.js (host)
new ModuleFederationPlugin({
  name: 'shell',
  remotes: {
    catalog: 'catalog@https://cdn.example.com/catalog/remoteEntry.js'
  },
  shared: {
    react: { singleton: true, requiredVersion: '^19.0.0' },
    'react-dom': { singleton: true, requiredVersion: '^19.0.0' }
  }
});

// shell application—an async boundary triggers remote loading.
const ProductCard = React.lazy(() => import('catalog/ProductCard'));
```

At startup/import time, the host downloads the remote container. Webpack initializes a **share scope** containing providers and versions of shared packages. The host and remote register their provided versions; the runtime selects a compatible version according to `requiredVersion`, `singleton`, and loading order, then the container's `get('./ProductCard')` factory loads its additional chunks. Marking React singleton is crucial: two React instances can cause invalid hook calls and split context identity. Shared dependencies are negotiable runtime modules, not a guarantee that only one physical version will ever load—configuration and compatibility still matter.

A sound production architecture also defines routing ownership, a small versioned contract (props/events rather than another team's store internals), an error/loading boundary around every remote, design tokens, authentication capability interfaces, independent smoke/contract tests, and observability tagged with remote name/version.

**Trade-offs and pitfalls.** Independent deployment moves failures to runtime: a missing entry, incompatible API, CSP/CORS problem, or bad remote can break a route. Pin or promote immutable remote manifests, support rollback, use timeouts/fallbacks, constrain allowed origins and CSP, and treat remote code as fully trusted executable code. Too many remotes create request waterfalls, duplicated dependencies, inconsistent UX, and distributed ownership overhead. Avoid `eager: true` unless startup mechanics require it because it can inflate the initial bundle. Module Federation is justified for genuinely autonomous teams/deployments; a modular monolith or package-based monorepo is simpler when releases remain coordinated.

**References:** [webpack Module Federation concepts](https://webpack.js.org/concepts/module-federation/), [webpack Module Federation plugin options](https://webpack.js.org/plugins/module-federation-plugin/), [webpack public-path guidance for federated modules](https://webpack.js.org/concepts/module-federation/#dynamic-public-path).

### 45. How do you audit and optimize Core Web Vitals (LCP, FID, CLS, INP) for a highly interactive, enterprise-grade React SPA?

**Interview-ready answer.** Start with field data, segment it, diagnose in a reproducible lab trace, fix the largest cause, and enforce a regression budget. As of 2024, **FID is retired as a Core Web Vital and INP replaced it**; FID remains useful only when interpreting historical dashboards. Current “good” thresholds at the 75th percentile, evaluated separately for mobile and desktop, are LCP ≤ 2.5 s, INP ≤ 200 ms, and CLS ≤ 0.1.

```ts
import { onCLS, onINP, onLCP } from 'web-vitals';

const report = (metric: unknown) => {
  const body = JSON.stringify({ metric, route: location.pathname, release: APP_SHA });
  navigator.sendBeacon('/rum/vitals', body) ||
    fetch('/rum/vitals', { method: 'POST', body, keepalive: true });
};
onLCP(report); onINP(report); onCLS(report);
```

**Audit workflow.** Use RUM/web-vitals and CrUX/PageSpeed Insights for real devices; segment by route, geography, device class, browser, tenant, and release. Averages conceal the harmed tail, so inspect p75/p95. Use Chrome Performance panel and React Profiler for the same slow user journey. Lighthouse/TBT is useful in CI, but a synthetic no-interaction run cannot measure field INP. Add bundle analysis and source maps, and correlate long tasks/resource waterfalls with releases.

**Targeted fixes.** For **LCP**, reduce TTFB via CDN/caching or pre-rendering; make the hero resource discoverable in initial HTML; compress/resize responsive images; use `fetchpriority="high"` or preload sparingly; inline only critical CSS; defer third-party/noncritical JS. Never lazy-load the actual LCP image. For **INP**, reduce initial JS, route-split, virtualize huge lists, avoid synchronous storage/layout thrashing, move CPU work to Web Workers, debounce nonvisual work, and break/yield long tasks. Keep event handlers small; `startTransition` can deprioritize nonurgent React rendering but cannot interrupt arbitrary expensive JavaScript. FID measured only the delay before the first handler; INP includes input delay, handler processing, and presentation delay across interactions.

For **CLS**, give images/video dimensions or `aspect-ratio`, reserve space for banners/skeletons, do not inject content above existing content, animate transforms rather than layout properties, and use font metrics/fallbacks plus carefully chosen preload. Set route-level budgets for JS bytes, long tasks, and vital p75; run canaries and compare RUM before broad rollout.

**Trade-offs and pitfalls.** Optimizing a desktop lab score can worsen low-end mobile users. Preloading everything competes with the LCP resource. SSR may improve discovery but add server TTFB and hydration CPU. A fast skeleton that later jumps can improve perceived speed while damaging CLS. Optimize real journeys, accessibility, and business outcomes—not a single green score.

**References:** [Google web.dev: current Core Web Vitals and thresholds](https://web.dev/articles/vitals), [web.dev: optimize INP](https://web.dev/articles/optimize-inp), [Chrome DevTools performance tooling](https://developer.chrome.com/docs/devtools/performance/).

### 46. Explain the operational differences, SEO impacts, and infrastructure trade-offs between CSR, SSR (Next.js), and Static Site Generation (SSG).

**Interview-ready answer.** The distinction is *when and where initial HTML is produced*; all three can become interactive after client JavaScript loads.

| Model | Initial response | Best use | Main operational cost |
|---|---|---|---|
| CSR | Minimal shell; browser downloads JS, fetches data, renders | Authenticated dashboards and highly interactive apps where indexing is secondary | CDN/static hosting is simple, but startup depends on JS/network/device CPU |
| SSR | Server renders HTML per request (often streamed), then client hydrates | Personalized or frequently changing indexable pages | Compute on every miss, capacity/cold starts, dependency latency, cache variation, more failure modes |
| SSG | HTML/data payload generated at build time (or revalidation) and served from CDN | Marketing, docs, catalog/blog pages with bounded change rate | Build duration, invalidation/revalidation, and potentially stale content |

**SEO.** SSG and SSR put meaningful content, status codes, headings, links, and metadata in the first response, which improves crawler reliability and sharing previews. Major crawlers can execute JavaScript, so CSR is not automatically “invisible,” but rendering can be delayed or unavailable to smaller crawlers, and client-only metadata/status handling is weaker. SEO also depends on canonical URLs, crawlable links, structured data, robots rules, performance, and content—not just rendering mode.

**Performance and operation.** CSR offers cheap static delivery and smooth later navigation but risks a blank shell, large JS, request waterfalls, and poor low-end-device performance. SSR can improve first content and LCP resource discovery, but the user still needs hydration/event code; large hydration work can hurt INP. It requires Node/serverless/edge runtime capacity, timeouts, tracing, secure server data access, and a caching strategy. SSG is fastest and most resilient at the edge, but rebuilding millions of pages is costly and a pricing/inventory page can be stale. Incremental/static revalidation is the hybrid: serve cached HTML and regenerate under an explicit freshness policy.

Next.js applications normally mix modes by route and component: static marketing pages, dynamically rendered account pages, cached server components, and client components only where state/browser APIs are needed. Avoid treating “SSR” as “render everything on every request.”

**Trade-offs and pitfalls.** Hydration requires server and first-client output to match; time-dependent/random/browser-only render logic causes mismatch. Never cache personalized SSR HTML under a shared key, and set correct `Vary`/private cache directives. SSG must have an emergency purge/revalidation path. Choose by freshness, personalization, crawlability, traffic shape, latency SLO, and failure tolerance, then measure field performance.

**References:** [Next.js: Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components), [Next.js: static exports](https://nextjs.org/docs/app/guides/static-exports), [Google Search: JavaScript SEO basics](https://developers.google.com/search/docs/crawling-indexing/javascript/javascript-seo-basics).

## Part 2: Low-Level Design and Object-Oriented Design

### 47. Provide a practical production example where breaking the Interface Segregation Principle (ISP) ruins API client maintainability. How do you fix it?

**Interview-ready answer.** ISP says a client should not be forced to depend on methods it does not use. A realistic violation is a “universal” payment-provider SDK interface:

```java
interface PaymentProviderClient {
    Authorization authorize(Card card, Money amount);
    Capture capture(String authorizationId, Money amount);
    Refund refund(String chargeId, Money amount);
    PayoutStatus getPayout(String payoutId);
    WalletAddress createCryptoWallet(String network);
    void submitThreeDSResult(String paymentId, ThreeDSResult result);
}

final class CryptoClient implements PaymentProviderClient {
    // Crypto has no card authorization, 3DS, or provider payout.
    public Authorization authorize(Card c, Money m) {
        throw new UnsupportedOperationException();
    }
    // ...more meaningless stubs...
}
```

In production this becomes more than ugly code. A Stripe payout API version change modifies the shared interface, forcing PayPal/Crypto adapters and all their mocks to recompile. Generic callers see methods that are invalid for the configured provider and discover capability errors only at runtime. Test doubles implement twenty irrelevant operations. Provider-specific authentication, retry rules, and data models leak into every consumer; a “harmless” method addition becomes an organization-wide release.

Define cohesive, consumer-facing roles and adapt vendor SDKs behind them:

```java
interface PaymentAuthorizer {
    Authorization authorize(AuthorizeCommand command);
}
interface Capturer {
    Capture capture(CaptureCommand command);
}
interface Refunder {
    Refund refund(RefundCommand command);
}
interface PayoutReader {
    PayoutStatus findPayout(String payoutId);
}

// This adapter implements only capabilities Stripe actually supplies.
final class StripeAdapter
        implements PaymentAuthorizer, Capturer, Refunder, PayoutReader {
    private final StripeSdk sdk;
    StripeAdapter(StripeSdk sdk) { this.sdk = sdk; }
    public Authorization authorize(AuthorizeCommand c) { /* translate */ return null; }
    public Capture capture(CaptureCommand c) { /* translate */ return null; }
    public Refund refund(RefundCommand c) { /* translate */ return null; }
    public PayoutStatus findPayout(String id) { /* translate */ return null; }
}

final class RefundService {
    private final Refunder refunds;       // least-powerful dependency
    RefundService(Refunder refunds) { this.refunds = refunds; }
    Refund issue(RefundCommand c) { return refunds.refund(c); }
}
```

At an HTTP boundary, ISP likewise means resources/capability-specific clients such as `RefundsApi` and `PayoutsApi`, generated from separated OpenAPI tags/modules where practical. Keep vendor DTOs inside adapters and expose stable domain commands/results. Contract tests then target only the capability an adapter claims.

**Trade-offs and pitfalls.** ISP does not mean “one interface per method” automatically. Over-segregation creates navigation and wiring overhead and can split operations that share one cohesive transaction contract. Group methods by client role and reason to change. If capability is genuinely dynamic, make it explicit during provider selection (`supports(REFUND)`) rather than returning `null` or throwing deep in a call. Composition is usually clearer than a deep adapter hierarchy.

**References:** [Robert C. Martin's original Interface Segregation Principle paper](https://d3s.mff.cuni.cz/f/teaching/nprg043/extras/martin96-interface_segregation_principle.pdf), [Oracle `ServiceLoader` guidance for service/provider abstractions](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ServiceLoader.html).

### 48. Explain the Liskov Substitution Principle (LSP). How does subclassing a square from a rectangle break LSP? What is the correct object-oriented fix?

**Interview-ready answer.** LSP is behavioral subtyping: if `S` is a subtype of `T`, code proved correct against `T` must remain correct when given `S`. A subtype may not strengthen preconditions, weaken postconditions, violate invariants, or introduce incompatible observable/history behavior.

The mutable rectangle contract usually says width and height can be changed independently:

```java
class Rectangle {
    private int width, height;
    void setWidth(int width) { this.width = width; }
    void setHeight(int height) { this.height = height; }
    int area() { return width * height; }
}

class Square extends Rectangle {
    @Override void setWidth(int x) { super.setWidth(x); super.setHeight(x); }
    @Override void setHeight(int x) { super.setWidth(x); super.setHeight(x); }
}

static void resizeForBanner(Rectangle r) {
    r.setWidth(5);
    r.setHeight(4);
    if (r.area() != 20) throw new AssertionError(); // Square gives 16
}
```

`Square` preserves its own invariant but violates `Rectangle`'s independent-dimension postconditions. Making only one setter synchronize both sides does not fix it; it merely makes behavior depend on call order. The mathematical “is-a” relation is insufficient because the *mutable software contracts* differ.

The clean fix is not to inherit between these mutable types. Model their common capability at the abstraction clients actually need, preferably with immutable values:

```java
sealed interface Shape permits Rectangle, Square {
    double area();
}

record Rectangle(double width, double height) implements Shape {
    Rectangle {
        if (width <= 0 || height <= 0) throw new IllegalArgumentException();
    }
    public double area() { return width * height; }
    Rectangle withWidth(double w) { return new Rectangle(w, height); }
    Rectangle withHeight(double h) { return new Rectangle(width, h); }
}

record Square(double side) implements Shape {
    Square { if (side <= 0) throw new IllegalArgumentException(); }
    public double area() { return side * side; }
    Square withSide(double s) { return new Square(s); }
}
```

If a downstream renderer needs a bounding box, expose `Bounds bounds()` or convert a square to `new Rectangle(side, side)` without claiming the square supports rectangle mutation. Another valid design is an immutable `Rectangle(width,height)` plus a `Rectangle.square(side)` factory when no distinct `Square` behavior is needed.

**Trade-offs and pitfalls.** Removing setters is not a magic LSP proof; all observable contracts—including exceptions, timing, nullability, ordering, and thread safety—matter. A repository subtype that throws for `delete`, or a “read-only list” subtype whose inherited `add` throws, has the same smell. Prefer small capability interfaces, composition, and contract tests that run against every implementation.

**References:** [Liskov and Wing: Specifications and Their Use in Defining Subtypes](https://www.cs.cmu.edu/~wing/publications/LiskovWing93a.pdf), [Java sealed classes/interfaces specification](https://docs.oracle.com/javase/specs/jls/se21/html/jls-8.html#jls-8.1.6).

### 49. Implement the Strategy Pattern and Factory Pattern concurrently to build an extensible payment processing module supporting Stripe, PayPal, and Crypto.

**Interview-ready answer.** Each provider is a **Strategy** implementing one stable payment contract. A **Factory/registry** selects the strategy from runtime payment method/configuration. Adding a provider means registering another strategy, not modifying checkout conditionals.

```java
import java.math.BigDecimal;
import java.net.URI;
import java.util.*;
import java.util.function.Function;
import static java.util.stream.Collectors.toUnmodifiableMap;

enum PaymentMethod { STRIPE, PAYPAL, CRYPTO }
enum PaymentStatus { SUCCEEDED, PENDING, FAILED }

record Money(BigDecimal amount, Currency currency) {
    Money {
        Objects.requireNonNull(amount); Objects.requireNonNull(currency);
        if (amount.signum() <= 0) throw new IllegalArgumentException("amount > 0");
    }
}
record PaymentCommand(
        UUID orderId, PaymentMethod method, Money money,
        String paymentToken, String idempotencyKey) {
    PaymentCommand {
        Objects.requireNonNull(orderId); Objects.requireNonNull(method);
        Objects.requireNonNull(money); Objects.requireNonNull(idempotencyKey);
    }
}
record PaymentResult(String providerReference, PaymentStatus status, URI nextAction) {}

interface PaymentStrategy {
    PaymentMethod method();
    PaymentResult pay(PaymentCommand command) throws PaymentException;
}

final class StripeStrategy implements PaymentStrategy {
    private final StripeGateway gateway;
    StripeStrategy(StripeGateway gateway) { this.gateway = gateway; }
    public PaymentMethod method() { return PaymentMethod.STRIPE; }
    public PaymentResult pay(PaymentCommand c) {
        var r = gateway.createPaymentIntent(
                c.money(), c.paymentToken(), c.idempotencyKey(), c.orderId());
        return new PaymentResult(r.id(), map(r.status()), r.nextAction());
    }
    private PaymentStatus map(String s) { /* exhaustive vendor-status mapping */ return PaymentStatus.PENDING; }
}

final class PayPalStrategy implements PaymentStrategy {
    private final PayPalGateway gateway;
    PayPalStrategy(PayPalGateway gateway) { this.gateway = gateway; }
    public PaymentMethod method() { return PaymentMethod.PAYPAL; }
    public PaymentResult pay(PaymentCommand c) {
        var order = gateway.createOrder(c.money(), c.idempotencyKey(), c.orderId());
        return new PaymentResult(order.id(), PaymentStatus.PENDING, order.approvalUrl());
    }
}

final class CryptoStrategy implements PaymentStrategy {
    private final ChainGateway gateway;
    CryptoStrategy(ChainGateway gateway) { this.gateway = gateway; }
    public PaymentMethod method() { return PaymentMethod.CRYPTO; }
    public PaymentResult pay(PaymentCommand c) {
        var invoice = gateway.createInvoice(c.money(), c.idempotencyKey(), c.orderId());
        return new PaymentResult(invoice.id(), PaymentStatus.PENDING, invoice.paymentUri());
    }
}

final class PaymentStrategyFactory {
    private final Map<PaymentMethod, PaymentStrategy> strategies;
    PaymentStrategyFactory(Collection<PaymentStrategy> implementations) {
        this.strategies = implementations.stream().collect(toUnmodifiableMap(
                PaymentStrategy::method, Function.identity(),
                (a, b) -> { throw new IllegalStateException("duplicate " + a.method()); }));
    }
    PaymentStrategy forMethod(PaymentMethod method) {
        var result = strategies.get(method);
        if (result == null) throw new UnsupportedOperationException("No strategy for " + method);
        return result;
    }
}

final class PaymentService {
    private final PaymentStrategyFactory factory;
    private final PaymentAttemptRepository attempts;
    PaymentService(PaymentStrategyFactory f, PaymentAttemptRepository r) { factory=f; attempts=r; }

    PaymentResult process(PaymentCommand c) {
        return attempts.findResult(c.idempotencyKey()).orElseGet(() -> {
            attempts.reserve(c.idempotencyKey(), c.orderId(), c.money()); // unique DB key
            PaymentResult result = factory.forMethod(c.method()).pay(c);
            attempts.record(c.idempotencyKey(), result);
            return result;
        });
    }
}
```

`StripeGateway`, `PayPalGateway`, and `ChainGateway` are anti-corruption adapters around actual SDKs. Spring can inject `List<PaymentStrategy>` into the factory. The local `payment_attempt` table needs a unique idempotency key/request fingerprint because provider idempotency alone does not coordinate application retries. Treat webhooks/chain confirmations as authenticated, idempotently consumed state transitions; a timeout does not prove failure, and crypto normally remains `PENDING` until enough confirmations.

**Trade-offs and pitfalls.** Do not force all providers into fake synchronous success semantics. Model redirect/3DS approval, asynchronous settlement, refund capability, and status explicitly. Use integer minor units or validated `BigDecimal` rules; never `double`. Redact tokens, isolate secrets, set timeouts, retry only safely, reconcile uncertain attempts, and verify webhook signatures. A factory switch is acceptable for three fixed implementations, but a registry supports extension without modifying the factory.

**References:** [Oracle `ServiceLoader` provider mechanism](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ServiceLoader.html), [Stripe idempotent request guidance](https://docs.stripe.com/api/idempotent_requests), [PayPal Orders API](https://developer.paypal.com/docs/api/orders/v2/).

### 50. How does the Observer Pattern differ structurally from a modern Pub/Sub architecture? Show code representations for both.

**Interview-ready answer.** In classic **Observer**, a subject owns direct references to observers and invokes them, usually synchronously in the same process. In **Publish/Subscribe**, publishers address a topic/event type through an intermediary broker; publishers and subscribers do not know each other, messages cross serialization/network boundaries, and the broker defines routing, retention, delivery, ordering, retries, and backpressure.

```java
// Observer: direct, in-memory, object-level coupling.
interface OrderObserver { void onChanged(OrderSnapshot order); }

final class OrderSubject {
    private final List<OrderObserver> observers =
            new java.util.concurrent.CopyOnWriteArrayList<>();
    void subscribe(OrderObserver o) { observers.add(Objects.requireNonNull(o)); }
    void unsubscribe(OrderObserver o) { observers.remove(o); }
    void publish(OrderSnapshot value) {
        for (var o : observers) {
            try { o.onChanged(value); }
            catch (RuntimeException ex) { /* record; define isolation policy */ }
        }
    }
}
```

The subject and observers share types, memory, lifecycle, and often a call stack. Delivery is immediate, but a slow observer blocks the publisher unless explicitly scheduled; there is normally no durable replay after a crash.

```java
// Pub/Sub producer: knows topic and schema, not consumer objects.
record OrderCreated(UUID eventId, UUID orderId, long occurredAt, int schemaVersion) {}

final class OrderEvents {
    private final org.apache.kafka.clients.producer.KafkaProducer<String, OrderCreated> producer;
    java.util.concurrent.CompletionStage<?> publish(OrderCreated e) {
        var record = new org.apache.kafka.clients.producer.ProducerRecord<>(
                "orders.created.v1", e.orderId().toString(), e);
        return producer.send(record); // inspect/handle completion in production
    }
}

// Independent subscriber process (outline).
while (running) {
    var records = consumer.poll(java.time.Duration.ofMillis(500));
    for (var record : records) {
        inboxRepository.runOnce(record.value().eventId(), () ->
                emailService.persistRequestedEmail(record.value()));
    }
    consumer.commitSync(); // only after durable side effect/inbox commit
}
```

Pub/sub is decoupled in location and time: multiple consumer groups can independently replay the log. The remaining coupling is the event contract/topic and its semantic evolution, so use schema compatibility, ownership, event IDs, and observability.

**Trade-offs and pitfalls.** An in-process “event bus” with topic strings may be pub/sub structurally but has none of a durable broker's failure guarantees. Distributed pub/sub adds duplicates, out-of-order delivery across partitions, lag, poison messages, schema evolution, and eventual consistency. Consumers must be idempotent; committing before the side effect loses work, while committing afterward permits replay. Observer is ideal for local UI/domain notifications with controlled lifecycles; a broker is justified across services/deployments. Avoid publishing to Kafka directly beside a database update without an outbox, or a crash can leave only one side committed.

**References:** [Java `Flow` reactive-streams interfaces](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/Flow.html), [Apache Kafka design and delivery semantics](https://kafka.apache.org/43/design/design/), [RabbitMQ exchanges and routing](https://www.rabbitmq.com/docs/exchanges).

### 51. Design a thread-safe, double-checked locking Singleton pattern in Java that is entirely safe against Reflection, Serialization, and Cloning attacks.

**Interview-ready answer.** The honest answer is that **double-checked locking (DCL) can guarantee safe lazy publication, but an ordinary class cannot be “entirely safe” against fully privileged reflection/`Unsafe`/instrumentation**. The language-supported answer for one JVM/class-loader is an enum singleton: Java prohibits reflective enum construction, serialization preserves the enum constant, and `Enum.clone()` is final. It is thread-safe by class initialization.

```java
public enum ApplicationRegistry {
    INSTANCE;

    private final java.util.concurrent.ConcurrentMap<String, Object> values =
            new java.util.concurrent.ConcurrentHashMap<>();

    public void put(String key, Object value) { values.put(key, value); }
    public Object get(String key) { return values.get(key); }
}
```

If the interviewer explicitly wants DCL, show why `volatile` is mandatory and then explain the security boundary:

```java
public final class DclSingleton implements java.io.Serializable {
    @java.io.Serial private static final long serialVersionUID = 1L;
    private static volatile DclSingleton instance;
    private static final java.util.concurrent.atomic.AtomicBoolean CONSTRUCTED =
            new java.util.concurrent.atomic.AtomicBoolean();

    private DclSingleton() {
        if (!CONSTRUCTED.compareAndSet(false, true))
            throw new IllegalStateException("Singleton already constructed");
    }

    public static DclSingleton getInstance() {
        DclSingleton local = instance;       // one volatile read on fast path
        if (local == null) {
            synchronized (DclSingleton.class) {
                local = instance;
                if (local == null) instance = local = new DclSingleton();
            }
        }
        return local;
    }

    @java.io.Serial
    private Object readResolve() { return getInstance(); }

    @Override
    protected final Object clone() throws CloneNotSupportedException {
        throw new CloneNotSupportedException("singleton");
    }
}
```

`volatile` prevents publication of the reference from being observed before construction is visible and supplies the required happens-before relationship. The inner check prevents a second instance after two threads wait on the monitor. `readResolve` replaces a deserialized object, a final class prevents a cloneable subclass, and the constructor guard blocks ordinary repeat reflective construction. However, reflection invoked *before* normal creation can construct the one guarded object and cause denial of service; sufficiently privileged code can mutate the guard/fields or allocate without a constructor. Thus this is hardening, not an absolute security claim.

**Trade-offs and pitfalls.** Singleton scope is per class loader, not per distributed service or necessarily per application server. Serialization filters/module boundaries reduce attack surface but do not make hostile code in the same trusted JVM harmless. Singletons hide dependencies and complicate tests; dependency-injection singleton scope is often better. Use enum when the identity guarantee matters; use the initialization-on-demand holder idiom when lazy ordinary-class construction is needed without DCL.

**References:** [Java Language Specification §8.9 enum guarantees](https://docs.oracle.com/javase/specs/jls/se21/html/jls-8.html#jls-8.9), [Java Object Serialization Specification: enum serialization](https://docs.oracle.com/en/java/javase/21/docs/specs/serialization/serial-arch.html#serialization-of-enum-constants), [JLS §17.4 memory model and happens-before](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html#jls-17.4).

### 52. Write an elegant implementation of the Decorator Pattern to dynamically add logging, encryption, and compression layers onto an I/O data stream.

**Interview-ready answer.** Java stream filters are a native Decorator example: each wrapper has the same `InputStream`/`OutputStream` abstraction, delegates to another stream, and adds behavior. Order matters. For storage, a sensible write pipeline is plaintext metering/logging → compression → authenticated encryption → file; reading reverses it: file → decryption → decompression → metering/application.

```java
import javax.crypto.*;
import javax.crypto.spec.GCMParameterSpec;
import javax.crypto.SecretKey;
import java.io.*;
import java.nio.file.*;
import java.security.SecureRandom;
import java.util.concurrent.atomic.LongAdder;
import java.util.zip.*;

final class MeteredOutputStream extends FilterOutputStream {
    private final LongAdder bytes = new LongAdder();
    MeteredOutputStream(OutputStream out) { super(out); }
    @Override public void write(int b) throws IOException { out.write(b); bytes.increment(); }
    @Override public void write(byte[] b, int off, int len) throws IOException {
        out.write(b, off, len); bytes.add(len);
    }
    @Override public void close() throws IOException {
        try { super.close(); }
        finally { System.getLogger("secure-io").log(
                System.Logger.Level.INFO, "plaintext bytes written={0}", bytes.sum()); }
    }
}

final class SecureStreams {
    private static final byte[] MAGIC = { 'S', 'I', 'O', 1 };
    private static final SecureRandom RNG = new SecureRandom();

    static OutputStream encryptedCompressed(Path file, SecretKey key) throws Exception {
        OutputStream sink = Files.newOutputStream(file, StandardOpenOption.CREATE_NEW);
        try {
            byte[] iv = new byte[12]; RNG.nextBytes(iv); // unique IV per AES-GCM key
            DataOutputStream header = new DataOutputStream(sink);
            header.write(MAGIC); header.writeByte(iv.length); header.write(iv); header.flush();

            Cipher aes = Cipher.getInstance("AES/GCM/NoPadding");
            aes.init(Cipher.ENCRYPT_MODE, key, new GCMParameterSpec(128, iv));
            OutputStream encrypted = new CipherOutputStream(sink, aes);
            OutputStream compressed = new GZIPOutputStream(encrypted, 64 * 1024);
            return new MeteredOutputStream(compressed); // caller closes outermost
        } catch (Exception e) { sink.close(); throw e; }
    }

    static InputStream decryptedDecompressed(Path file, SecretKey key) throws Exception {
        InputStream source = Files.newInputStream(file);
        try {
            DataInputStream header = new DataInputStream(source);
            if (!java.util.Arrays.equals(header.readNBytes(4), MAGIC))
                throw new StreamCorruptedException("bad header");
            int n = header.readUnsignedByte();
            if (n != 12) throw new StreamCorruptedException("bad IV length");
            byte[] iv = header.readNBytes(n);

            Cipher aes = Cipher.getInstance("AES/GCM/NoPadding");
            aes.init(Cipher.DECRYPT_MODE, key, new GCMParameterSpec(128, iv));
            return new GZIPInputStream(new CipherInputStream(source, aes), 64 * 1024);
        } catch (Exception e) { source.close(); throw e; }
    }
}

// try-with-resources finalizes GZIP and the GCM authentication tag.
try (OutputStream out = SecureStreams.encryptedCompressed(path, key)) {
    payload.transferTo(out);
}
```

Compress *before* encryption because ciphertext is intentionally incompressible. Close only the outermost decorator; closure cascades and finalizes both GZIP trailers and the GCM authentication tag. In production, use a KMS/envelope-encrypted data key, bind the unencrypted header as GCM additional authenticated data, include a key/version identifier, and write to a temporary file followed by atomic rename so a crash cannot publish a partial object.

**Trade-offs and pitfalls.** Never reuse an AES-GCM IV with the same key. Do not log plaintext, keys, or sensitive payloads; log counts, correlation IDs, duration, and perhaps a safe digest. Streaming GCM authentication is verified at end-of-stream, so callers must fully read/close and treat earlier plaintext cautiously. Compression of attacker-controlled text adjacent to secrets can create length side channels. Buffer sizes affect throughput/memory, and decorator order must be documented in a versioned header.

**References:** [Oracle `FilterOutputStream` decorator API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/io/FilterOutputStream.html), [Oracle `CipherOutputStream`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/javax/crypto/CipherOutputStream.html), [NIST recommendation for GCM](https://csrc.nist.gov/pubs/sp/800/38/d/final).

## Part 2: Machine Coding and Practical Architectural Patterns

### 53. **Design a Concurrent Rate Limiter:** Code a fully functional backend rate limiter class implementing either the Token Bucket or Leaky Bucket algorithm. Ensure thread safety without global blocking.

**Interview-ready answer.** A token bucket has capacity `C`, refills at `r` tokens/second, and spends tokens per request. It permits bursts up to `C` while limiting the long-run rate to `r`. The following Java implementation keeps `(availableTokens,lastRefillTime)` in one immutable atomic state and updates it with compare-and-set (CAS), so no global monitor is taken. Each key/user should have its own bucket.

```java
import java.time.Duration;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicReference;

public final class TokenBucket {
    public record Decision(boolean allowed, Duration retryAfter,
                           double remainingTokens) {}
    private record State(double tokens, long refilledAtNanos) {}

    private final long capacity;
    private final double tokensPerSecond;
    private final AtomicReference<State> state;

    public TokenBucket(long capacity, double tokensPerSecond) {
        if (capacity <= 0 || !Double.isFinite(tokensPerSecond) || tokensPerSecond <= 0)
            throw new IllegalArgumentException("positive capacity and refill rate required");
        this.capacity = capacity;
        this.tokensPerSecond = tokensPerSecond;
        this.state = new AtomicReference<>(new State(capacity, System.nanoTime()));
    }

    public Decision tryAcquire() { return tryAcquire(1); }

    public Decision tryAcquire(long permits) {
        if (permits <= 0 || permits > capacity)
            throw new IllegalArgumentException("permits must be 1..capacity");

        for (;;) {
            State old = state.get();
            long now = System.nanoTime();       // monotonic; wall-clock changes irrelevant
            long elapsed = Math.max(0L, now - old.refilledAtNanos());
            double refilled = Math.min(capacity,
                    old.tokens() + elapsed * tokensPerSecond / 1_000_000_000d);

            if (refilled >= permits) {
                State next = new State(refilled - permits, now);
                if (state.compareAndSet(old, next))
                    return new Decision(true, Duration.ZERO, next.tokens());
            } else {
                double missing = permits - refilled;
                long waitNanos = (long) Math.ceil(missing / tokensPerSecond * 1_000_000_000d);
                State next = new State(refilled, now); // preserve fractional refill progress
                if (state.compareAndSet(old, next))
                    return new Decision(false, Duration.ofNanos(waitNanos), refilled);
            }
            // Another thread won; recompute from its newer state.
        }
    }
}

final class PerCustomerLimiters {
    private final ConcurrentHashMap<String, TokenBucket> buckets = new ConcurrentHashMap<>();

    TokenBucket.Decision allow(String customerId) {
        // Production code also evicts idle keys or uses a bounded cache.
        return buckets.computeIfAbsent(customerId, ignored -> new TokenBucket(100, 20.0))
                .tryAcquire();
    }
}
```

CAS makes each bucket lock-free at the state-variable level; unrelated customers never contend. Under extreme contention for one hot key, threads may retry CAS, which is still preferable to one application-wide lock. A fixed-point integer (for example microtokens) can replace `double` if deterministic rounding is required.

Return `429 Too Many Requests` with a rounded `Retry-After` for denials. Define the key carefully: authenticated tenant/user/API key, route/cost class, and possibly a separate IP abuse limit. Weighted endpoints can consume multiple permits. Metrics should include allowed/denied count, hot keys (with privacy-safe hashing), and CAS retry/latency.

**Trade-offs and pitfalls.** This class limits only one JVM. In a horizontally scaled service, use a gateway with sticky ownership/consistent hashing or an atomic Redis/Lua/database service and accept its availability trade-off. `ConcurrentHashMap` without eviction is a memory DoS risk. Client IP can be spoofed unless trusted proxy headers are validated. A token bucket allows bursts by design; use a sliding window/leaky bucket if burst smoothing matters. Rate limiting is not a substitute for concurrency limits, queue bounds, quotas, or DDoS protection.

**References:** [Oracle atomic package: lock-free thread-safe single-variable programming](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/atomic/package-summary.html), [Oracle `AtomicReference.compareAndSet`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/atomic/AtomicReference.html), [HTTP `429` status definition](https://www.rfc-editor.org/rfc/rfc6585.html#section-4).

### 54. **Design a Thread Pool:** Write a custom implementation of a blocking task queue and thread worker execution loops from scratch.

**Interview-ready answer.** A minimal pool needs a bounded queue (backpressure), worker lifecycle, task exception isolation, rejection after shutdown, and orderly termination. This implementation deliberately does not use `BlockingQueue` or `ExecutorService`; it builds a circular buffer with `ReentrantLock` and two condition variables.

```java
import java.time.Duration;
import java.util.*;
import java.util.concurrent.RejectedExecutionException;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.locks.*;

final class BoundedTaskQueue<T> {
    private final Object[] items;
    private int head, tail, size;
    private boolean closed;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notEmpty = lock.newCondition();
    private final Condition notFull = lock.newCondition();

    BoundedTaskQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException("capacity > 0");
        items = new Object[capacity];
    }

    void put(T item) throws InterruptedException {
        Objects.requireNonNull(item);
        lock.lockInterruptibly();
        try {
            while (size == items.length && !closed) notFull.await();
            if (closed) throw new RejectedExecutionException("queue closed");
            items[tail] = item;
            tail = (tail + 1) % items.length;
            size++;
            notEmpty.signal();
        } finally { lock.unlock(); }
    }

    @SuppressWarnings("unchecked")
    T take() throws InterruptedException {
        lock.lockInterruptibly();
        try {
            while (size == 0 && !closed) notEmpty.await();
            if (size == 0) return null; // closed and fully drained
            T item = (T) items[head];
            items[head] = null;         // do not retain completed task graph
            head = (head + 1) % items.length;
            size--;
            notFull.signal();
            return item;
        } finally { lock.unlock(); }
    }

    void closeForWrites() {
        lock.lock();
        try {
            closed = true;
            notEmpty.signalAll();
            notFull.signalAll();
        } finally { lock.unlock(); }
    }
}

public final class SimpleThreadPool implements AutoCloseable {
    private final BoundedTaskQueue<Runnable> queue;
    private final List<Thread> workers;
    private final AtomicBoolean accepting = new AtomicBoolean(true);

    public SimpleThreadPool(int workerCount, int queueCapacity) {
        if (workerCount <= 0) throw new IllegalArgumentException("workers > 0");
        queue = new BoundedTaskQueue<>(queueCapacity);
        var list = new ArrayList<Thread>(workerCount);
        for (int i = 0; i < workerCount; i++) {
            Thread t = new Thread(this::workerLoop, "simple-pool-" + i);
            t.setUncaughtExceptionHandler((thread, error) ->
                    System.getLogger("pool").log(System.Logger.Level.ERROR,
                            "worker died: " + thread.getName(), error));
            t.start();
            list.add(t);
        }
        workers = List.copyOf(list);
    }

    public void execute(Runnable task) throws InterruptedException {
        if (!accepting.get()) throw new RejectedExecutionException("pool shut down");
        queue.put(task); // bounded backpressure; interruptible
    }

    private void workerLoop() {
        try {
            for (;;) {
                Runnable task = queue.take();
                if (task == null) return;
                try { task.run(); }
                catch (Throwable failure) {
                    System.getLogger("pool").log(System.Logger.Level.ERROR,
                            "task failed", failure); // one bad task must not stop the loop
                }
            }
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
        }
    }

    /** Reject new tasks; workers finish queued tasks and then exit. */
    @Override public void close() {
        if (accepting.compareAndSet(true, false)) queue.closeForWrites();
    }

    public boolean awaitTermination(Duration timeout) throws InterruptedException {
        long deadline = System.nanoTime() + timeout.toNanos();
        for (Thread worker : workers) {
            long left = deadline - System.nanoTime();
            if (left <= 0) return false;
            worker.join(Math.max(1L, left / 1_000_000L));
        }
        return workers.stream().noneMatch(Thread::isAlive);
    }
}
```

Both `await` calls are in `while` loops because conditions can wake spuriously and other threads can consume the condition before reacquisition. Every lock is released in `finally`. Clearing the array slot avoids retaining large closures. The bounded queue creates deliberate backpressure; another valid policy is nonblocking rejection or caller-runs.

**Trade-offs and pitfalls.** This educational pool lacks task futures/cancellation, delayed tasks, dynamic sizing, worker replacement, metrics, context propagation, priority/fairness, and a carefully specified `shutdownNow`. Catching `Throwable` keeps this simple worker alive but a real runtime may choose to let fatal VM errors terminate it and replace the worker. Never submit tasks that wait for other tasks in the same saturated bounded pool—thread starvation deadlock can result. Production code should use the thoroughly tested `ThreadPoolExecutor` or virtual-thread executors and expose queue depth, active workers, rejection count, task wait time, and runtime.

**References:** [Oracle `Condition` contract and spurious wakeups](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/locks/Condition.html), [Oracle `ThreadPoolExecutor` reference and rejection policies](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html), [Oracle `ExecutorService` shutdown lifecycle](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ExecutorService.html).

### 55. **Design a Parking Lot System:** Provide class blueprints, database schemas, and clean code for a multi-story parking system prioritizing SOLID design.

**Interview-ready answer.** Separate policy from orchestration and persistence. The core use cases are spot allocation, ticket lifecycle, fee calculation, payment, and gate control. Spot compatibility, allocation order, and pricing change independently, so they are injected strategies rather than `switch` statements inside entities.

```java
enum VehicleType { MOTORCYCLE, CAR, TRUCK, ELECTRIC_CAR }
enum SpotType { MOTORCYCLE, COMPACT, LARGE, EV }
enum TicketStatus { ACTIVE, PAYMENT_PENDING, PAID, CLOSED, LOST }

record Vehicle(String plate, VehicleType type) {}
record Spot(long id, long levelId, String label, SpotType type) {}
record Ticket(UUID id, Vehicle vehicle, Spot spot,
              java.time.Instant enteredAt, TicketStatus status) {}
record Money(long minorUnits, java.util.Currency currency) {}

interface CompatibilityPolicy {
    List<SpotType> preferredSpotsFor(VehicleType type);
}
interface SpotRepository {
    /** Atomically claims one compatible available spot, or returns empty. */
    Optional<Spot> claim(long lotId, List<SpotType> preference);
    void release(long spotId);
}
interface TicketRepository {
    Ticket create(Vehicle vehicle, Spot spot, java.time.Instant enteredAt);
    Ticket lockActive(UUID ticketId);
    void markPaid(UUID ticketId, String paymentId);
    void close(UUID ticketId, java.time.Instant exitedAt);
}
interface PricingPolicy { Money price(Ticket ticket, java.time.Instant exitAt); }
interface PaymentPort { String createPayment(UUID ticketId, Money amount, String idempotencyKey); }
interface GatePort { void openEntry(String gateId); void openExit(String gateId); }

final class EntryService {
    private final CompatibilityPolicy compatibility;
    private final SpotRepository spots;
    private final TicketRepository tickets;

    // Transaction boundary belongs around claim + ticket creation.
    Ticket enter(long lotId, Vehicle vehicle, java.time.Instant now) {
        Spot spot = spots.claim(lotId, compatibility.preferredSpotsFor(vehicle.type()))
                .orElseThrow(() -> new IllegalStateException("LOT_FULL"));
        return tickets.create(vehicle, spot, now);
    }
}
```

The database is the concurrency authority. A PostgreSQL-oriented schema is:

```sql
CREATE TABLE parking_lot (
  id bigserial PRIMARY KEY, name text NOT NULL, timezone text NOT NULL
);
CREATE TABLE parking_level (
  id bigserial PRIMARY KEY, lot_id bigint NOT NULL REFERENCES parking_lot,
  level_no int NOT NULL, UNIQUE (lot_id, level_no)
);
CREATE TABLE parking_spot (
  id bigserial PRIMARY KEY, level_id bigint NOT NULL REFERENCES parking_level,
  label text NOT NULL, spot_type text NOT NULL,
  status text NOT NULL CHECK (status IN ('AVAILABLE','OCCUPIED','OUT_OF_SERVICE')),
  version bigint NOT NULL DEFAULT 0, UNIQUE (level_id, label)
);
CREATE INDEX available_spot_idx
  ON parking_spot(level_id, spot_type, id) WHERE status = 'AVAILABLE';

CREATE TABLE parking_ticket (
  id uuid PRIMARY KEY, lot_id bigint NOT NULL REFERENCES parking_lot,
  spot_id bigint NOT NULL REFERENCES parking_spot,
  plate text NOT NULL, vehicle_type text NOT NULL,
  entered_at timestamptz NOT NULL, exited_at timestamptz,
  status text NOT NULL, payment_id text
);
CREATE UNIQUE INDEX one_active_ticket_per_spot
  ON parking_ticket(spot_id) WHERE status IN ('ACTIVE','PAYMENT_PENDING','PAID');
CREATE UNIQUE INDEX one_active_ticket_per_vehicle
  ON parking_ticket(lot_id, plate) WHERE status IN ('ACTIVE','PAYMENT_PENDING','PAID');

CREATE TABLE parking_payment (
  id uuid PRIMARY KEY, ticket_id uuid NOT NULL REFERENCES parking_ticket,
  idempotency_key text NOT NULL UNIQUE, amount_minor bigint NOT NULL,
  currency char(3) NOT NULL, provider_ref text, status text NOT NULL
);
```

Claim a spot without serializing all entries:

```sql
WITH candidate AS (
  SELECT s.id
  FROM parking_spot s JOIN parking_level l ON l.id = s.level_id
  WHERE l.lot_id = :lotId AND s.status = 'AVAILABLE'
    AND s.spot_type = ANY(:orderedCompatibleTypes)
  ORDER BY array_position(:orderedCompatibleTypes, s.spot_type),
           l.level_no, s.id
  FOR UPDATE SKIP LOCKED
  LIMIT 1
)
UPDATE parking_spot s
SET status='OCCUPIED', version=version+1
FROM candidate c WHERE s.id=c.id
RETURNING s.*;
```

Run that update and ticket insert in one short transaction; rollback restores the spot. `SKIP LOCKED` lets concurrent gates select other spots. Display-board counts are derived/read-model data updated through an outbox and can be eventually consistent; the allocation query remains authoritative.

For exit, lock the active ticket, calculate a versioned price, and create an idempotent payment attempt—but never hold a database row lock while calling a payment provider. After an authenticated payment confirmation, a short transaction marks payment/ticket paid, releases the spot, closes the ticket, and writes an outbox event; only then open the gate. Repeated confirmations see the already-closed ticket and safely return the same result.

**Trade-offs and pitfalls.** `SKIP LOCKED` improves throughput but does not guarantee perfect fairness. EV charging, reservations, accessible spots, monthly passes, lost tickets, overnight tariff boundaries, grace periods, and timezone/DST rules belong in explicit policies. Keep money in minor units and store the pricing-rule version used. Gate hardware commands need retries and command IDs; “DB committed” does not prove the physical barrier moved. Partial unique indexes and transactions—not an in-memory counter—prevent double assignment across service instances.

**References:** [PostgreSQL explicit row locking](https://www.postgresql.org/docs/current/explicit-locking.html), [PostgreSQL `SKIP LOCKED` semantics](https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE), [Spring transaction management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html).

### 56. **Design a Movie Ticket Booking Engine (e.g., BookMyShow):** Focus heavily on the exact class structures and database lock orchestration preventing multiple users from booking the identical seat simultaneously.

**Interview-ready answer.** Treat a seat in a particular show as inventory. A physical `seat` is static; `show_seat(show_id,seat_id)` is the mutable sellable unit. The database—not Redis or the React UI—must decide the winner. Use short holds, deterministic row locks, conditional state transitions, unique constraints, and idempotent payment/confirmation.

```java
enum SeatState { AVAILABLE, HELD, BOOKED, BLOCKED }
enum BookingState { HELD, PAYMENT_PENDING, CONFIRMED, EXPIRED, CANCELLED }

record SeatId(long value) {}
record Hold(UUID bookingId, UUID token, long showId,
            List<SeatId> seats, java.time.Instant expiresAt, long amountMinor) {}

interface ShowSeatRepository {
    List<ShowSeat> lockInOrder(long showId, List<SeatId> sortedSeatIds);
    void hold(long showId, List<SeatId> seats, UUID token, java.time.Instant expiresAt);
    void markBooked(long showId, List<SeatId> seats, UUID token, UUID bookingId);
}
interface BookingRepository {
    Booking lock(UUID bookingId);
    void createHeld(Hold hold, UUID userId, String priceVersion);
    void confirm(UUID bookingId, String paymentReference);
}

final class BookingService {
    private final ShowSeatRepository inventory;
    private final BookingRepository bookings;
    private final PricingService pricing;
    private final java.time.Clock clock;

    // @Transactional: DB transaction; no remote calls inside.
    Hold hold(UUID userId, long showId, List<SeatId> requested) {
        List<SeatId> seats = requested.stream().distinct()
                .sorted(java.util.Comparator.comparingLong(SeatId::value)).toList();
        if (seats.isEmpty() || seats.size() != requested.size())
            throw new IllegalArgumentException("invalid/duplicate seats");

        var rows = inventory.lockInOrder(showId, seats);
        var now = clock.instant();
        if (rows.size() != seats.size() || rows.stream().anyMatch(s ->
                s.state() == SeatState.BOOKED || s.state() == SeatState.BLOCKED ||
                (s.state() == SeatState.HELD && s.holdExpiresAt().isAfter(now))))
            throw new SeatsUnavailableException();

        UUID token = UUID.randomUUID();
        var expires = now.plusSeconds(5 * 60);
        var quote = pricing.quote(showId, seats, now);
        Hold hold = new Hold(UUID.randomUUID(), token, showId, seats,
                expires, quote.amountMinor());
        inventory.hold(showId, seats, token, expires);
        bookings.createHeld(hold, userId, quote.ruleVersion());
        return hold;
    }
}
```

Core schema and constraints:

```sql
CREATE TABLE movie_show (
  id bigserial PRIMARY KEY, screen_id bigint NOT NULL,
  movie_id bigint NOT NULL, starts_at timestamptz NOT NULL,
  status text NOT NULL
);
CREATE TABLE seat (
  id bigserial PRIMARY KEY, screen_id bigint NOT NULL,
  row_label text NOT NULL, seat_no int NOT NULL, category text NOT NULL,
  UNIQUE(screen_id, row_label, seat_no)
);
CREATE TABLE show_seat (
  show_id bigint NOT NULL REFERENCES movie_show,
  seat_id bigint NOT NULL REFERENCES seat,
  price_minor bigint NOT NULL, currency char(3) NOT NULL,
  state text NOT NULL CHECK (state IN ('AVAILABLE','HELD','BOOKED','BLOCKED')),
  hold_token uuid, hold_expires_at timestamptz, booking_id uuid,
  version bigint NOT NULL DEFAULT 0,
  PRIMARY KEY(show_id, seat_id),
  CHECK ((state='HELD') = (hold_token IS NOT NULL AND hold_expires_at IS NOT NULL)),
  CHECK (state <> 'BOOKED' OR booking_id IS NOT NULL)
);
CREATE TABLE booking (
  id uuid PRIMARY KEY, user_id uuid NOT NULL, show_id bigint NOT NULL,
  hold_token uuid NOT NULL UNIQUE, state text NOT NULL,
  expires_at timestamptz NOT NULL, amount_minor bigint NOT NULL,
  currency char(3) NOT NULL, price_version text NOT NULL,
  payment_reference text UNIQUE, created_at timestamptz NOT NULL
);
CREATE TABLE booking_item (
  booking_id uuid NOT NULL REFERENCES booking,
  show_id bigint NOT NULL, seat_id bigint NOT NULL,
  price_minor bigint NOT NULL,
  PRIMARY KEY(booking_id, seat_id),
  UNIQUE(show_id, seat_id) -- at most one booking record owns that inventory
);
CREATE TABLE payment_attempt (
  id uuid PRIMARY KEY, booking_id uuid NOT NULL REFERENCES booking,
  idempotency_key text NOT NULL UNIQUE, provider_ref text UNIQUE,
  state text NOT NULL, amount_minor bigint NOT NULL
);
```

The lock query must order every contender identically to reduce deadlocks:

```sql
SELECT * FROM show_seat
WHERE show_id=:showId AND seat_id = ANY(:seatIds)
ORDER BY seat_id
FOR UPDATE;

-- After verifying all rows in the same transaction:
UPDATE show_seat
SET state='HELD', hold_token=:token, hold_expires_at=:expires, version=version+1
WHERE show_id=:showId AND seat_id = ANY(:seatIds)
  AND (state='AVAILABLE' OR (state='HELD' AND hold_expires_at <= now()));
-- Verify affected row count == requested count; otherwise roll back everything.
```

Transaction A locks the seat rows. Transaction B requesting an overlapping seat waits; after A commits, B sees `HELD` and fails. There is no check-then-write gap. As an alternative, a single conditional `UPDATE ... RETURNING` plus an exact affected-count check is valid in PostgreSQL, but deterministic pre-locking is easier to reason about for multi-seat deadlocks.

**Payment/confirmation flow.** Commit the hold before calling the PSP. Create/reuse `payment_attempt` by idempotency key and call the PSP outside the seat transaction. On signed webhook or verified status callback, start a new transaction: lock `booking`, then its `show_seat` rows in seat order; require matching `hold_token`, unexpired hold, and confirmed amount; set seats `BOOKED` and booking `CONFIRMED`; insert an outbox `BookingConfirmed` event; commit. Duplicate webhook delivery returns the existing confirmed result. If payment succeeds after expiry, never silently steal a seat now held/booked by somebody else—void/refund it or attempt a clearly defined re-hold workflow.

Expiration is both lazy (an allocator treats an expired `HELD` row as reclaimable using database `now()`) and proactive (a worker marks expired bookings/seats available). Redis may cache availability and host timers, but is never the final allocation authority. Partition Kafka events by `bookingId`/`showId`, use authenticated webhooks, and reconcile PSP state against local `PAYMENT_PENDING` records.

**Trade-offs and pitfalls.** Pessimistic row locking is straightforward but hot premiere shows create contention; shard/route by `show_id`, keep transactions tiny, cap seats per booking, and fail fast with lock timeouts. Optimistic `version` updates work for single seats but multi-seat all-or-nothing retries are more complex. Do not hold locks during user think time or PSP calls. A unique constraint is the last line of defense, not a replacement for a good state machine. Price, convenience fee, tax, cancellation, and hold duration must be versioned and auditable.

**References:** [PostgreSQL row-level lock behavior](https://www.postgresql.org/docs/current/explicit-locking.html#LOCKING-ROWS), [PostgreSQL transaction isolation and predicate re-evaluation](https://www.postgresql.org/docs/current/transaction-iso.html), [Spring transaction management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html).

## Part 2: High-Level Design and Distributed Systems

### 57. Explain the CAP Theorem. If a network partition occurs, how do you mathematically choose between Consistency and Availability in a banking app vs. a social media feed?

**Interview-ready answer.** In the formal CAP result, **C** is atomic consistency/linearizability (not the “C” in ACID), **A** means every request received by a non-failing node eventually gets a response, and **P** means the model tolerates messages being arbitrarily lost/delayed between groups of nodes. During an actual partition, a replicated read/write object cannot guarantee both linearizability and that availability definition.

The proof intuition is mathematical. Suppose replicas `A` and `B` cannot communicate. A write `x=1` completes at `A`, then a client invokes a read at `B` after that completion. `B` cannot distinguish “the write happened” from “it did not happen.” If `B` returns the old value, it violates real-time linearizability; if it waits/rejects until it can learn the result, it gives up availability. No algorithm can obtain the missing information.

There is no universal numeric formula that labels an entire business “CP” or “AP.” Choose **per operation and invariant**:

- For a bank balance debit/transfer, the invariant `availableBalance >= 0` and single posting of a transfer normally dominate. Route a given account to a leader or require a write quorum/consensus; the minority side returns an explicit unavailable/retryable result during partition. Use idempotency keys and an append-only ledger. This is a CP choice for that mutation. Banks may still offer AP operations such as reading a marked-stale statement or accepting an offline authorization under a bounded risk limit—the product decides the financial exposure.
- A social feed can accept posts/likes in both regions, attach globally unique IDs and causal/logical timestamps, and merge/reorder later. Users may temporarily see different feeds, so the feed path favors availability and eventual convergence. Security policy, username uniqueness, or payment inside the same company may still be CP.

For `N` replicas, quorum designs often choose write acknowledgements `W` and read responses `R` such that `R + W > N` (read/write sets intersect), and often `W > N/2` to prevent disjoint successful writes. For `N=3`, `W=2,R=2` rejects a side with only one replica during a 2–1 split and therefore sacrifices availability there. Intersection alone is not a complete linearizability proof: the protocol still needs version/order rules, correct failure detection, and read repair/leader semantics. “Sloppy” quorums may favor availability and weaken the guarantee.

**Trade-offs and pitfalls.** “Pick any two of three” is misleading: when no partition exists, systems can provide C and A; when a partition exists, P is the failure condition, and the real choice is which requests sacrifice C or A. Timeouts make a slow link indistinguishable from a partition, so practical systems make a latency-bounded choice. CAP says nothing about normal-case latency, durability, transaction isolation, or conflict semantics—that motivates PACELC and additional models.

**References:** [Gilbert and Lynch's formal CAP paper](https://www.cs.princeton.edu/courses/archive/spr22/cos418/papers/cap.pdf), [Amazon Dynamo paper: an availability-oriented production design](https://www.amazon.science/publications/dynamo-amazons-highly-available-key-value-store).

### 58. Explain PACELC theorem. How does it expand on CAP when a distributed network is running normally without partitions?

**Interview-ready answer.** PACELC is a design formulation: **if there is a Partition (P), choose between Availability (A) and Consistency (C); Else (E), under normal operation, choose between lower Latency (L) and Consistency (C).**

```text
          partition? 
          /         \
       yes           no (“else”)
      A vs C          L vs C
```

CAP discusses an impossibility during a particular failure. PACELC highlights the trade-off paid every day. A globally linearizable write may need a leader round trip or quorum across regions before acknowledging. Waiting makes replicas agree on the operation's real-time order, but adds at least network propagation and tail latency. A local write/read can return in a few milliseconds but another region may temporarily have an older value.

Examples by *operation/configuration*, rather than immutable product labels:

- A multi-region ledger may be **PC/EC**: reject minority writes during a partition, and in the else case pay cross-region coordination latency for strong consistency.
- A geo-replicated feed/cache may be **PA/EL**: accept local operations during partitions and, normally, return a local replica immediately while replication catches up.
- A tunable system can use a local consistency level for feed reads and quorum/serial consistency for username claims. Session guarantees such as read-your-writes, monotonic reads, causal consistency, or bounded staleness occupy useful points between the extremes.

Suppose client-to-local-region latency is 10 ms and inter-region round-trip time is 120 ms. A local eventually consistent read might complete around the local cost, while a linearizable quorum/leader read that must cross the ocean has a lower bound influenced by that 120 ms path. The exact latency is deployment/protocol dependent; PACELC is a reasoning framework, not a formula claiming consistency always costs the same number of milliseconds.

**Trade-offs and pitfalls.** “Else” does not mean there are no faults—replica lag, node failures, and congestion still occur. Low latency versus consistency is not binary, and caching, leases, TrueTime-style bounded clocks, locality-aware leaders, and follower reads with staleness bounds change the curve. Classifying a database without naming consistency level, topology, and operation is usually wrong. Measure p95/p99 latency and define the staleness/invariant users can tolerate.

**References:** [Daniel Abadi's original PACELC paper](https://www.cs.umd.edu/~abadi/papers/abadi-pacelc.pdf), [Google Spanner paper: globally distributed consistency and latency engineering](https://www.usenix.org/system/files/conference/osdi12/osdi12-final-16.pdf).

### 59. How does Consistent Hashing work in horizontal scaling load balancers and distributed databases? Walk through the hash ring mechanics and virtual nodes.

**Interview-ready answer.** Consistent hashing maps both nodes and keys into the same circular hash space, commonly `[0, 2^m)`. A key belongs to the first node token encountered clockwise from `hash(key)`, wrapping to the first token. With ordinary `hash(key) % n`, changing `n` remaps most keys; on a well-balanced consistent ring, adding/removing one of `n` equal nodes moves roughly `1/n` of keys.

```java
final class HashRing<N> {
    private final java.util.NavigableMap<Long, N> ring = new java.util.TreeMap<>();
    private final int virtualNodes;
    HashRing(int virtualNodes) { this.virtualNodes = virtualNodes; }

    void add(String stableNodeId, N node) {
        for (int replica = 0; replica < virtualNodes; replica++)
            ring.put(hash64(stableNodeId + "#" + replica), node);
    }
    void remove(String stableNodeId) {
        for (int replica = 0; replica < virtualNodes; replica++)
            ring.remove(hash64(stableNodeId + "#" + replica));
    }
    N owner(String key) {
        if (ring.isEmpty()) throw new IllegalStateException("empty ring");
        var entry = ring.ceilingEntry(hash64(key));
        return (entry != null ? entry : ring.firstEntry()).getValue();
    }
    private static long hash64(String value) {
        // Illustration only: use a stable, well-distributed 64/128-bit hash in production.
        byte[] d;
        try { d = java.security.MessageDigest.getInstance("SHA-256")
                .digest(value.getBytes(java.nio.charset.StandardCharsets.UTF_8)); }
        catch (java.security.GeneralSecurityException e) { throw new AssertionError(e); }
        return java.nio.ByteBuffer.wrap(d).getLong() & Long.MAX_VALUE;
    }
}
```

If tokens are `A=10`, `B=40`, `C=80`, key hash 25 maps to B. Adding D at 30 moves only keys in `(10,30]` from B to D. Removing B moves B's `(30,40]` range to the next clockwise owner C.

One token per physical node has high statistical variance. **Virtual nodes (vnodes)** place each physical node at many independently hashed tokens. Its many small ranges are interleaved with other nodes, improving balance, spreading a failed node's load across many survivors, and allowing a larger server to receive more tokens. A replicated database commonly stores a key on its owner plus the next `RF-1` distinct physical nodes clockwise, while placement logic must respect rack/zone failure domains.

Load balancers use the same idea for cache affinity/session routing; clients/proxies need a consistent membership view. Databases additionally stream reassigned ranges during topology changes, track replicas, and repair divergence—the ring only chooses placement.

**Trade-offs and pitfalls.** Consistent hashing balances hash ranges, not hot keys: one celebrity key can still overload its owners and needs replication, caching, request coalescing, or key splitting. Vnodes consume metadata and increase rebalancing streams. Membership changes must be versioned; inconsistent rings route requests incorrectly. Range scans are awkward with random hashing. Rendezvous and jump consistent hashing are useful alternatives. Capacity weights, zones, draining, replication, and bounded-load variants are production requirements beyond the basic diagram.

**References:** [Karger et al.'s original consistent hashing paper](https://people.csail.mit.edu/karger/Papers/web.pdf), [Amazon Science: Dynamo publication and paper](https://www.amazon.science/publications/dynamo-amazons-highly-available-key-value-store).

### 60. Design a multi-tier caching architecture. Explain Cache-Aside, Write-Through, Write-Behind, and Refresh-Ahead patterns. When does each fail?

**Interview-ready answer.** A practical read path is browser/private cache → CDN/edge (public HTTP objects) → per-process bounded L1 → distributed Redis/Memcached L2 → authoritative database/service. Each tier has a shorter latency but a harder invalidation/freshness story. Keys must include tenant, authorization-relevant variation, representation/schema version, and entity/query version; never let a shared tier cache personalized data under a public key.

```text
request -> CDN -> service L1 -> Redis L2 -> system of record
                         ^          ^
                     short TTL   shared TTL + invalidation events
```

**Cache-aside (lazy loading):** application reads cache; on miss reads the database and fills cache. On update, commit the database then invalidate the cache.

```text
value = cache.get(k)
if absent: value = db.read(k); cache.set(k, value, ttl+jitter)
return value
```

It is simple and caches only demanded data. It fails via cold-miss latency, stale TTLs, stampedes, and the race where a slow reader fetches old data, a writer commits/deletes the key, then the reader repopulates the old value. Use versioned values/keys, single-flight loading, invalidation events, and a TTL backstop.

**Write-through:** the controlled write path updates the system of record and cache before reporting completion (or uses an outbox/repair workflow because two independent systems are not one atomic transaction). It improves read-after-write freshness for hot data, but adds write latency, caches never-read values, and can leave divergence if one write succeeds and the other fails. The database remains authority.

**Write-behind (write-back):** acknowledge a cache write, queue/coalesce it, and persist to the database asynchronously. It offers low write latency and batching but can lose acknowledged writes when the cache/queue fails, reorder updates, violate constraints, and make recovery/reconciliation difficult. Use a durable log and idempotent versioned sink if data matters; avoid it for ledger balances. It suits reconstructible counters/telemetry or deliberately asynchronous systems.

**Refresh-ahead:** before a hot entry expires, one worker recomputes it so readers continue getting a fresh or stale-while-revalidate value. It removes cliff-edge misses for predictable hot keys but wastes origin work on entries no longer requested, can amplify refresh failures, and still serves stale data under slow refresh. Use leases/single-flight, jitter, refresh budgets, and stale limits.

**Trade-offs and pitfalls.** L1 makes event invalidation fan-out harder; keep its TTL short and include versions. CDN purges can take time, so immutable asset names/content hashes are safer. Negative caching protects the origin but must expire quickly. Define behavior when Redis is down: bounded direct-origin fallback, stale serving, or fail closed according to data criticality. Track hit ratio by tier, fill/refresh latency, evictions, stale age, origin amplification, and invalidation lag—not hit ratio alone.

**References:** [Microsoft Azure Cache-Aside pattern and consistency caveats](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside), [AWS Redis caching patterns](https://docs.aws.amazon.com/whitepapers/latest/database-caching-strategies-using-redis/caching-patterns.html), [Microsoft caching guidance including write-through](https://learn.microsoft.com/en-us/azure/architecture/best-practices/caching).

### 61. How do you handle cache invalidation at scale? Explain the thundering herd problem, cache avalanche, and cache penetration, and detail structural fixes for each.

**Interview-ready answer.** Make invalidation part of the write architecture, not an ad-hoc `DEL`. In the same transaction as the authoritative mutation, write an outbox record containing `{entityId,newVersion,eventId}`. CDC/relay publishes it; every cache layer invalidates the old key or accepts only a value whose version is at least `newVersion`. A TTL with randomized jitter is the repair boundary if an event is lost. Consumers and invalidations are idempotent.

Versioned keys (`product:{id}:v{version}`) prevent a late, stale cache fill from overwriting a newer version. For query/list caches, maintain dependency tags or bump a namespace/generation (`catalog-search:g42:...`) instead of enumerating millions of keys. Do not delete broad wildcards synchronously in the request path.

The three failure modes differ:

- **Thundering herd/cache stampede:** many requests for the *same hot key* miss/expire together and all query the origin. Use per-key single-flight/request coalescing; a short distributed lease with token-safe unlock; stale-while-revalidate; probabilistic early refresh; TTL jitter; bounded waiters/backpressure. The lease winner fills; others wait briefly or receive explicitly bounded stale data. Always cap lease duration and tolerate the loader dying.
- **Cache avalanche:** many *different keys* expire together, or an entire cache tier fails, suddenly shifting a large fraction of traffic to the origin. Add TTL jitter, stagger warm-up, avoid one shared expiration timestamp, run cache replicas/cluster with tested failover, keep origin capacity/headroom, use circuit breakers and load shedding, and degrade optional features. Prewarm only genuinely hot keys at a controlled rate.
- **Cache penetration:** repeated requests for keys that do not exist bypass the cache and hit the database—often abuse or malformed IDs. Validate format/authorization early, rate-limit, cache `NOT_FOUND` for a short jittered TTL, and optionally place a Bloom filter before the database. A Bloom negative means definitely absent; a positive means “possibly present” and still requires the authoritative lookup. Keep the filter updated/rebuilt and size it for an acceptable false-positive rate.

```text
DB transaction: update product + insert outbox(version=18)
       -> CDC event product/7/v18
       -> Redis/L1/CDN consumers discard any version < 18
read miss -> one loader reads DB v18 -> conditional cache set(version >= existing)
```

**Trade-offs and pitfalls.** A distributed lock alone does not make cache contents correct, and unsafe lock deletion can release another owner's lease; use a unique token and atomic compare-delete. Pub/Sub invalidation without retention can lose events while a node is offline, so combine a durable stream/version check with TTL. Long negative TTLs hide newly created objects. Serving stale authorization, account balance, inventory, or revocation data can be unsafe—these paths need stricter freshness or no cache. Monitor misses by reason, origin QPS amplification, key cardinality, stale age, lease contention, invalidation lag, and cache-node saturation.

**References:** [Azure cache consistency/concurrency guidance](https://learn.microsoft.com/en-us/azure/architecture/best-practices/caching), [Redis distributed-lock correctness guidance](https://redis.io/docs/latest/develop/use/patterns/distributed-locks/), [Redis Bloom filter semantics](https://redis.io/docs/latest/develop/data-types/probabilistic/bloom-filter/).

### 62. Deep dive into Apache Kafka Architecture. How do partitions, consumer groups, offsets, and broker replication factors guarantee both high performance and zero data loss?

**Interview-ready answer.** Kafka's architecture can provide high throughput and strong durability *within a defined failure model*, but the wording “guarantee ... zero data loss” is too absolute. Partitions, groups, offsets, and replication each solve a different problem; correct configuration and application behavior are all required, and correlated failures, ignored errors, retention, operator deletion, or lost external side effects can still lose data.

**Storage and performance.** A topic is split into partitions. Each partition is an ordered, append-only log; an offset is that partition's monotonically increasing logical position, not a global event ID. A record key normally chooses a partition, giving order only for records in that partition. Producers batch and compress records; brokers use sequential append, page cache, and large network transfers. Many partitions distribute I/O and leadership across brokers, allowing producers and consumers to work in parallel. Too many partitions increase metadata, open files, recovery, and rebalance costs.

**Replication.** One replica is leader; followers fetch its log. `replication.factor=3` means three intended copies, while the **in-sync replica (ISR)** set represents replicas currently caught up enough to be eligible under the replication protocol. A leader failure promotes an eligible replica. Replication factor alone says neither how many replicas acknowledged the current record nor whether all three are healthy.

**Consumption.** In a classic consumer group, the coordinator assigns each partition to at most one group member at a time, so a group scales processing up to its partition count while preserving per-partition order. A different group has independent assignments and can read the same records. Consumers pull batches and track a current position; committed offsets in Kafka's internal offsets topic are recovery checkpoints. During rebalances, partitions move and consumers resume from committed offsets.

Offsets do not prove a business side effect happened. Commit before processing and a crash can skip work (**at-most-once**). Process first and commit afterward and a crash between the two causes replay (**at-least-once**), so the consumer must be idempotent. Kafka transactions can atomically couple consumed offsets with Kafka output records for Kafka-to-Kafka processing; they do not automatically include an arbitrary database or HTTP call.

For a strong single-cluster durability baseline:

```properties
# topic/broker intent
replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false

# producer
acks=all
enable.idempotence=true
retries=2147483647
delivery.timeout.ms=120000
```

Also place replicas across racks/zones, use durable volumes, handle every send future/error, give each event a business/event ID, monitor ISR shrink/offline or under-replicated partitions, secure destructive administration, test restore, and define cross-region backup/mirroring and an RPO. With `acks=all`, a write succeeds only after the current ISR acknowledges and the ISR has at least `min.insync.replicas`; if not, the producer gets an error instead of a false success. Idempotence uses producer identity/sequence numbers to deduplicate protocol retries and preserve ordering under the allowed configuration.

**Why this is not magical zero loss.** `acks=0` or `acks=1` can lose an acknowledged-to-the-app record under ordinary broker failure scenarios. Even with the stronger settings, losing all replicas that contain an acknowledged record, correlated storage corruption, an unsafe leader policy, credential misuse/topic deletion, retention before a lagging consumer reads, client code swallowing a send failure, or an asynchronous disaster-recovery gap can lose it. Kafka often relies on replication rather than fsyncing every message on every acknowledgement, so the exact storage/power failure envelope matters. “Zero loss” should be replaced by a tested statement such as: “No acknowledged record is lost after any one broker/AZ failure; producer errors are surfaced; cross-region RPO is 5 minutes.”

**Trade-offs and pitfalls.** More partitions improve parallelism but weaken global ordering and raise overhead. `acks=all` plus a larger `min.insync.replicas` reduces availability during failures in exchange for durability. Consumer lag can exceed retention. Log compaction keeps the latest value per key, not an immutable history of every update. Monitor end-to-end reconciliation counts rather than assuming a green broker dashboard proves business completion.

**References:** [Apache Kafka 4.3 introduction: topics, partitions, and replication](https://kafka.apache.org/43/getting-started/introduction/), [Apache Kafka design: delivery semantics and replication](https://kafka.apache.org/43/design/design/), [Apache Kafka topic configuration including `min.insync.replicas`](https://kafka.apache.org/43/configuration/topic-configs/).

### 63. What is the difference between Kafka and RabbitMQ regarding message ingestion? Compare log-centric pull architectures against smart-broker push routing.

**Interview-ready answer.** First correct the shorthand: in both systems a producer normally pushes/publishes data **to the broker**. The contrast is mainly downstream consumption and storage/routing. Kafka is a retained partitioned log whose consumers pull by offset. Traditional RabbitMQ queues are broker-managed work queues: exchanges perform rich routing into queues and registered consumers normally receive pushed deliveries under a prefetch/ack window. RabbitMQ also offers Streams, and AMQP has a polling `basic.get`, so this is an architectural default, not a law.

| Dimension | Kafka | RabbitMQ (classic/quorum queues) |
|---|---|---|
| Primary abstraction | Topic-partition append log | Exchange → bindings → queue → delivery |
| Routing | Topic plus key/partitioner; consumers filter/process | Direct, topic, fanout, headers exchanges; routing keys and multiple queues |
| Consumption | Consumer polls batches; controls offset/pace | Broker pushes to subscribed consumer; prefetch limits unacknowledged deliveries |
| Message lifetime | Retained by time/size or compaction independent of reads; replay is normal | Normally removed after acknowledgement; queue represents pending work |
| Scaling unit | Partitions; one active classic-group consumer per partition | Queues/consumers; quorum queues replicate with a leader, and sharding requires topology choices |
| Position/state | Consumer-group committed offsets; independent groups replay | Broker tracks ready/unacknowledged deliveries and requeues unacked work |
| Ordering | Guaranteed only within a partition | Queue order is affected by priorities, redelivery, and concurrent consumers |
| Typical strength | High-throughput event streams, replay, analytics/CDC, event history | Flexible command/task routing, request/reply, low-latency work queues, per-message TTL/DLX/priority |

Kafka's pull model lets a slow consumer fall behind without the broker maintaining a distinct copy per subscriber; batching sequential offsets makes catch-up efficient. Backpressure is natural—poll less/limit processing—but consumers must stay within liveness settings and ensure poll/processing/offset commits are coordinated. More independent use cases create consumer groups, not destructive competing reads.

RabbitMQ's “smart broker” evaluates exchange bindings and pushes eligible messages. Manual acknowledgements transfer responsibility only after processing; if a connection closes first, unacked deliveries are requeued. `basic.qos` prefetch bounds in-flight messages, preventing an eager broker/client pipeline from exhausting consumer memory. Publisher confirms and durable/quorum queues are required for stronger safety; persistent flags alone are not an end-to-end guarantee.

**Trade-offs and pitfalls.** Neither product is universally faster. Kafka's sequential batches dominate sustained stream throughput and replay, but a partition is a coarse scaling/order boundary. RabbitMQ's richer routing and per-message behavior consume broker CPU/memory and a single hot queue can bottleneck, while it can be operationally natural for task dispatch. Both can deliver duplicates, so consumers need idempotency. RabbitMQ acknowledgements/publisher confirms do not create generic exactly-once effects; Kafka EOS has a defined transactional scope. Choose using replay horizon, routing semantics, ordering key, fan-out model, per-message features, throughput/latency, failure recovery, and operator expertise—not “push vs pull” alone.

**References:** [Apache Kafka design: consumer pull model](https://kafka.apache.org/43/design/design/#theconsumer), [RabbitMQ consumers and pushed subscriptions](https://www.rabbitmq.com/docs/consumers), [RabbitMQ acknowledgements, publisher confirms, and prefetch](https://www.rabbitmq.com/docs/confirms).

### 64. How do you guarantee exactly-once processing semantics across a complex, distributed microservice architecture utilizing Kafka topics?

**Interview-ready answer.** You do not obtain a universal exactly-once guarantee by enabling one Kafka flag. Define the desired **exactly-once business effect**, then make every boundary atomic or idempotent. Kafka can provide atomic read → process → write semantics when inputs, outputs, offsets, and Kafka Streams state are inside Kafka's transaction domain. An external SQL transaction, payment API, email, or another cluster requires cooperation such as an inbox/outbox and idempotency.

**Kafka-to-Kafka path.** An idempotent producer gives each producer session an ID/epoch and per-partition sequence numbers, so broker retries do not append duplicates. A transactional producer additionally makes output records across partitions and consumed offsets one atomic commit and fences an old (“zombie”) producer using `transactional.id`.

```properties
# producer: unique per concurrently running instance, stable enough to fence restarts
enable.idempotence=true
transactional.id=order-enricher-${INSTANCE_ORDINAL}
acks=all

# consumer
enable.auto.commit=false
isolation.level=read_committed
```

```java
producer.initTransactions();
while (running) {
    var records = consumer.poll(java.time.Duration.ofMillis(500));
    try {
        producer.beginTransaction();
        for (var r : records) {
            var output = transform(r); // deterministic/pure or transaction-safe
            producer.send(new ProducerRecord<>("orders.enriched", r.key(), output));
        }
        producer.sendOffsetsToTransaction(
                nextOffsets(records), consumer.groupMetadata()); // offset = last + 1
        producer.commitTransaction();
    } catch (org.apache.kafka.common.KafkaException failure) {
        producer.abortTransaction();
        // retry/recreate producer for fatal fencing errors; do not commit consumer offsets
    }
}
```

`read_committed` consumers do not expose aborted output. If the process dies before commit, neither outputs nor offsets become committed and input is retried. Kafka Streams supplies this machinery with `processing.guarantee=exactly_once_v2`, atomically coordinating input offsets, output topics, changelogs, and supported state stores.

**Database boundary: transactional inbox/outbox.** Give every event a globally unique immutable `event_id`, plus aggregate key, schema version, correlation/causation IDs. A consumer applies an event and records its ID in the *same database transaction*:

```sql
CREATE TABLE processed_event (
  consumer_name text NOT NULL,
  event_id uuid NOT NULL,
  processed_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (consumer_name, event_id)
);
CREATE TABLE outbox_event (
  event_id uuid PRIMARY KEY,
  aggregate_type text NOT NULL, aggregate_id text NOT NULL,
  event_type text NOT NULL, payload jsonb NOT NULL,
  occurred_at timestamptz NOT NULL DEFAULT now()
);

BEGIN;
INSERT INTO processed_event(consumer_name,event_id)
VALUES ('inventory', :eventId)
ON CONFLICT DO NOTHING;
-- Only if one row was inserted: update inventory and INSERT outgoing outbox event.
COMMIT;
-- Then commit Kafka offset. A crash before this commit causes replay; the PK makes it a no-op.
```

The producing service updates its business row and inserts `outbox_event` in one local transaction. Debezium/CDC or a polling relay publishes the outbox to Kafka. The relay can publish twice if it crashes at an ambiguous point, which is why downstream inbox deduplication remains required. Key events by aggregate ID to preserve that aggregate's partition order.

For a payment/email/HTTP side effect, propagate `event_id` as a provider idempotency key and persist the provider request/result before acknowledging. If the external API neither supports idempotency nor participates in an atomic transaction and its result cannot be queried, exactly-once effect is impossible under crash ambiguity: after timeout, the caller cannot know whether retrying duplicates the action. Use reconciliation, a provider status lookup, or accept/document at-least-once/at-most-once behavior.

Across many microservices, repeat the local pattern at every hop and model the workflow as an idempotent saga/state machine with legal versioned transitions and compensations. A global transaction across all topics and databases is usually neither available nor scalable. Add retry topics/DLQs for poison events, but preserve event identity; monitor transaction aborts/fencing, consumer lag, duplicate-inbox hits, outbox age, and source-to-sink reconciliation totals.

**Trade-offs and pitfalls.** Kafka's producer idempotence only deduplicates protocol retries; sending the same business command twice with two `send()` calls still creates two records. Transactional IDs reused concurrently cause fencing; never derive them randomly if restart fencing is required. Long transactions delay visibility and can time out. “Chained” Kafka+database transaction managers still have a crash window unless an actual shared atomic coordinator exists. Exactly-once costs latency, broker state, storage for dedupe IDs, and operational complexity—apply it to effects that require it, and prefer explicit idempotency everywhere.

**References:** [Apache Kafka design: message delivery semantics and transactions](https://kafka.apache.org/43/design/design/#semantics), [Apache Kafka Streams processing guarantees](https://kafka.apache.org/43/streams/core-concepts/#streams_processing_guarantee), [Debezium Outbox Event Router documentation](https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html).

---

> These answers are written for interview preparation. Product/version behavior is called out where it matters, and the links under **References** point to primary specifications or official vendor documentation checked in August 2026.

## Part 2: Databases, Transactions, and Distributed Operations

### 65. Explain Database Isolation Levels (Read Uncommitted, Read Committed, Repeatable Read, Serializable). What anomalies (Dirty Read, Non-repeatable Read, Phantom Read) occur in each?

**Interview-ready summary.** Isolation controls how much one concurrent transaction can observe another transaction's work. The SQL standard describes four levels by the anomalies they must prevent. Higher isolation gives stronger reasoning guarantees, but may add blocking, bookkeeping, or transaction aborts that the application must retry.

| Level | Dirty read | Non-repeatable read | Phantom read | Typical model |
|---|---:|---:|---:|---|
| Read Uncommitted | allowed | allowed | allowed | Statement may see uncommitted writes |
| Read Committed | prevented | allowed | allowed | A fresh committed snapshot per statement |
| Repeatable Read | prevented | prevented | standard permits phantoms | Stable snapshot for the transaction |
| Serializable | prevented | prevented | prevented | Outcome equivalent to some serial order |

- A **dirty read** observes data written by a transaction that may later roll back.
- A **non-repeatable read** occurs when rereading the same row returns a different committed value.
- A **phantom** occurs when rerunning a predicate query returns a different set of rows because another transaction inserted/deleted matching rows.
- A **lost update/write skew** is also worth mentioning. The three classic anomalies do not fully characterize every consistency bug; snapshot isolation can still permit write skew.

Implementations are stronger or different than the minimum standard. PostgreSQL maps `READ UNCOMMITTED` to `READ COMMITTED`, and its MVCC-based `REPEATABLE READ` prevents phantoms but can still allow serialization anomalies such as write skew. PostgreSQL `SERIALIZABLE` uses Serializable Snapshot Isolation and may abort a transaction with SQLSTATE `40001`; the caller must retry the *whole* transaction. MySQL/InnoDB defaults and locking behavior differ, so always name the database when discussing guarantees.

```sql
-- Session A
BEGIN ISOLATION LEVEL READ COMMITTED;
SELECT balance FROM account WHERE id = 7; -- 100

-- Session B commits: UPDATE account SET balance = 80 WHERE id = 7;

-- Session A, Read Committed: now 80 (non-repeatable read)
SELECT balance FROM account WHERE id = 7;
COMMIT;
```

Choose Read Committed for ordinary short OLTP operations whose invariants are enforced by constraints or atomic statements. Use Repeatable Read for a stable analytical view, understanding database-specific write conflicts. Use Serializable for cross-row invariants when correctness outweighs retry overhead. Keep transactions short; do not “solve” anomalies by holding user think-time inside a transaction. Handle `40001`/deadlock errors with bounded exponential-backoff retries, and make side effects idempotent so a retry does not resend an email or charge.

**References:** [PostgreSQL 18 transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html), [PostgreSQL `SET TRANSACTION`](https://www.postgresql.org/docs/current/sql-set-transaction.html)

### 66. How does Multi-Version Concurrency Control (MVCC) operate in relational databases like PostgreSQL to allow non-blocking reads during active updates?

**Interview-ready summary.** MVCC stores multiple logical versions of a row. A statement reads the version visible to its transaction snapshot, while an updater creates a new tuple version rather than overwriting what concurrent readers need. Consequently, ordinary readers and writers generally do not block one another; conflicting writers still coordinate.

In PostgreSQL, each heap tuple contains visibility metadata, conceptually `xmin` (the creating transaction) and `xmax` (the deleting/superseding transaction). An `UPDATE` behaves like an insert of a new tuple plus expiration of the old tuple. A snapshot contains enough transaction-ID information to decide whether a version was committed and visible at that snapshot. Under Read Committed, each statement normally gets a new snapshot; under Repeatable Read/Serializable, the transaction retains a stable snapshot.

```text
Before update:  tuple V1(balance=100, xmin=T10, xmax=0)
T20 updates:    V1(balance=100, xmin=T10, xmax=T20)
                V2(balance= 80, xmin=T20, xmax=0)

Reader whose snapshot predates T20 -> V1
Reader whose snapshot includes committed T20 -> V2
```

Indexes may temporarily lead to multiple physical versions; the visibility check against the heap determines which is usable. PostgreSQL can perform a HOT (heap-only tuple) update when indexed columns are unchanged and page conditions allow it, reducing index churn. Row locks are still taken for conflicting modifications, and DDL or explicit locks can block.

Old versions cannot remain forever. `VACUUM` reclaims dead tuples once no active snapshot can see them, updates visibility information, and prevents transaction-ID wraparound. Autovacuum is therefore a correctness and capacity mechanism, not merely housekeeping. Long-running or “idle in transaction” sessions pin old snapshots, causing table/index bloat, more I/O, replica lag, and wraparound risk. Monitor dead tuples, autovacuum progress, transaction age, and long transactions; set sensible `idle_in_transaction_session_timeout` and tune autovacuum per high-churn table.

MVCC does not mean “no locks.” Two writers touching the same row serialize, unique constraints coordinate concurrent inserts, and Serializable mode tracks predicate dependencies. It also does not automatically enforce every business invariant: snapshot isolation can permit write skew when transactions update different rows after reading a shared condition. Use constraints, atomic conditional updates, `SELECT ... FOR UPDATE`, or Serializable isolation as the invariant requires.

```sql
-- Atomic state transition avoids a read-then-write race.
UPDATE seats
SET status = 'BOOKED', booked_by = :user
WHERE seat_id = :seat AND status = 'AVAILABLE'
RETURNING seat_id;
-- zero rows means another transaction won
```

**References:** [PostgreSQL MVCC introduction](https://www.postgresql.org/docs/current/mvcc-intro.html), [PostgreSQL routine vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html)

### 67. Explain Sharding vs. Partitioning. How do you choose an optimal sharding key to completely eliminate hot spot partition issues?

**Interview-ready summary.** Partitioning splits a logical table into pieces, often inside one database system; sharding distributes pieces across independent database nodes and adds routing, rebalancing, and cross-shard operational concerns. A good shard key has high cardinality, even frequency, non-monotonic write distribution, and appears in common queries. No key can *completely* eliminate hotspots under every workload, so challenge that premise and design detection plus mitigation.

Declarative PostgreSQL range/list/hash partitioning can improve pruning, retention, and maintenance while one cluster still owns the table. Sharding provides horizontal storage/write scale and fault-domain isolation, but cross-shard joins, unique constraints, transactions, schema changes, and resharding become harder. A sharded system may also partition each shard locally.

Evaluate candidate keys with actual traffic:

1. **Cardinality/frequency:** `country` has low cardinality; one celebrity tenant can dominate `tenant_id`. Estimate bytes, QPS, and write QPS per value, including the 99.9th percentile.
2. **Distribution:** an increasing timestamp/order ID routes new writes to the final range. Hashing spreads writes, but sacrifices efficient global range scans.
3. **Query locality:** queries carrying the full shard key route to one shard; missing it creates scatter-gather. Co-locate data that must transact together.
4. **Growth and mobility:** support virtual buckets so a logical tenant maps to many movable units instead of directly to one physical node.
5. **Geography/compliance:** prefix or zone by region when residency and latency require it, then hash within the region.

```text
virtual_bucket = hash(tenant_id, entity_id) mod 4096
physical_shard = shard_map[region][virtual_bucket]

orders primary key: (region, virtual_bucket, order_id)
```

For multi-tenant SaaS, `hash(tenant_id)` is simple but a single huge tenant remains hot. A composite `tenant_id + bucket(user_id/order_id)` spreads that tenant, while a metadata service records its bucket count. Alternatively, isolate “whale” tenants on dedicated shards. Pre-split ranges before a launch, use consistent hashing/virtual nodes, and rebalance asynchronously with dual-read or change-data-capture migration.

Pitfalls include globally unique IDs tied to a central sequence, secondary indexes that require fan-out, cross-shard pagination, and shard-key changes. Prefer client-generated time-sortable IDs whose random bits still distribute writes, and maintain dedicated lookup indexes when users query by a non-shard key. Hash tags or composite prefixes can intentionally co-locate transactional records.

The honest interview answer is: you reduce and bound hotspots, then monitor per-shard CPU, storage, queueing, p99 latency, key-frequency skew, and scatter-gather rate. Add adaptive splitting, admission control, caching/read replicas, tenant quotas, and a resharding runbook. “Perfectly even data” is not enough if one key receives most traffic.

**References:** [MongoDB shard-key selection](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/), [MongoDB shard-key troubleshooting](https://www.mongodb.com/docs/manual/core/sharding-troubleshooting-shard-keys/), [PostgreSQL table partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)

### 68. Explain SQL vs. NoSQL indexing mechanisms. Contrast B-Trees/B+ Trees used in relational systems with Log-Structured Merge (LSM) Trees in write-heavy NoSQL databases.

**Interview-ready summary.** SQL versus NoSQL is not the actual boundary—both families can use B-trees, LSM trees, hashes, inverted indexes, or columnar indexes. A B/B+ tree maintains a page-oriented sorted structure optimized for point and range reads with in-place page updates. An LSM engine batches writes in memory and immutable sorted files, optimizing sustained writes at the cost of compaction and potential read/write amplification.

In a B+ tree, internal pages contain separator keys and child pointers; leaf pages contain keys plus row locators/data and are linked for ordered scans. Its high fan-out keeps tree depth small. Lookup is roughly `O(log_B N)` page visits; range scans seek once and traverse adjacent leaves. Inserts update a leaf and may split/merge pages, generate WAL, and cause random I/O. PostgreSQL calls its default access method B-tree; MySQL/InnoDB stores the entire row in the clustered primary-key B+ tree and the primary key in secondary leaves.

An LSM write typically goes to a write-ahead/commit log and a sorted in-memory memtable. When full, the memtable flushes as an immutable SSTable. Reads check memory and one or more SSTable levels, using Bloom filters, fence pointers, caches, and indexes to skip files. Background compaction merges files, discards obsolete versions/tombstones, and restores level organization.

```text
write -> WAL -> memtable -> flush L0 SSTable
                           -> compact L0 + L1 -> new L1 files
read  -> memtable -> Bloom/index checks -> candidate SSTable blocks
```

Tradeoffs:

- **B+ tree:** predictable reads and ordered scans, usually lower read amplification; random-write/page-split cost, fragmentation, and cache pressure.
- **LSM:** sequential/batched writes, good compression, and high ingestion; read amplification across runs, space amplification during compaction, write amplification from rewriting levels, tombstone delays, and compaction-induced latency spikes.
- **Indexes cost writes:** every secondary index adds storage and mutation work. In an LSM database, a badly chosen compaction strategy can overwhelm disks even when front-door write throughput looks healthy.

```sql
CREATE INDEX ix_orders_customer_created
ON orders (customer_id, created_at DESC)
INCLUDE (status, total);
-- Equality prefix + ordered range; INCLUDE may make this covering.
```

Choose from workload measurements: point/range-read ratio, write rate, working-set size, durability target, acceptable p99 during compaction, and storage budget. Tune B-tree fill factor and vacuum/rebuild strategy; tune LSM memtable size, Bloom filters, leveled versus size-tiered compaction, and compaction I/O limits. Do not claim LSM means NoSQL or B-tree means SQL—Cassandra is LSM-based, while MongoDB's WiredTiger commonly uses B-trees, and SQL engines can expose LSM-backed storage.

**References:** [PostgreSQL index types](https://www.postgresql.org/docs/current/indexes-types.html), [Apache Cassandra storage engine](https://cassandra.apache.org/doc/latest/cassandra/architecture/storage-engine.html), [MySQL 8.4 InnoDB index structure](https://dev.mysql.com/doc/refman/8.4/en/innodb-physical-structure.html)

### 69. How do you implement the Saga Pattern (Orchestration vs. Choreography) to manage cross-service distributed transactions across multiple microservices?

**Interview-ready summary.** A saga turns one distributed ACID transaction into ordered local transactions. Each successful step commits locally and publishes durable progress; a later failure triggers a business **compensation**, not a database rollback. Orchestration uses a durable coordinator/state machine; choreography has participants react to events. Both normally provide eventual consistency and at-least-once delivery, so handlers must be idempotent.

Example order saga:

```text
CreateOrder -> ReserveInventory -> AuthorizePayment -> ArrangeShipment
failure:        ReleaseInventory <- VoidPayment     <- CancelShipment
```

With **orchestration**, an order-saga service stores `saga_id`, current step, attempt, deadline, and outcome. It sends commands, waits for correlated replies, retries on timeout, and issues compensating commands in reverse semantic order. This centralizes visibility, timeouts, and policy and works well for long/branching workflows. Downsides are coordinator complexity, a possible logic bottleneck, and coupling the coordinator to participants. Make it replicated and persist every transition before sending the next command.

With **choreography**, `OrderCreated` causes inventory to emit `InventoryReserved`, which causes payment to emit `PaymentAuthorized`, etc. It is decentralized and easy for short flows, but event dependencies become hard to discover, debug, version, and prevent from cycling as participants grow. Do not let generic integration events silently become an undocumented workflow language.

The critical implementation primitive is a transactional outbox. Update business data and insert an event in the *same local transaction*; a CDC/publisher process sends the outbox row. A consumer records an inbox/deduplication key in the same transaction as its local effect.

```sql
BEGIN;
UPDATE inventory SET available = available - :qty
 WHERE sku = :sku AND available >= :qty;
INSERT INTO outbox(event_id, aggregate_id, type, payload)
VALUES (:eventId, :orderId, 'InventoryReserved', :json);
COMMIT;

CREATE UNIQUE INDEX ux_inbox_consumer_event ON inbox(consumer, event_id);
```

Design compensations explicitly: refund is not “undo charge” after settlement; it is a new audited action that can itself fail. Set timeouts, retry classifications, backoff/jitter, dead-letter/manual-repair states, and a terminal invariant such as `COMPLETED`, `COMPENSATED`, or `NEEDS_REVIEW`. Include `sagaId`, `stepId`, causation/correlation IDs, schema version, and idempotency key in every message. Never hold a database transaction open while calling another service.

Interview pitfalls: assuming global isolation (users can observe intermediate states), compensating irreversible actions, publishing before commit, using message ordering as the sole correctness mechanism, and retrying non-idempotent APIs. Reserve resources with expirations and use semantic locks/status transitions where concurrent sagas conflict.

**References:** [AWS saga patterns](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-patterns.html), [AWS saga orchestration guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-orchestration.html), [AWS transactional outbox pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)

### 70. Explain the Two-Phase Commit (2PC) protocol. Why is it considered an anti-pattern in high-throughput, cloud-native modern microservices?

**Interview-ready summary.** 2PC atomically commits across participants through a coordinator. In phase 1 each participant prepares and durably promises it can commit; in phase 2 the coordinator decides commit only if all voted yes, otherwise rollback. It offers strong atomicity but introduces synchronous coupling, lock retention, blocking failure modes, and poor availability/latency across services.

```text
Coordinator -> participants: PREPARE(tx42)
participants: force redo/undo state, hold locks -> YES/NO
all YES: coordinator durably logs COMMIT -> COMMIT(tx42)
any NO:  coordinator logs ABORT       -> ROLLBACK(tx42)
```

After `YES`, a participant cannot unilaterally abort: it may have to keep prepared state and locks while waiting for the decision. If the coordinator is unavailable, classic 2PC can block. A slow participant extends the critical path; a network partition reduces availability. Recovery requires durable coordinator logs and reconciliation of in-doubt transactions. At cloud scale, cross-region RTT, autoscaling/ephemeral instances, heterogeneous stores, driver/protocol support, and independent service ownership make this operationally expensive. Prepared transactions left behind can prevent vacuum/cleanup and exhaust resources.

It is not universally an “anti-pattern.” Inside a tightly controlled database cluster or a small set of XA-capable resources, for low-volume operations requiring immediate atomicity, 2PC can be appropriate. The anti-pattern is stretching it across independently deployable microservices and external APIs, or believing it removes failure handling. Three-phase commit does not magically solve arbitrary asynchronous partitions; consensus-backed transaction systems make different availability/latency tradeoffs.

PostgreSQL exposes the participant side:

```sql
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 1;
PREPARE TRANSACTION 'transfer-8b7f';
-- Later, after the external coordinator's durable decision:
COMMIT PREPARED 'transfer-8b7f'; -- or ROLLBACK PREPARED
```

For cloud-native services, prefer a local ACID transaction plus transactional outbox, a saga with compensations, or redesign boundaries so one service owns the invariant. Use idempotency keys and reconciliation. If money or inventory cannot temporarily diverge, reserve first and expose a pending state rather than pretending eventual consistency is instantaneous.

Key interview tradeoff: 2PC prioritizes atomic consistency during normal operation but may sacrifice progress during failures; sagas prioritize availability and autonomy but expose intermediate states and require business compensations. State the business invariant and recovery objective before choosing.

**References:** [PostgreSQL `PREPARE TRANSACTION`](https://www.postgresql.org/docs/current/sql-prepare-transaction.html), [PostgreSQL two-phase transaction functions](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-RECOVERY-CONTROL), [AWS saga orchestration motivation](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-orchestration.html)

### 71. How do you handle distributed race conditions across detached application instances? Compare Optimistic Locking (versioning) vs. Pessimistic Locking vs. Distributed Locking (Redlock via Redis).

**Interview-ready summary.** Put concurrency control beside the authoritative state whenever possible. Optimistic locking detects conflicts at commit/update time; pessimistic locking prevents competing database updates by locking; a distributed lock coordinates work outside one transactional store but adds leases, fencing, and failure assumptions. A lock alone does not make a multi-system operation atomic.

**Optimistic locking** suits low/moderate contention and short retries. Read `version`, then condition the update on it:

```sql
UPDATE document
SET body = :body, version = version + 1
WHERE id = :id AND version = :expected;
-- affected rows = 0 => conflict; reload, merge/reject, and retry if safe
```

It avoids waiting and scales well, but high contention wastes work. Do not blindly retry a user edit and overwrite meaning; return HTTP `409 Conflict` or use `If-Match`/ETag. JPA's `@Version` implements this pattern.

**Pessimistic locking** (`SELECT ... FOR UPDATE`) is useful when conflicts are likely, the critical section is short, and the data is in one database. It prevents another locker/writer from proceeding until commit. Set lock/statement timeouts, acquire multiple rows in a canonical order, and retry deadlock victims. Never hold the lock across user input or slow remote calls.

```sql
BEGIN;
SELECT balance FROM account WHERE id = :id FOR UPDATE;
UPDATE account SET balance = balance - :amount WHERE id = :id;
COMMIT;
```

**Distributed locks** are for a shared external resource or cross-instance singleton activity. The basic Redis single-node lease uses `SET resource random-token NX PX ttl`; release only if the stored token matches. Redlock attempts this on a majority of independent Redis masters within the lease validity period. However, pauses, clock/lease expiry, network delay, and a slow former owner mean mutual exclusion at acquisition time is insufficient for correctness-critical storage.

Use a monotonically increasing **fencing token**: each successful lock acquisition gets token `n`; the protected resource rejects operations with tokens lower than the greatest it has accepted. Without the protected resource enforcing fencing, an expired owner can resume and corrupt state. For strict coordination, consider a consensus/lease system such as etcd or ZooKeeper and still fence operations. For database data, prefer a unique constraint, conditional update, advisory lock, or row lock over Redis.

Also make commands idempotent, persist state transitions, and reconcile after ambiguous timeouts. Evaluate contention, failure model, lease duration, maximum pause, clock assumptions, and what happens when the lock service is unavailable. The safest interview conclusion is not “Redlock solves races,” but “correctness resides in conditional state changes/fencing at the resource; locks are coordination aids.”

**References:** [PostgreSQL explicit and row locking](https://www.postgresql.org/docs/current/explicit-locking.html), [Redis distributed locks and Redlock](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/), [Redis `SET` command](https://redis.io/docs/latest/commands/set/)

### 72. Design a real-time notification engine capable of handling 50,000 push, email, and SMS notifications per second with strict prioritization queues.

### Requirements and capacity

Assume 50,000 accepted notification intents/second at peak, multi-channel fan-out, priorities `P0..P3`, per-tenant quotas, user preferences, templates, scheduling, retries, and provider delivery callbacks. Acceptance should be highly available with p99 below 100 ms; channel delivery is asynchronous. “Strict priority” must be clarified: absolute priority can starve lower classes, so use strict priority within a bounded interval plus reserved capacity/aging. Promise at-least-once attempts and idempotent effects—not impossible end-to-end exactly once with third-party providers.

At 50k/s, 4.32 billion intents/day. A 1 KB canonical event is roughly 4.3 TB/day before replication; retain the durable log briefly, archive compressed audit data to object storage, and partition hot status tables. If average fan-out is 1.4 channels, workers see 70k delivery jobs/s. Size each channel independently from measured provider latency and quotas: `concurrency ~= target_rate × p95_provider_latency`, then cap by vendor limits.

### Architecture

```text
Clients -> API Gateway/auth/rate limit -> Notification API
          |-> idempotency/status DB + transactional outbox
          `-> durable event log (partition by tenant/notification)
                -> preference/template fan-out service
                -> P0/P1/P2/P3 queues per channel/region
                     -> push workers -> APNs/FCM
                     -> email workers -> email providers
                     -> SMS workers -> SMS providers
Callbacks -> webhook verifier -> delivery-event log -> status projector
DLQ/replay + scheduler + audit archive + metrics/tracing
```

Use separate physical queues/topics for priority and channel; consumers poll P0 first but reserve, for example, 70/20/8/2% capacity and age old jobs upward. Partition by `tenantId` or `recipientHash` for scale; use a per-recipient sequence only where ordering is genuinely required. Queue lag by priority is the primary autoscaling signal. Batch provider calls where supported and pool connections.

### API and data model

```http
POST /v1/notifications
Idempotency-Key: 98b7...
{
  "tenantId":"t1", "priority":"P0", "templateId":"otp-v4",
  "recipients":[{"userId":"u9"}], "channels":["PUSH","SMS"],
  "variables":{"code":"123456"}, "expiresAt":"2026-08-26T12:01:00Z"
}
-> 202 {"notificationId":"n_...","status":"ACCEPTED"}

GET /v1/notifications/{id}
POST /v1/provider-callbacks/{provider}
```

```sql
notification(id, tenant_id, idempotency_key, priority, template_version,
             payload_ref, scheduled_at, expires_at, created_at)
delivery(id, notification_id, recipient_hash, channel, provider,
         state, attempt, next_attempt_at, provider_message_id, updated_at)
UNIQUE (tenant_id, idempotency_key)
UNIQUE (notification_id, recipient_hash, channel)
```

Store sensitive destinations encrypted/tokenized; keep large payloads outside the queue. Version rendered templates and locale, and evaluate preferences/quiet hours before enqueueing, except explicitly authorized emergency traffic.

### Correctness and failures

Commit intent plus outbox atomically. A worker claims a delivery with a conditional transition (`QUEUED -> SENDING`) and a lease; retries reuse a stable provider idempotency key when supported. Classify errors: retry `429/5xx/timeouts` with exponential backoff/jitter and provider circuit breakers; do not retry invalid addresses or expired OTPs. Ambiguous timeout may mean the provider accepted the request, so reconcile by provider ID/callback rather than immediately duplicating. Verify signed callbacks, deduplicate callback event IDs, and allow out-of-order status transitions only according to an explicit state machine. DLQ after bounded attempts, with safe replay tooling.

Deploy multi-AZ; replicate the log and database synchronously inside a region. For regional failure, route new intents to another region using globally unique IDs and tenant home-region metadata; reconcile delayed replicas. Apply tenant/channel token buckets, overload shedding for P3, and provider failover with cost/compliance controls. Monitor accepted/delivered/failed rate, priority queue age, provider latency/error/429s, duplicate suppression, DLQ size, preference latency, and cost per channel.

**References:** [Amazon SQS throughput and batching](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-throughput-horizontal-scaling-and-batching.html), [Amazon SQS quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-messages.html), [Amazon SNS publishing and filtering](https://docs.aws.amazon.com/sns/latest/dg/sns-publishing.html)

### 73. Design a globally scalable URL shortening service (like Bitly). Detail the API specs, database choices, base62 encoding logic, and caching layers.

### Requirements and estimates

Core operations are create, redirect, optional custom alias, expiry, deletion, and asynchronous analytics. Redirect availability and latency dominate; creation needs uniqueness and idempotency. Assume 100 million new links/month (about 39/s average, perhaps 400/s peak) and 10 billion redirects/day (about 116k/s average, 1M/s peak). At roughly 500 bytes/link, five years of metadata is about 3 TB raw; analytics is much larger and belongs in an event pipeline/object store. Define abuse scanning, tenant quotas, privacy/retention, and whether expired/deleted links return `404` or `410`.

### APIs

```http
POST /v1/links
Idempotency-Key: 7a4c...
{"longUrl":"https://example.com/a?x=1","customAlias":null,
 "expiresAt":null,"domain":"sho.rt"}
-> 201 {"id":"...","shortUrl":"https://sho.rt/aZ91k"}

GET /{code} -> 302 Location: <longUrl>
GET /v1/links/{code}
DELETE /v1/links/{code}
```

Prefer `302/307` while destinations are editable and analytics matters; a cached `301/308` can be difficult to revoke. Validate schemes (`https/http` only), length, punycode/host policy, and block malicious destinations asynchronously with a safe interstitial.

### ID and Base62

Allocate a globally unique numeric ID from ranges (for example, each region leases blocks) or use a distributed ID containing timestamp/region/sequence bits. Base62 is a reversible representation, not encryption; predictable enumeration may require a keyed permutation or random 7–10 character code plus a unique constraint.

```java
private static final char[] ALPHABET =
    "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz".toCharArray();

static String base62(long n) {
  if (n == 0) return "0";
  StringBuilder out = new StringBuilder();
  while (n > 0) {
    out.append(ALPHABET[(int) (n % 62)]);
    n /= 62;
  }
  return out.reverse().toString();
}
```

Seven Base62 characters provide `62^7` (about 3.5 trillion) combinations; capacity is not the same as collision safety for random codes, so calculate birthday-collision probability and retry on the database uniqueness constraint. Treat codes as case-sensitive and avoid locale-dependent transformations.

### Storage, routing, and caches

Use a horizontally partitioned KV/wide-column store keyed by `(domain, code)`, replicated across regions; a relational store also works at moderate write volume and supports ownership/custom-alias constraints. Keep the redirect record compact:

```text
Link{domain, code PK, long_url, owner_id, created_at, expires_at,
     status, url_hash, version}
Unique(owner_id, idempotency_key); Unique(domain, custom_alias)
```

Route users through anycast/geo-DNS to edge/CDN workers. Cache hierarchy: CDN/edge cache -> regional Redis -> authoritative database. Cache positive records with TTL below the edit/revocation SLA; publish invalidations on update/delete. Negative-cache unknown codes briefly to stop penetration, but rate-limit scanners. Use request coalescing/single-flight and TTL jitter to prevent stampedes. A Bloom filter can reject obvious misses, but false positives still query storage and deletions require lifecycle planning.

Redirect path must not synchronously write analytics. Emit `{code, timestamp, coarseGeo, referrer, device}` to a durable stream, sampled/redacted as policy requires, then aggregate into OLAP. If the event stream is down, redirect anyway and buffer/drop analytics according to an explicit loss budget.

### Failure and capacity handling

Creation uses an idempotency key and conditional insert; collisions retry. Multi-region custom aliases need a globally serialized owner or deterministic home region. Stale caches after deletion are a safety risk, so support a fast denylist at the edge. Protect against redirect loops, SSRF in preview fetchers, phishing, Unicode aliases, oversized URLs, and hot viral links. Viral links should be absorbed at CDN/Redis; partitioning only the database is insufficient. Monitor redirect p50/p99, hit ratio by tier, origin QPS, hot keys, 404 rate, creation collisions, replication lag, abuse detections, and analytics lag.

**References:** [IETF RFC 3986 URI syntax](https://datatracker.ietf.org/doc/html/rfc3986), [Redis strings for key/value caching](https://redis.io/docs/latest/develop/data-types/strings/), [Redis client-side caching concepts](https://redis.io/docs/latest/develop/reference/client-side-caching/)

### 74. Design a high-volume financial ledger system. How do you guarantee absolute auditability, idempotency, and partition tolerance?

**Interview-ready summary.** Build an append-only, double-entry ledger whose authoritative write is one ACID transaction. Every journal transaction has balanced debit/credit postings, immutable identifiers, effective and recorded times, currency/asset, and causal metadata. Idempotency is a persisted uniqueness constraint plus request fingerprint. “Absolute” auditability and availability during every partition cannot both be promised: for money-changing writes, reject/queue work when quorum cannot establish a single authoritative order.

### Requirements and invariants

Functional: create accounts, post/void/reverse transfers, retrieve entries/balances/statements, and export/reconcile. Non-functional: no lost/duplicated money, 10k–100k postings/s, deterministic recovery, multi-AZ durability, regulatory retention, encryption, least privilege, and traceable operator actions. State the invariants:

```text
For every journal transaction and asset:
SUM(debits) = SUM(credits)

posted entries are immutable;
corrections are new reversing entries;
available balance never violates configured limits.
```

### Data model and API

```sql
CREATE TABLE journal_tx (
  tx_id UUID PRIMARY KEY, tenant_id UUID NOT NULL,
  idempotency_key TEXT NOT NULL, request_hash BYTEA NOT NULL,
  status TEXT NOT NULL CHECK (status IN ('POSTED','REVERSED')),
  effective_at TIMESTAMPTZ NOT NULL, recorded_at TIMESTAMPTZ NOT NULL,
  correlation_id TEXT, prev_audit_hash BYTEA, audit_hash BYTEA,
  UNIQUE (tenant_id, idempotency_key)
);

CREATE TABLE posting (
  tx_id UUID REFERENCES journal_tx, line_no SMALLINT,
  account_id UUID NOT NULL, asset CHAR(3) NOT NULL,
  direction CHAR(1) CHECK (direction IN ('D','C')),
  amount_minor BIGINT CHECK (amount_minor > 0),
  PRIMARY KEY (tx_id, line_no)
);
```

Use integer minor units or fixed precision—never binary floating point. Account normal-side rules determine whether debit increases an asset and credit increases a liability. Enforce balance with a deferred database trigger/procedure or a single trusted posting service, plus an independent verifier. A materialized balance table is a performance projection, not the source of truth; rebuild it from postings.

```http
POST /v1/ledger/transactions
Idempotency-Key: transfer-123
{"effectiveAt":"...","postings":[
 {"accountId":"payer-cash","asset":"INR","direction":"C","amountMinor":5000},
 {"accountId":"payee-cash","asset":"INR","direction":"D","amountMinor":5000}]}
```

Within one serializable/local transaction: insert the idempotency record, verify the same key has the same canonical request hash, lock/conditionally update affected available-balance rows in canonical account order, insert journal and postings, update projections, and insert an outbox event. A replay returns the original response; same key/different payload is `409`. Provider callbacks also have unique external event IDs.

### Architecture and partitioning

```text
API/auth -> posting service -> leader/quorum ledger shard (WAL + replicas)
                              |-> balance projection
                              `-> transactional outbox -> stream
stream -> statements, risk, notifications, warehouse, reconciliation
immutable backups/WORM archive <- signed audit batches + CDC/WAL
```

Partition by legal entity/tenant/account group so both sides of common transfers share a shard. Do not hash individual postings if it separates a transaction's lines. Cross-shard transfers require a clearing-account workflow: commit balanced legs within each ledger using a durable saga and expose `PENDING` until settlement; reconcile clearing accounts. For strict immediate cross-shard atomicity, accept consensus/distributed-transaction latency and lower availability.

During a network partition, only the side holding quorum/lease may post. A minority becomes read-only or queues authenticated intents; never run two writable primaries. Use fencing terms/epochs so a stale leader cannot write. Synchronous multi-AZ replication and WAL/fsync protect acknowledged commits; cross-region replicas can serve bounded-stale statements, not authoritative spend decisions.

### Audit and failure handling

Restrict `UPDATE/DELETE` on journal tables, retain before/after administrative audit logs, hash canonical records into chained batches, sign batch roots with managed/HSM keys, and export to immutable object retention. Hash chains reveal tampering but do not replace access controls, independent backups, or external reconciliation. Daily reconcile ledger totals against banks/processors and recompute balances from entries; alert on any imbalance.

Handle ambiguous client timeouts through idempotent lookup. Retry serialization/deadlock errors, never retry with a new key. Use a transactional outbox so downstream duplicates cannot change money; consumers maintain inbox keys. Test crash points before/after WAL flush, failover, replica lag, duplicate/reordered messages, reconciliation, backup restoration, and key rotation. Capacity-plan WAL IOPS, indexes, replication bandwidth, posting row growth, hot institutional accounts, and month-end statement reads; bucket only projections, not the audit invariant.

**References:** [PostgreSQL write-ahead logging](https://www.postgresql.org/docs/current/wal-intro.html), [Stripe idempotent requests](https://docs.stripe.com/api/idempotent_requests), [Stripe Treasury transaction entries](https://docs.stripe.com/api/treasury/transaction_entries)

## Part 3: Relational Databases (PostgreSQL / MySQL)

### 75. Write an optimized SQL query to find the 3rd highest-paid employee within each distinct department from an enterprise schema without using loops.

**Interview-ready answer.** Use a window function partitioned by department. Because “3rd highest-paid” usually means the third **distinct salary**, use `DENSE_RANK`; it returns every employee tied at that salary. `ROW_NUMBER` instead chooses exactly one third row, while `RANK` leaves gaps after ties.

```sql
WITH ranked AS (
  SELECT e.employee_id,
         e.employee_name,
         e.department_id,
         e.salary,
         DENSE_RANK() OVER (
           PARTITION BY e.department_id
           ORDER BY e.salary DESC
         ) AS salary_rank
  FROM employee AS e
  WHERE e.salary IS NOT NULL
)
SELECT employee_id, employee_name, department_id, salary
FROM ranked
WHERE salary_rank = 3
ORDER BY department_id, employee_id;
```

Example salaries `100, 100, 90, 80` receive dense ranks `1, 1, 2, 3`, so the employee(s) on 80 are returned. With `RANK`, the ranks would be `1, 1, 3, 4`; with `ROW_NUMBER`, they would be `1, 2, 3, 4` in an order that must be made deterministic.

If the interviewer explicitly wants “the third employee per department, resolving ties,” use:

```sql
WITH ranked AS (
  SELECT e.*,
         ROW_NUMBER() OVER (
           PARTITION BY department_id
           ORDER BY salary DESC, employee_id ASC
         ) AS rn
  FROM employee e
  WHERE salary IS NOT NULL
)
SELECT * FROM ranked WHERE rn = 3;
```

For PostgreSQL, an index such as `(department_id, salary DESC) INCLUDE (employee_id, employee_name)` can reduce sorting or heap access, but the optimizer decides based on table size and visibility; do not claim an index guarantees no sort. A wide covering index increases write cost. If the query runs frequently on a large payroll table, measure `EXPLAIN (ANALYZE, BUFFERS)` and consider a narrower index or a precomputed reporting table. Departments with fewer than three distinct non-null salaries naturally return no row. Decide whether a salary is current, effective-dated, converted to a common currency, or filtered to active employees—enterprise schemas often require these predicates for the question to be meaningful.

For MySQL 8+ and SQL Server, the same window-function form works with minor syntax differences. On legacy engines without window functions, a correlated distinct-count solution exists, but it usually scales worse and is not the preferred interview answer.

**References:** [PostgreSQL window functions](https://www.postgresql.org/docs/current/functions-window.html), [PostgreSQL multicolumn indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html)

### 76. Explain the explicit operational execution differences between a `Clustered Index` and a `Non-Clustered Index`. How does this change physical disk storage?

**Interview-ready summary.** In engines that implement a clustered index, the table's leaf-level rows are stored in the clustered key's order/page structure; therefore a table can have only one clustering organization. A non-clustered/secondary index is a separate ordered structure whose leaves contain the index key plus a row locator (or included columns), so there can be many. Exact behavior is engine-specific.

In MySQL/InnoDB, the primary-key B+ tree is the clustered index and its leaves contain the full row. Each secondary-index leaf stores its key plus the primary-key value; finding non-covered columns needs a second B-tree lookup (“bookmark lookup”) into the clustered index. Thus a wide primary key bloats *every* secondary index.

```sql
CREATE TABLE orders (
  id BIGINT PRIMARY KEY,          -- clustered key in InnoDB
  customer_id BIGINT NOT NULL,
  created_at TIMESTAMP NOT NULL,
  total DECIMAL(12,2) NOT NULL,
  INDEX ix_customer_created (customer_id, created_at)
) ENGINE=InnoDB;
```

The secondary leaf is conceptually `(customer_id, created_at, id)`. A query selecting only those values may be covered; selecting `total` generally follows `id` to the clustered leaf. SQL Server exposes clustered and nonclustered indexes explicitly and can add `INCLUDE` columns. PostgreSQL normally uses heap tables: B-tree leaves hold tuple identifiers; `CLUSTER` physically rewrites a table once but does not continuously maintain that order, so calling every primary key “clustered” is incorrect.

Operational tradeoffs:

- Clustered-key range scans have excellent locality and often fewer I/Os. A monotonically increasing key gives append-like locality, but can make the rightmost page a concurrency hotspot.
- Random UUID clustering causes page splits, fragmentation, poorer cache locality, and larger indexes; time-ordered/random-hybrid IDs can help.
- Updating a clustering key is expensive because it effectively moves the row and changes secondary locators. Choose a narrow, stable, immutable key.
- A non-clustered index speeds selective predicates/orderings but consumes pages, cache, WAL/redo, and maintenance on every insert/update/delete.
- A covering index avoids the base-table lookup but duplicates columns and increases write/storage cost.

“Physical order” is page-level logical organization, not a promise that sequential rows occupy adjacent rotating-disk sectors; buffer pools, SSDs, extents, splits, and storage virtualization intervene. Statistics and selectivity still determine whether the optimizer uses an index, bitmap/index merge, or table scan.

**References:** [MySQL 8.4 clustered and secondary indexes](https://dev.mysql.com/doc/refman/8.4/en/innodb-index-types.html), [MySQL 8.4 physical InnoDB index structure](https://dev.mysql.com/doc/refman/8.4/en/innodb-physical-structure.html), [PostgreSQL `CLUSTER`](https://www.postgresql.org/docs/current/sql-cluster.html)

### 77. How do you profile an expensive, slow-running SQL query? Walk through the interpretation of an `EXPLAIN ANALYZE` output looking for sequential scans vs index scans.

**Interview-ready summary.** First capture the actual SQL, bind values, frequency, latency distribution, waits, and data/plan environment. Then use `EXPLAIN (ANALYZE, BUFFERS, WAL, SETTINGS)` safely on a representative system. Read from the innermost nodes outward and compare estimated versus actual rows, loops, timing, buffer I/O, joins, sorts, and spills—not simply “Seq Scan bad, Index Scan good.”

```sql
EXPLAIN (ANALYZE, BUFFERS, WAL, SETTINGS, FORMAT TEXT)
SELECT o.id, o.total
FROM orders o
WHERE o.customer_id = 42
  AND o.created_at >= CURRENT_DATE - INTERVAL '30 days';
```

Illustrative plan:

```text
Bitmap Heap Scan on orders (cost=120..820 rows=9000)
  (actual time=2.1..14.8 rows=8700 loops=1)
  Recheck Cond: (customer_id = 42)
  Heap Blocks: exact=690
  Buffers: shared hit=600 read=110
  -> Bitmap Index Scan on ix_orders_customer_created
       (actual rows=8700 loops=1)
Planning Time: 0.5 ms  Execution Time: 15.6 ms
```

`cost=a..b` is the planner's dimensionless startup/total estimate, not milliseconds. Multiply a node's per-loop actual rows/time appropriately by `loops`. A large estimated/actual row mismatch suggests stale statistics, correlated columns needing extended statistics, skewed bind values, or non-sargable predicates; bad cardinality propagates into bad join choices. `Buffers: hit` means served from PostgreSQL shared buffers, not necessarily CPU-cache-free; `read` indicates blocks requested from storage/OS. Look for disk sorts (`Sort Method ... Disk`), hash batches, rows removed by filter, nested-loop inner scans repeated thousands of times, lock/I/O waits, and excessive temporary/WAL output.

A sequential scan is optimal when a large fraction of a small/table is needed or random lookups cost more. An index scan is useful for selective predicates and ordering; a bitmap scan batches many row locations. Suspect an avoidable sequential scan when selectivity is high and the predicate matches no usable index—perhaps due to `LOWER(column)`, an implicit cast, leading wildcard, or wrong column order.

```sql
CREATE INDEX CONCURRENTLY ix_orders_customer_created
ON orders (customer_id, created_at)
INCLUDE (id, total);
ANALYZE orders;
```

Verify with production-like data and cold/warm runs. `EXPLAIN ANALYZE` executes the query; wrap mutating statements in `BEGIN ... ROLLBACK` or use a safe replica, and remember triggers/side effects may still occur. Also inspect database-wide tools such as `pg_stat_statements`, slow-query logs, lock views, CPU/I/O saturation, connection-pool wait, and plan regression history. Optimize total workload impact (`mean × calls`) rather than only the single slowest sample.

**References:** [PostgreSQL `EXPLAIN`](https://www.postgresql.org/docs/current/sql-explain.html), [PostgreSQL using `EXPLAIN`](https://www.postgresql.org/docs/current/using-explain.html), [PostgreSQL `pg_stat_statements`](https://www.postgresql.org/docs/current/pgstatstatements.html)

### 78. What is Connection Pooling? How do you tune maximum pool sizing limits using empirical formulas for production databases like HikariCP?

**Interview-ready summary.** A connection pool keeps authenticated database sessions and leases them to application requests, avoiding handshake/session setup and bounding database concurrency. The correct pool is usually much smaller than request concurrency. Tune it through load tests and queueing metrics, under the database's total connection/CPU/I/O budget—not by copying a universal formula.

HikariCP's often-cited starting heuristic is:

```text
active connections ~= (physical CPU cores × 2) + effective disk spindles
```

It is a benchmark starting point, not a law, especially for SSDs, cached datasets, managed databases, and mixed query times. A second capacity view uses Little's Law: if the service needs 1,000 DB operations/s and average connection hold time is 10 ms, average concurrency is about `1000 × 0.010 = 10`; add measured headroom, then test the saturation knee. More connections past that knee increase context switching, memory, lock contention, and tail latency.

Budget across every replica/process:

```text
usable DB connections = max_connections - admin/migration/replication reserve
per-pool cap <= usable / (app instances × pools per instance)
```

Autoscaling makes this dynamic: 50 pods × `maximumPoolSize=20` is 1,000 potential sessions. Use a connection proxy/pooler where appropriate, cap pod count, and reserve emergency access.

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 16
      minimum-idle: 16       # fixed-size behavior; validate against startup bursts
      connection-timeout: 1000
      validation-timeout: 500
      max-lifetime: 1740000  # shorter than DB/network forced lifetime; add jitter if supported
      keepalive-time: 120000
      leak-detection-threshold: 5000 # temporary diagnostic, not a permanent cure
```

Exact values are illustrative. Align timeouts: the HTTP deadline must leave time for pool acquisition and SQL execution; fail fast rather than queue indefinitely. Keep transactions short, close connections in `try-with-resources`, and separate long reporting work from latency-sensitive OLTP if necessary. Avoid setting `minimumIdle` high on hundreds of bursty pods without calculating the idle-session cost.

Run step-load tests while graphing pool active/idle/pending/acquisition p95/p99, connection hold time, request latency, DB CPU, IOPS, locks, cache hit, and TPS. Raise the pool until throughput stops improving or tail latency/waits rise, then back off and retain headroom. A sustained pending queue with idle DB may indicate slow connection creation/networking; a saturated pool and saturated DB calls for query/index/capacity work, not a bigger pool. Validate failover behavior, keepalives, leaked sessions, and connection storms.

**References:** [HikariCP pool-sizing guidance](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing), [HikariCP configuration](https://github.com/brettwooldridge/HikariCP#configuration-knobs-baby), [PostgreSQL connection settings](https://www.postgresql.org/docs/current/runtime-config-connection.html)

### 79. Explain Deadlocks in SQL databases. How does the database engine automatically detect them, and how do you write application code to avoid them?

**Interview-ready summary.** A deadlock is a cycle in the wait-for graph: transaction A holds a resource B needs while B holds one A needs. Waiting longer cannot resolve the cycle. The database detects or times out the cycle, aborts a victim, rolls it back, releases its locks, and returns an error; applications must be prepared to retry the entire idempotent transaction.

```text
T1 locks account 1 -> waits for account 2
T2 locks account 2 -> waits for account 1
                 ^ cycle ^
```

```sql
-- Safe transfer pattern: lock in canonical order in every code path.
BEGIN;
SELECT id FROM account
WHERE id IN (:from_id, :to_id)
ORDER BY id
FOR UPDATE;

UPDATE account SET balance = balance - :amount WHERE id = :from_id;
UPDATE account SET balance = balance + :amount WHERE id = :to_id;
COMMIT;
```

Engines maintain lock owners/waiters and periodically detect cycles or perform detection when a wait occurs. PostgreSQL waits `deadlock_timeout` before running its expensive deadlock check, then cancels one transaction with SQLSTATE `40P01`. InnoDB also detects transactional deadlocks and chooses a transaction to roll back. This is distinct from a simple lock timeout, which may have no cycle.

Avoidance practices:

- Acquire rows/tables in one global order across all endpoints, jobs, and migrations.
- Keep transactions small; do network calls, large computation, and user interaction outside them.
- Index predicates so updates lock/scan only intended rows; understand foreign-key and unique-index locks too.
- Avoid unnecessary isolation/explicit locks, but do not weaken correctness merely to suppress deadlocks.
- Use atomic updates/upserts and process batches in deterministic chunks.
- Make observability include transaction IDs, SQL, lock waits, and deadlock graphs/logs.

Retry only recognized transient errors (`40P01`, and often serialization failure `40001`) with bounded attempts, exponential backoff, and jitter. Roll back first and rerun from the transaction boundary with fresh reads. Ensure external effects use an outbox/idempotency key.

```java
for (int attempt = 1; attempt <= 4; attempt++) {
  try {
    transferInNewTransaction(command); // proxy starts a fresh transaction each call
    return;
  } catch (TransientDataAccessException ex) {
    if (attempt == 4 || !isDeadlockOrSerialization(ex)) throw ex;
    Thread.sleep(backoffWithJitter(attempt));
  }
}
```

Do not catch and continue inside the already-aborted transaction. A zero-deadlock goal is unrealistic in sufficiently concurrent systems; the goal is low frequency, automatic safe recovery, and enough diagnostics to fix repeatable lock-order defects.

**References:** [PostgreSQL explicit locking and deadlocks](https://www.postgresql.org/docs/current/explicit-locking.html#LOCKING-DEADLOCKS), [PostgreSQL lock management settings](https://www.postgresql.org/docs/current/runtime-config-locks.html), [MySQL 8.4 deadlocks](https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlocks.html)

## Part 3: MongoDB and Redis

### 80. How does MongoDB store and look up data using BSON documents? Explain the architecture of replica sets, primary elections, and write concerns.

**Interview-ready summary.** MongoDB represents records as BSON—binary-encoded, ordered documents with typed values such as strings, dates, ObjectIds, arrays, nested documents, decimals, and 32/64-bit integers. Collections store those documents, and indexes map encoded field keys (including nested/multikey values) to records. A replica set has one writable primary and secondaries that replicate its oplog; majority elections choose a primary after failure. Write concern specifies what acknowledgement makes a write successful.

```javascript
db.orders.insertOne({
  _id: ObjectId(),
  customerId: UUID("..."),
  createdAt: new Date(),
  total: Decimal128("1250.50"),
  items: [{ sku: "A7", qty: NumberInt(2) }]
}, { writeConcern: { w: "majority", j: true, wtimeout: 5000 } })

db.orders.createIndex({ customerId: 1, createdAt: -1 })
```

BSON is richer and faster to traverse than plain JSON for database use, but it has overhead and a document-size limit (16 MiB in current MongoDB documentation). Field order can matter for byte equality/compound keys, and numeric types are distinct; use `Decimal128` for money rather than `double`. Embedding makes one aggregate read/atomic update efficient; referencing avoids unbounded arrays and duplication. Indexes support lookup, but each adds RAM/storage/write cost; without a suitable index MongoDB scans documents.

A typical production replica set has three voting, data-bearing members across fault domains. Clients send writes to the primary. The primary records operations in its local oplog; secondaries continuously copy and apply it. If the primary is unreachable beyond election conditions, an eligible secondary seeks a majority vote and becomes primary. Writes pause during election; drivers discover topology changes and can retry supported writes. A former primary rolls back operations that never reached the surviving majority.

Write concern choices express durability/latency:

- `w: 1` acknowledges after the primary accepts the write; it can be rolled back after failover.
- `w: "majority"` waits for acknowledgement by the calculated majority of data-bearing voting members' oplogs and is the normal durability choice.
- `j: true` requests journal durability according to deployment behavior; use a bounded `wtimeout` so the caller does not wait forever, while remembering a timeout is an **ambiguous result**, not proof the write failed.

Pair write concern with read concern/read preference. Reading from a lagging secondary may return stale data; majority/linearizable requirements have availability and latency costs. Use idempotent keys for retry ambiguity, monitor replication lag/oplog window/election frequency, deploy an odd voting topology, and avoid a primary-secondary-arbiter layout when it makes majority acknowledgement depend on every data-bearing member.

**References:** [MongoDB BSON types](https://www.mongodb.com/docs/manual/reference/bson-types/), [MongoDB replica-set elections](https://www.mongodb.com/docs/manual/core/replica-set-elections/), [MongoDB write concern](https://www.mongodb.com/docs/manual/reference/write-concern/)

### 81. Explain Redis data persistence architectures: RDB (snapshots) vs. AOF (Append-Only File). What are the performance and recovery trade-offs of combining both?

**Interview-ready summary.** RDB periodically writes a compact point-in-time snapshot; AOF logs write commands and replays them at startup. RDB favors compact backups and faster bulk recovery but loses changes since the last snapshot. AOF offers a smaller recovery point objective according to its fsync policy, but consumes more disk/I/O and generally takes longer to replay. Combining them provides useful backups plus better recency, not zero-loss durability by itself.

```conf
# RDB examples: snapshot after 1 change/15 min or 10k changes/60 sec
save 900 1
save 60 10000

# AOF
appendonly yes
appendfsync everysec       # usual latency/durability compromise
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

RDB forks a child and relies on copy-on-write pages while creating a temporary snapshot that is atomically renamed. Advantages: compact portable file, good backup/DR artifact, less steady write overhead, and typically faster restart. Drawbacks: the snapshot interval defines possible data loss, and `fork()`/copy-on-write memory plus I/O can cause latency spikes on a large, write-heavy instance.

AOF appends each mutating command. `appendfsync always` offers the strongest local persistence but puts fsync latency in the write path; `everysec` can lose roughly the most recent second around a crash; `no` delegates flush timing to the OS. Background rewrite compacts historical commands into a minimal representation. AOF is larger and rewrite/replay costs matter. Since Redis 7, open-source Redis uses multi-part AOF files (base plus incremental files tracked by a manifest), so operational advice must match the deployed version.

When RDB and AOF are both enabled, Redis uses AOF on restart because it is normally the more complete representation. Keep RDB for periodic/off-site backups and faster inspection/recovery options; use AOF for a tighter RPO. Stagger/monitor `BGSAVE` and AOF rewrite to control memory and disk saturation. Replication improves availability but is not a backup, and asynchronous replication can still lose acknowledged writes during failover.

For a pure disposable cache, no persistence may be correct if the source of truth can refill it safely and a cold-start storm is controlled. For locks, queues, or primary data, define RPO/RTO, test corrupted/truncated-file recovery, restore into an isolated environment, monitor `aof_last_write_status`, rewrite duration, fork latency, memory copy-on-write, disk space, and backup integrity. If truly durable transactional records are required, use an appropriate database as source of truth rather than assuming Redis persistence equals a relational WAL guarantee.

**References:** [Redis persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/), [Redis replication](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/)

### 82. How do you implement a distributed sliding-window rate limiter using Redis sorted sets (`ZSET`) via atomic Lua scripts?

**Interview-ready summary.** Keep one sorted set per limiting subject. Each accepted/request attempt is a unique member scored by timestamp. Atomically remove entries older than the window, count remaining entries, add the current event only if below the limit, and set expiry. Lua prevents two application instances from both observing spare capacity and overspending it.

```lua
-- KEYS[1] = rate:{tenantId}:login  (hash tag keeps one Redis Cluster slot)
-- ARGV[1] = now_ms, ARGV[2] = window_ms, ARGV[3] = limit
-- ARGV[4] = unique request member (timestamp + UUID)
local key       = KEYS[1]
local now       = tonumber(ARGV[1])
local window    = tonumber(ARGV[2])
local limit     = tonumber(ARGV[3])
local member    = ARGV[4]
local oldest    = now - window

redis.call('ZREMRANGEBYSCORE', key, '-inf', '(' .. oldest)
local used = redis.call('ZCARD', key)

if used >= limit then
  local first = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
  local retryAfter = 0
  if #first == 2 then retryAfter = tonumber(first[2]) + window - now end
  redis.call('PEXPIRE', key, window)
  return {0, limit - used, math.max(0, retryAfter)}
end

redis.call('ZADD', key, now, member)
redis.call('PEXPIRE', key, window)
return {1, limit - used - 1, 0}
```

Load once and call with `EVALSHA`; handle `NOSCRIPT` by loading again. Redis executes the script atomically, but it blocks the server while running, so keep it bounded and short. In Redis 7+, managed Redis Functions are another deployment option. Use a UUID member: using only the millisecond timestamp overwrites concurrent members and undercounts.

The exact sliding log is accurate but costs `O(requests in window)` memory per key and removal is `O(log N + M)`. It is appropriate for strict, moderate limits such as login attempts. For massive API traffic, use a sliding-window counter with time buckets or token bucket for bounded memory, accepting approximation/burst semantics. Apply both per-user/API-key and coarser IP/tenant/global limits.

Clock choice matters. Client clocks can skew or be manipulated; a production script can call Redis `TIME` to derive server time, or use a trusted gateway timestamp. Define boundary semantics (`<= oldest` versus `< oldest`) and test them. Return `Retry-After` and standard rate-limit headers, but avoid leaking account existence on authentication routes.

Redis Cluster requires all script keys in one hash slot; the example's `{tenantId}` hash tag provides that. Decide failure policy: fail-closed for password/OTP attempts, perhaps locally degrade/fail-open for low-risk reads. Replication/failover can lose recent limiter entries, so this is abuse control, not a financial invariant. Bound key cardinality, TTL every key, cap untrusted identity lengths, monitor script latency, memory, evictions, denied rate, and hot keys.

**References:** [Redis rate-limiter guidance](https://redis.io/docs/latest/develop/use-cases/rate-limiter/), [Redis Lua atomic execution](https://redis.io/docs/latest/develop/interact/programmability/eval-intro/), [Redis sorted sets and `ZADD`](https://redis.io/docs/latest/commands/zadd/)

### 83. Explain MongoDB indexing strategies. How do compound indexes work, and why does the ordering of fields inside a compound index matter for query optimization?

**Interview-ready summary.** MongoDB indexes keep ordered index keys pointing to documents. A compound index sorts first by its first field, then by the second within equal first-field values, and so on. It efficiently serves queries on the leftmost prefix; field order determines which equality, sort, and range operations can be bounded without scanning or sorting large sections.

Use the ESR heuristic: **Equality**, then **Sort**, then **Range** (with workload-dependent exceptions). Given:

```javascript
db.orders.createIndex({ tenantId: 1, status: 1, createdAt: -1 })

db.orders.find({ tenantId: "t1", status: "PAID" })
         .sort({ createdAt: -1 })
         .limit(50)
```

MongoDB can seek to the equality prefix `(t1, PAID)` and read `createdAt` in index order. The same index supports prefixes `{tenantId}` and `{tenantId,status}`. It does not efficiently support a query only on `status` in the general case because `tenantId` is the leading ordering. A range before a sort field often prevents using later keys to provide a global sort:

```javascript
// Candidate alternatives depend on the dominant query:
{ tenantId: 1, status: 1, createdAt: -1 } // equality + sort/range
{ tenantId: 1, createdAt: -1, status: 1 } // timeline query, status filters later
```

Use `explain("executionStats")` and compare `totalKeysExamined`, `totalDocsExamined`, returned rows, in-memory sort, and execution time on realistic distributions. Field selectivity alone is not the complete ordering rule; equality keys can appear in either order for that query, while sort and range placement strongly affect usable bounds. Index direction matters for mixed-direction compound sorts.

Other strategies:

- A **covered query** projects only indexed fields and avoids document fetch, subject to query/index details.
- **Partial indexes** index only matching documents; queries must imply the filter. Sparse indexes differ and require care with missing fields.
- **Multikey indexes** index array elements, with restrictions on compound arrays and potentially many keys per document.
- TTL indexes expire data asynchronously; text, hashed, wildcard, and geospatial indexes serve specialized access patterns.
- Unique indexes enforce invariants, including deployment-specific sharding rules.

Every index increases RAM/storage, insert/update work, replication traffic, and build/maintenance cost. Avoid redundant prefix indexes only after checking uniqueness/sparsity/collation differences. Be cautious with low-cardinality standalone keys and unbounded arrays. Use hidden indexes to test removal where supported, monitor index usage and cache pressure, and create indexes with the same collation as queries. In a sharded collection, include the shard key in frequent targeted queries; an excellent local index cannot fix scatter-gather routing.

**References:** [MongoDB compound indexes](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-compound/), [MongoDB ESR guideline](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-guideline/), [MongoDB `explain` results](https://www.mongodb.com/docs/manual/reference/explain-results/)

## Part 4: CI/CD, Containers, and Cloud-Native Architecture

### 84. Explain the difference between Docker images and containers. How do copy-on-write file systems and namespaces/cgroups guarantee host isolation?

**Interview-ready summary.** An image is an immutable, content-addressed package of read-only filesystem layers plus metadata such as entrypoint and environment. A container is a running or stopped instance of that image: an isolated host process with a thin writable layer and runtime configuration. Copy-on-write makes instances space-efficient; Linux namespaces isolate views; cgroups account for and limit resources. These mechanisms provide isolation, but “guarantee” is too strong—containers share the host kernel and must be hardened.

When a container reads a file, the overlay/snapshotter resolves it from the topmost layer containing that path. When it first modifies an image file, copy-up places a copy in its private writable layer; deletion creates a whiteout. Multiple containers share immutable lower layers but have independent upper layers. This is efficient for application binaries, not ideal for write-heavy durable databases; use volumes because the writable layer is ephemeral and CoW may add overhead.

Namespaces provide separate views:

- PID: process IDs/process tree; mount: filesystem mounts; network: interfaces/routes/ports.
- IPC: shared-memory/message queues; UTS: hostname; user: maps container IDs to less-privileged host IDs.
- cgroup namespace hides parts of the cgroup hierarchy.

Cgroups enforce/account CPU, memory, process-count, and I/O resources so one container cannot casually consume the entire host. They do not make resources invisible; CPU limits can cause throttling, memory limits can trigger OOM kills, and reservations/limits must be monitored.

```bash
docker run --read-only --user 65532:65532 \
  --cap-drop ALL --security-opt no-new-privileges \
  --pids-limit 200 --memory 512m --cpus 1.0 \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  my-api@sha256:<verified-digest>
```

Security boundaries also depend on Linux capabilities, seccomp, AppArmor/SELinux, read-only mounts, device access, rootless/user namespaces, patched kernels/runtime, and not exposing the Docker socket. `--privileged`, host PID/network, broad bind mounts, or a mounted Docker socket can largely defeat isolation. Image supply-chain controls—minimal bases, signatures/provenance, SBOMs, vulnerability scanning, pinned digests—address a different risk.

Containers are lighter than VMs because they share a kernel; a VM has its own guest kernel/hardware boundary and is generally a stronger isolation boundary. For hostile multi-tenancy, add sandboxed runtimes or VMs. A good interview answer says namespaces/cgroups *implement* process/resource isolation; it does not claim they prove perfect security against kernel/container-runtime vulnerabilities.

**References:** [Docker images](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-an-image/), [Docker storage drivers and copy-on-write](https://docs.docker.com/engine/storage/drivers/), [Docker Engine security](https://docs.docker.com/engine/security/)

### 85. How do you optimize a Dockerfile for a React application to minimize final production image sizes down to under 20MB using multi-stage builds?

**Interview-ready summary.** Compile React in a Node build stage, then copy only static output into a tiny runtime image. Exclude source, package manager, `node_modules`, caches, and build tools from the final image. “Under 20 MB” must specify compressed registry size versus local uncompressed size and includes the generated assets; measure rather than promise it for every app.

For a static SPA served behind a CDN/ingress, BusyBox `httpd` gives a very small runtime:

```dockerfile
# syntax=docker/dockerfile:1
FROM node:24-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci

FROM deps AS build
COPY . .
ENV NODE_ENV=production
RUN npm run build && npm cache clean --force

# Tiny static-file runtime; pin a reviewed digest in production.
FROM busybox:1.37.0-musl AS runtime
WORKDIR /site
COPY --from=build --chown=65532:65532 /app/dist/ ./
USER 65532:65532
EXPOSE 8080
HEALTHCHECK CMD wget -q -O /dev/null http://127.0.0.1:8080/ || exit 1
CMD ["httpd", "-f", "-p", "8080", "-h", "/site"]
```

Create a `.dockerignore` so changes/secrets do not enter build context:

```gitignore
node_modules
dist
.git
.env*
coverage
*.log
Dockerfile*
README.md
```

Some tools output `build/` rather than `dist/`; match the actual bundler. Keep dependency-copy before source-copy so lockfile-stable dependency layers are cached. Use `npm ci`, an exact lockfile, BuildKit cache mounts, and deterministic builds. Remove source maps from the public artifact if policy requires, but archive private maps for error symbolication. Analyze/minify/tree-shake bundles, split routes, compress assets at CDN/proxy, and ensure no runtime secrets were embedded—frontend environment values are visible to users.

BusyBox has limited production HTTP features and does not automatically provide SPA history fallback, sophisticated cache/security headers, Brotli, or TLS. Put it behind an ingress/CDN that supplies those controls, or use a reviewed Nginx/Caddy/unprivileged static-server image and accept/measure its larger size. For client-side routing, configure the gateway to fall back to `/index.html`; returning it for missing `.js` assets can mask bad deployments, so scope the fallback.

Verify both size and contents:

```bash
docker build --pull -t ui:test .
docker image inspect ui:test --format '{{.Size}}'
docker history ui:test
docker run --rm -p 8080:8080 --read-only --tmpfs /tmp ui:test
```

Pin base images by digest, rebuild for security fixes, run as non-root/read-only, scan the final stage, generate an SBOM, and test deep links, cache headers, health checks, and graceful rollout. A small image improves transfer/startup and attack surface, but unsupported minimal components are a worse trade than a few megabytes.

**References:** [Docker multi-stage builds](https://docs.docker.com/build/building/multi-stage/), [Docker build best practices](https://docs.docker.com/build/building/best-practices/), [Docker `.dockerignore`](https://docs.docker.com/build/concepts/context/#dockerignore-files)

### 86. Explain the operational runtime differences between a Kubernetes Deployment, StatefulSet, DaemonSet, and Generic Pod.

**Interview-ready summary.** A Pod is the smallest scheduled unit and has no higher-level replacement/rollout policy by itself. A Deployment reconciles interchangeable stateless replicas through ReplicaSets. A StatefulSet reconciles replicas with stable identity, ordered lifecycle, and per-Pod storage. A DaemonSet ensures a Pod runs on every eligible node (or selected nodes).

| Resource | Identity/scaling | Storage/lifecycle | Typical use |
|---|---|---|---|
| Bare Pod | one named ephemeral instance | no controller recreates it after deletion/node loss | debugging or controller-managed output; rarely direct production use |
| Deployment | fungible Pods, arbitrary names; scale replica count | rolling/recreate update; shared/stateless or external state | APIs, frontends, consumers |
| StatefulSet | stable ordinals such as `db-0`; ordered by default | stable DNS and `volumeClaimTemplates`; PVC follows replacement identity | databases, brokers, clustered stateful apps |
| DaemonSet | one Pod per matching node | follows node arrival/removal; rolling updates supported | log/metric agents, CNI/CSI node components, security agents |

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: api}
spec:
  replicas: 4
  strategy:
    rollingUpdate: {maxSurge: 1, maxUnavailable: 0}
  selector: {matchLabels: {app: api}}
  template:
    metadata: {labels: {app: api}}
    spec:
      containers:
        - name: api
          image: registry.example/api@sha256:<digest>
          readinessProbe: {httpGet: {path: /ready, port: 8080}}
```

A Deployment rollout creates a new ReplicaSet and gradually shifts replica counts; a Service selects ready Pods. Pod names/IPs are disposable. A StatefulSet does **not** make software state-safe automatically: the application must support replication, quorum, backups, failover, and storage fencing. Its headless Service gives stable network identity; deletion behavior of PVCs must be configured and understood. Ordered updates can stop at a broken ordinal, while partitioned updates enable controlled rollout.

A DaemonSet reacts as nodes are added and uses selectors, affinity, and tolerations to decide eligibility. It is not “exactly one forever”: during updates/failures Kubernetes may temporarily replace instances, and unreachable nodes complicate observation. A static Pod, created by kubelet from a manifest on a node, is different from a normal bare Pod or DaemonSet.

For all controller types, requests/limits affect scheduling, probes affect traffic/restarts, disruption budgets constrain voluntary disruption, and affinity/topology spread improves fault isolation. Use Jobs/CronJobs for run-to-completion work. The decision rule: interchangeable replicas -> Deployment; stable identity/storage/order -> StatefulSet; node-local function -> DaemonSet; almost never manage a production Pod directly.

**References:** [Kubernetes workload management](https://kubernetes.io/docs/concepts/workloads/controllers/), [Kubernetes Pods](https://kubernetes.io/docs/concepts/workloads/pods/), [Kubernetes StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

### 87. How does Kubernetes manage service discovery and internal cluster routing via Kube-Proxy and CoreDNS?

**Interview-ready summary.** A Service gives a changing Pod set a stable virtual IP/name. EndpointSlice controllers publish ready backend Pod IPs. CoreDNS watches Kubernetes resources and answers names such as `orders.prod.svc.cluster.local`. Kube-proxy watches Services/EndpointSlices and programs node dataplane rules that translate/load-balance the Service virtual IP to an endpoint. Packets normally do not flow through a kube-proxy userspace process.

Request flow:

```text
client Pod -> DNS query for orders.prod
  -> CoreDNS returns Service ClusterIP 10.96.12.8
client -> 10.96.12.8:8080
  -> node rules installed by kube-proxy
  -> one ready EndpointSlice address, e.g. 10.244.3.21:8080
  -> target Pod
```

CoreDNS's Kubernetes plugin creates A/AAAA records for Services and SRV records for named ports. Search domains in `/etc/resolv.conf` allow short names within a namespace; cross-namespace callers use `orders.prod`. A headless Service (`clusterIP: None`) returns individual endpoint addresses and bypasses virtual-IP load balancing—useful for StatefulSet peer discovery. `ExternalName` returns a CNAME and has HTTP/TLS hostname caveats.

Kube-proxy runs on nodes (typically as a DaemonSet) and reconciles desired Service state into the available proxy backend. Implementations/version choices include iptables/nftables and, in some deployments, IPVS; an eBPF CNI/dataplane can replace kube-proxy entirely. The rules perform destination NAT and endpoint selection. `sessionAffinity`, `internalTrafficPolicy`, topology-aware routing, and `externalTrafficPolicy` alter selection/source-IP behavior. Service routing is generally L4; Ingress/Gateway/service meshes add L7 routing, TLS, retries, or policy.

```yaml
apiVersion: v1
kind: Service
metadata: {name: orders, namespace: prod}
spec:
  selector: {app: orders}
  ports:
    - {name: http, port: 80, targetPort: 8080}
  type: ClusterIP
```

Common failures: selector/label mismatch leaves EndpointSlices empty; readiness failure removes an endpoint; wrong `targetPort`, NetworkPolicy, DNS search/`ndots`, conntrack exhaustion, stale rules, or CNI routing breaks traffic. Debug layer by layer:

```bash
kubectl get svc,endpointslice -n prod
kubectl exec -n prod <pod> -- nslookup orders.prod.svc.cluster.local
kubectl exec -n prod <pod> -- wget -S -O- http://orders.prod/
kubectl logs -n kube-system deployment/coredns
```

CoreDNS and kube-proxy are not the source of application health; readiness gates endpoint eligibility. Run CoreDNS replicas with anti-affinity, monitor DNS latency/errors/cache, EndpointSlice churn, dataplane sync errors, conntrack use, and per-Service failure/latency.

**References:** [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/), [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/), [Kubernetes virtual IPs and service proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/)

### 88. Design a comprehensive CI/CD pipeline using GitHub Actions or Jenkins that performs automated security linting, unit testing, sonarqube analysis, containerization, and canary deployment.

**Interview-ready summary.** Make CI reproducible and unprivileged; run fast quality/security checks before expensive builds; build one immutable image, scan/sign it, promote the same digest through environments, and let a progressive-delivery controller advance or roll back a canary from SLO metrics. Protect production with environment approval, OIDC short-lived credentials, concurrency control, and auditable provenance.

```text
PR: checkout -> dependency cache -> lint/SAST/secret/dependency scan
    -> unit/component tests + coverage -> Sonar quality gate -> build artifact
main: build image once -> SBOM + image scan -> sign/attest -> push by digest
    -> deploy staging -> smoke/integration tests
    -> 5% canary -> automated analysis -> 25% -> 50% -> 100%
    -> rollback on error/latency/saturation regression
```

Condensed GitHub Actions structure (pin every third-party action to a reviewed commit SHA in the real repository; symbolic placeholders below make that policy visible):

```yaml
name: secure-delivery
on: {pull_request: {}, push: {branches: [main]}}
permissions: {contents: read}
concurrency: {group: deploy-${{ github.ref }}, cancel-in-progress: false}

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<approved-sha>
      - uses: actions/setup-java@<approved-sha>
        with: {distribution: temurin, java-version: '21', cache: maven}
      - run: ./mvnw -B spotless:check test jacoco:report
      - run: ./mvnw -B org.owasp:dependency-check-maven:check
      - uses: github/codeql-action/init@<approved-sha>
        with: {languages: java-kotlin}
      - run: ./mvnw -B package -DskipTests
      - uses: github/codeql-action/analyze@<approved-sha>
      - uses: SonarSource/sonarqube-scan-action@<approved-sha>
        env: {SONAR_TOKEN: "${{ secrets.SONAR_TOKEN }}"}

  image:
    if: github.ref == 'refs/heads/main'
    needs: verify
    permissions: {contents: read, packages: write, id-token: write, attestations: write}
    outputs: {digest: "${{ steps.build.outputs.digest }}"}
    steps:
      - uses: actions/checkout@<approved-sha>
      - uses: docker/setup-buildx-action@<approved-sha>
      - uses: docker/login-action@<approved-sha>
        with: {registry: ghcr.io, username: "${{ github.actor }}", password: "${{ secrets.GITHUB_TOKEN }}"}
      - id: build
        uses: docker/build-push-action@<approved-sha>
        with: {context: ., push: true, tags: ghcr.io/acme/api:${{ github.sha }}, provenance: true, sbom: true}
      - run: trivy image --exit-code 1 --severity HIGH,CRITICAL ghcr.io/acme/api@${{ steps.build.outputs.digest }}
      - run: cosign sign --yes ghcr.io/acme/api@${{ steps.build.outputs.digest }}

  production:
    needs: image
    environment: production
    permissions: {contents: read, id-token: write}
    steps:
      - run: ./scripts/update-rollout-image.sh '${{ needs.image.outputs.digest }}'
      - run: kubectl argo rollouts status api --timeout 20m
```

The production script should commit a digest to GitOps or patch an Argo Rollout, not rebuild. A rollout analysis queries Prometheus for canary error rate, p95/p99 latency, and saturation versus baseline, with minimum sample size and abort thresholds. Synthetic smoke tests catch functional breakage; database migrations use expand/contract compatibility and a separately reviewed job. Keep rollback artifacts and make rollout cancellation automatic.

Pipeline hardening: fork PRs never receive secrets; actions are SHA-pinned and updated by automation; use artifact attestations/signature admission policy; isolate self-hosted runners; mask logs; set timeouts; cache only trusted keys; upload test/SAST/Sonar reports; require branch reviews and quality gates. Avoid long-lived cloud keys by GitHub OIDC. Parallelize independent scans, but do not allow flaky tests to become ignored gates.

Measure pipeline lead time, failure/flake rate, scan coverage, deployment frequency, change-failure rate, rollback time, and canary decision quality. Test the rollback path and disaster credentials regularly.

**References:** [GitHub Actions deployment controls](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/control-deployments), [GitHub CodeQL workflow guidance](https://docs.github.com/en/code-security/code-scanning/integrating-with-code-scanning/using-code-scanning-with-your-existing-ci-system), [Argo Rollouts canary strategy](https://argo-rollouts.readthedocs.io/en/stable/features/canary/)

### 89. Explain the technical differences between rolling updates, blue-green deployments, and canary release strategies. How do you roll back safely?

**Interview-ready summary.** Rolling replaces instances incrementally in one serving fleet; blue-green maintains two complete environments and switches traffic; canary exposes a small, controlled audience to the new version and expands based on evidence. Rollback safety depends as much on data/schema and external side effects as on switching application traffic.

| Strategy | Traffic/capacity | Strengths | Main risks/cost |
|---|---|---|---|
| Rolling | old/new coexist as replicas are replaced | simple, resource-efficient, native Kubernetes | mixed versions; rollback is another rollout; bad version reaches traffic throughout |
| Blue-green | blue and green are complete; router flips | fast cutover/rollback, strong pre-prod validation | near-double capacity; environment/data drift; active sessions/jobs |
| Canary | 1–5% then stages by weight/cohort | limits blast radius; validates real traffic/SLOs | routing/analysis complexity; low sample sizes; biased cohorts |

Kubernetes Deployment rolling behavior is controlled by `maxSurge` and `maxUnavailable`:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 0
minReadySeconds: 30
progressDeadlineSeconds: 600
```

Readiness/startup probes keep unready instances out of Services; a PodDisruptionBudget helps voluntary disruption but does not guarantee capacity during every failure. Blue-green changes a Service/Gateway selector or load-balancer target after green passes smoke/warmup tests. Canary needs weighted L7 routing or separate Services, and should compare new versus baseline error rate, latency, saturation, and business outcomes—not CPU alone. Abort on hard safety signals; pause when evidence is insufficient.

Safe rollback checklist:

1. Deploy immutable, signed image digests and retain known-good configuration/ReplicaSets.
2. Use backward/forward-compatible **expand-contract** database changes: add nullable/new structures, dual-read/write if necessary, backfill, switch code, and remove only after old code can no longer run. Never pair a canary with an immediately destructive migration.
3. Make messages/events version-compatible and side effects idempotent. Rolling back code cannot “un-send” emails, payments, or writes; use compensation/reconciliation.
4. Drain connections/jobs gracefully; handle session compatibility and cache namespace/version changes.
5. Automate stop/rollback from SLO thresholds, but include minimum traffic/window and guard against telemetry failure.
6. Verify recovery with smoke tests and continued monitoring; record the release/incident timeline.

For a Deployment, `kubectl rollout undo deployment/api --to-revision=N` restores a prior Pod template, not the database or external state. Blue-green rollback flips routing to blue only while blue is healthy and data-compatible. Canary rollback sets new weight to zero, then scales it down after draining. Feature flags can disable a risky code path faster than redeployment, but flags require ownership, secure defaults, audit, and cleanup.

Choose rolling for routine low-risk stateless changes, blue-green where fast full-environment cutover justifies cost, and canary for high-traffic/high-risk behavior where live comparative metrics add value. Combinations are common: rolling within each canary ReplicaSet or blue-green with a short canary preview.

**References:** [Kubernetes Deployment rolling updates and rollback](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/), [Kubernetes update-without-downtime task](https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/), [Argo Rollouts blue-green strategy](https://argo-rollouts.readthedocs.io/en/stable/features/bluegreen/)

## Part 4: High Availability, Monitoring, and Security

### 90. Explain the operational differences between horizontal autoscaling (HPA) and vertical autoscaling (VPA). What metrics should trigger scaling actions?

**Interview-ready summary.** HPA changes replica count; VPA changes each Pod's CPU/memory requests (and, according to policy/platform capability, limits). HPA is ideal for parallelizable stateless work and reacts without resizing every process. VPA right-sizes resource envelopes for workloads that cannot or need not add replicas, but resizing can require Pod replacement and therefore disruption. They solve different bottlenecks and neither creates node capacity—that is a node autoscaler's job.

HPA periodically compares observed metrics with targets. Conceptually, Kubernetes uses a ratio such as `desiredReplicas = ceil(currentReplicas × currentMetric / desiredMetric)`, with tolerance, readiness handling, stabilization, and rate policies to avoid flapping.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: {name: api}
spec:
  scaleTargetRef: {apiVersion: apps/v1, kind: Deployment, name: api}
  minReplicas: 4
  maxReplicas: 60
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: {type: Utilization, averageUtilization: 65}
    - type: Pods
      pods:
        metric: {name: http_inflight_requests}
        target: {type: AverageValue, averageValue: "30"}
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies: [{type: Percent, value: 100, periodSeconds: 60}]
    scaleDown:
      stabilizationWindowSeconds: 300
      policies: [{type: Percent, value: 20, periodSeconds: 60}]
```

CPU is useful for CPU-bound services, but utilization depends on accurate CPU **requests**; bad requests produce bad HPA math. Memory is often a poor scale-down signal because caches do not release memory. Prefer demand/concurrency metrics closest to saturation: requests or active work per Pod for APIs, queue age/backlog divided by processing rate for consumers, event-loop lag for Node.js, thread-pool queue for Java, and dependency-safe throughput. Scale early enough to cover Pod startup plus load-balancer readiness. Never scale only on latency if the downstream database is already saturated—more replicas may amplify the outage.

VPA's recommender learns usage and proposes request values; updater/admission components apply them according to modes/policies. Cap min/max resources and inspect recommendations before automatic use on sensitive stateful workloads. In-place resize support and VPA availability vary by Kubernetes distribution/version; otherwise Pods are evicted/recreated. Protect with disruption budgets and enough replicas.

Avoid HPA and VPA fighting over the same CPU/memory signal: VPA changing requests changes HPA utilization. Common combinations are VPA recommendation-only for requests plus HPA on custom/external demand metrics, or VPA for one resource and HPA for another. Coordinate with Cluster Autoscaler/Karpenter so unschedulable scaled Pods cause node scale-up.

Monitor desired/current/ready replicas, pending/unschedulable Pods, throttling, OOMs, queue age, startup time, scaling events, node headroom, SLOs, and downstream saturation. Load-test thresholds and failure modes; use stabilization, cooldown, predictive/scheduled minimums for known spikes, and hard maximums for cost/dependency protection.

**References:** [Kubernetes autoscaling workloads](https://kubernetes.io/docs/concepts/workloads/autoscaling/), [Kubernetes HPA algorithm](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/), [Kubernetes resource metrics pipeline](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)

### 91. How do you configure centralized application logging and metric tracking using the ELK Stack (Elasticsearch, Logstash, Kibana) or Prometheus and Grafana?

**Interview-ready summary.** Logs and metrics are complementary. Emit structured application logs to stdout, collect them with an agent, parse/enrich/redact, then index them in Elasticsearch and explore in Kibana. Expose bounded-cardinality numeric metrics, let Prometheus scrape them, define recording/alert rules, and visualize them in Grafana. Correlate both with service/version/environment and trace IDs.

```text
Pods/VM apps --JSON stdout--> Fluent Bit/Filebeat/OTel Collector
     -> optional Logstash transform/buffer -> Elasticsearch data streams -> Kibana

apps/exporters --/metrics--> Prometheus --rules--> Alertmanager
                                      `---------> Grafana
```

Application log example (one event, not a multi-line string):

```json
{"ts":"2026-08-26T12:00:01.123Z","level":"ERROR","service":"checkout",
 "env":"prod","version":"a1b2c3","trace_id":"4bf9...","span_id":"00f0...",
 "event":"payment_authorization_failed","order_id":"o_42",
 "provider":"p1","error_type":"timeout","duration_ms":2003}
```

Do not log passwords, tokens, full card numbers, or sensitive payloads; redact at source and again in the pipeline. Use an ingest schema/template so timestamps and numeric fields are typed correctly. Elasticsearch lifecycle/data-stream policies roll hot indices, move/delete old data, and bound cost. Configure replicas, disk watermarks, ingestion backpressure, authentication/TLS, tenant access, and snapshot restore. Log agents should buffer to disk with limits; an outage must not fill every application node.

Prometheus exposition from Spring Boot might use Actuator/Micrometer; Node.js can use a Prometheus client. A scrape configuration is conceptually:

```yaml
scrape_configs:
  - job_name: api
    kubernetes_sd_configs: [{role: pod}]
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        regex: "true"
        action: keep
```

Prefer counters for totals, gauges for current state, and histograms for latency/size distributions. Never label metrics with `user_id`, `request_id`, raw URL, or exception message—unbounded cardinality can exhaust Prometheus. Normalize routes (`/orders/{id}`), use recording rules for expensive queries, and remote-write/federate according to retention/HA needs. Prometheus is excellent operational telemetry, not an exact billing ledger.

Grafana dashboards should show SLO/error-budget and four golden signals first, then drill down by region/service/version/dependency. Alerts go through Alertmanager with grouping, deduplication, routing, inhibition, and runbook links. Alert on user-impacting symptoms plus actionable causes, not every log error. Validate alerts with synthetic failures.

ELK can also derive metrics, and Prometheus exemplars can link a histogram observation to a trace. In new deployments, OpenTelemetry Collector can standardize collection/export without requiring one backend. Monitor the observability stack itself: dropped logs/samples, ingest lag, Elasticsearch shard/disk health, scrape failures, rule duration, alert delivery, and cost.

**References:** [Elastic data streams](https://www.elastic.co/docs/manage-data/data-store/data-streams), [Prometheus overview](https://prometheus.io/docs/introduction/overview/), [Grafana dashboards and visualizations](https://grafana.com/docs/grafana/latest/visualizations/)

### 92. What are the key metrics you monitor on a production dashboard to track the health of a distributed microservice layer? (Explain the Four Golden Signals).

**Interview-ready summary.** Start with user-facing SLIs and Google's four golden signals: **latency, traffic, errors, and saturation**. Break them down by service, endpoint class, region, dependency, and release without creating unbounded labels. Tie alerts to SLO/error-budget impact, then provide resource/runtime/database drill-downs for diagnosis.

1. **Latency:** request/operation distribution, especially p50/p95/p99 and successful versus failed latency. A fast error is not success; average hides tails. Track end-to-end and downstream spans, queue wait separately from service time.
2. **Traffic:** requests/s, messages/s, bytes/s, active users/connections, and business operations such as checkout attempts. It supplies context for every other graph.
3. **Errors:** HTTP 5xx, meaningful 4xx, RPC status, timeout/circuit-breaker/retry failures, message DLQ, and incorrect-result/business failure rate. Measure numerator and denominator.
4. **Saturation:** how close a constrained resource is to capacity—CPU throttling, memory/OOM, worker/event-loop/thread-pool queue, DB pool/locks, connection limits, disk/IOPS, queue age, Kafka consumer lag, and downstream quotas.

Useful PromQL patterns (adapt metric names):

```promql
# request rate
sum by (service) (rate(http_requests_total[5m]))

# 5xx ratio
sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
/
sum by (service) (rate(http_requests_total[5m]))

# p99 from a histogram; use a bucket scheme suitable for the SLO
histogram_quantile(0.99,
  sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))
```

Dashboard hierarchy:

- **Global/SLO:** availability, latency objective compliance, burn rate, active incidents, traffic, regional health, deployment markers.
- **Service:** golden signals by route/status/version; instance count/restarts/readiness; retry amplification and circuit state.
- **Dependencies:** database query/pool latency, cache hit/eviction, broker produce/consume lag, external-provider quota/error.
- **Runtime/infra:** JVM heap/GC pauses/thread pools; Node event-loop lag/heap/GC; container CPU throttle/memory; node/network/DNS/TLS health.
- **Correctness/business:** duplicate suppression, reconciliation mismatches, orders/payment success, freshness/replication lag. These catch “HTTP 200 but wrong result.”

Pitfalls: alerting on CPU alone, averaging percentiles, summing already-computed quantiles, high-cardinality IDs, treating retries as independent traffic, and dashboards with no ownership/runbook. Use multi-window burn-rate alerts to page on fast and slow SLO consumption, ticket capacity trends, and avoid paging on transient causes without user impact. Correlate release/config changes and trace exemplars.

Every dashboard must answer: Are users affected? What scope? Since when/which change? Which dependency/resource is saturated? What action/runbook applies? Test instrumentation and alert delivery; missing telemetry should itself be visible, because a blank graph is not a healthy system.

**References:** [Google SRE—Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/), [Google SRE Workbook—SLO alerting](https://sre.google/workbook/alerting-on-slos/), [Prometheus metric types](https://prometheus.io/docs/concepts/metric_types/)

### 93. Explain the architectural setup of Distributed Tracing using OpenTelemetry and Jaeger. How does a unique correlation ID map requests across service networks?

**Interview-ready summary.** Instrument each service to create spans and propagate W3C trace context. A trace has one trace ID; every operation has its own span ID and parent relationship. OpenTelemetry SDKs/exporters send sampled spans—usually via an OpenTelemetry Collector—to Jaeger for storage/query/visualization. Put trace/span IDs into structured logs. A business correlation ID may coexist, but it is not a substitute for trace context.

```text
browser/gateway [trace T, span A]
  -> order service [T, child B]
       -> DB [T, child C]
       -> payment HTTP [inject traceparent]
            payment service extracts -> [T, child D]

SDK/auto-instrumentation -> OTLP -> Collector(s) -> Jaeger backend/storage/UI
```

The standard HTTP header looks like:

```text
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             version | 128-bit trace-id             | parent span | flags
```

On ingress, instrumentation extracts a valid context or starts a new root. On egress it injects the current context. The receiver creates a child span with the same trace ID and a new span ID. For messaging, propagate context in message headers; a consumer span may be a child or use a span link for batch/fan-out semantics. Async thread/reactive context must be carried by the runtime instrumentation—`ThreadLocal` alone is insufficient across every executor/reactor boundary.

Collector configuration sketch:

```yaml
receivers:
  otlp:
    protocols: {grpc: {}, http: {}}
processors:
  memory_limiter: {limit_mib: 512}
  batch: {send_batch_size: 1024, timeout: 2s}
  tail_sampling:
    decision_wait: 10s
    policies:
      - {name: errors, type: status_code, status_code: {status_codes: [ERROR]}}
exporters:
  otlp/jaeger: {endpoint: jaeger-collector.observability:4317, tls: {insecure: false}}
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, tail_sampling, batch]
      exporters: [otlp/jaeger]
```

Use semantic attributes such as service name/version, HTTP route (not raw ID-bearing path), RPC status, database system/operation, region, and deployment. Do not put secrets, SQL values, PII, or high-cardinality baggage into spans. Never trust an external trace ID for authorization; validate format and optionally start a new internal trace at trust boundaries. W3C `baggage` propagates and can leak broadly, so keep it tiny/non-sensitive.

Sampling is essential: head sampling is cheap but may miss rare failures; tail sampling sees completed traces but requires buffered collector capacity and consistent trace routing. Preserve all errors/high-latency traces plus a representative baseline within cost limits. Deploy collectors redundantly, set bounded queues/retries, and ensure telemetry backpressure never takes down the application.

A custom `X-Correlation-ID` is useful for a stable customer/workflow ID across retries or multiple traces; sanitize/generate it at the edge and log it. Trace ID maps one causal execution; idempotency key maps a command retry; business ID maps a domain entity—do not conflate them.

**References:** [OpenTelemetry context propagation](https://opentelemetry.io/docs/concepts/context-propagation/), [OpenTelemetry Collector configuration](https://opentelemetry.io/docs/collector/configuration/), [Jaeger architecture](https://www.jaegertracing.io/docs/latest/architecture/)

### 94. Explain the top 3 OWASP Top 10 vulnerabilities (SQL Injection, XSS, CSRF) and detail the concrete full-stack architectural remedies for each.

**Version note.** SQL injection, XSS, and CSRF are important web risks, but they are not literally the top three standalone categories in the current published OWASP Top 10 (the 2021 list). Injection is A03 and includes SQL/XSS-related risks; CSRF is covered through broader secure-design/authentication concerns and dedicated OWASP guidance. In an interview, answer the three requested vulnerabilities while showing this nuance.

### SQL injection

Untrusted input changes the structure of a SQL command, typically through string concatenation. The primary remedy is parameterization—not regex escaping.

```java
String sql = "SELECT id, role FROM users WHERE email = ?";
try (PreparedStatement ps = connection.prepareStatement(sql)) {
  ps.setString(1, email);
  try (ResultSet rs = ps.executeQuery()) { /* ... */ }
}
```

ORMs are safe only when using bound parameters; raw/native string construction is still vulnerable. Identifiers such as sort column cannot be bound, so map them through an allowlist. Add least-privilege DB roles, separate migration/runtime credentials, network segmentation, input length/type validation, secret rotation, query timeouts, SAST/DAST, and monitoring. A WAF is defense-in-depth, not the fix.

### Cross-site scripting (XSS)

Untrusted data executes as browser script through stored, reflected, or DOM sinks. Use context-sensitive output encoding and safe DOM/framework APIs. React escapes text interpolation:

```jsx
<p>{comment.text}</p> // text is escaped
```

Avoid `dangerouslySetInnerHTML`, `innerHTML`, `document.write`, string-built JavaScript/URLs, and unsafe template bypasses. If users may author HTML, sanitize with a mature allowlist library and keep it patched; validate URL schemes. Add a nonce/hash-based Content Security Policy, disallow inline/eval where possible, set cookies `HttpOnly; Secure; SameSite`, and use Trusted Types where supported. CSP limits impact but does not replace encoding/safe sinks.

### Cross-site request forgery (CSRF)

A malicious site causes a victim browser to send an authenticated state-changing request because cookies are attached automatically. For cookie/session authentication, use framework CSRF middleware with synchronizer tokens or a correctly implemented signed double-submit cookie; verify token on unsafe methods. Add `SameSite=Lax/Strict` as appropriate, check `Origin` (and carefully `Referer`) as defense-in-depth, require custom headers for AJAX APIs, and never change state via `GET`.

```http
Set-Cookie: session=...; HttpOnly; Secure; SameSite=Lax
X-CSRF-Token: <unpredictable token bound to session>
```

An API using an access token only in an `Authorization` header that browsers do not attach automatically is generally not vulnerable in the same way—but storing that token in readable browser storage increases XSS impact. CORS is not a CSRF defense by itself, and HTTPS does not prove user intent. Re-authenticate/step-up for sensitive actions.

Test all three in CI and threat models, centralize security defaults, encode at output context, and log blocked events without sensitive payloads. XSS can defeat CSRF controls, so remedies must be layered.

**References:** [OWASP SQL Injection Prevention](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html), [OWASP XSS Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html), [OWASP CSRF Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)

### 95. How do you protect an exposed, public-facing Node.js or Spring Boot REST API from Distributed Denial of Service (DDoS) and brute force credential stuffing attacks?

**Interview-ready summary.** Use defense in depth before traffic reaches the process: anycast/CDN and managed DDoS protection for volumetric L3/L4 attacks, WAF/bot controls and rate limits for L7, gateway authentication/quotas, then cheap application validation, bounded resources, and load shedding. Credential stuffing additionally needs breached-password resistance, adaptive identity-aware controls, MFA/passkeys, and detection. No in-process Express/Spring limiter can absorb a saturated network link.

```text
Internet -> CDN/Anycast + DDoS scrubbing -> WAF/bot challenge
 -> API gateway (TLS, body limits, IP/API-key/device/tenant token buckets)
 -> load balancer -> bounded API instances -> protected dependencies
```

At the edge, cache safe public responses, block malformed protocols/bodies, restrict methods/content types, enforce header/body/upload size and request/idle timeouts, and use layered rate rules: a broad flood limit plus stricter `/login`, `/signup`, `/password-reset`, OTP, and expensive search limits. Rate limit by multiple signals because IP-only rules harm NAT users and attackers rotate proxies. Apply per-account, per-device/risk, per-API-key, per-tenant, ASN/geo reputation, and global dependency budgets. Return `429` with `Retry-After`; add jitter and avoid synchronization.

Application hardening:

- Bound server threads/queues, DB/HTTP pools, file descriptors, decompression, parsing depth, pagination, regex complexity, and fan-out. Stream uploads and cancel work when clients disconnect.
- Use circuit breakers, bulkheads, concurrency limits, deadlines, cached fallbacks, and priority load shedding. Reserve capacity for health/admin/high-priority traffic.
- Scale on concurrency/queue age as well as CPU, but cap scale so an attack cannot create an unlimited bill or overload the database.
- Keep generic authentication responses/timing so attackers cannot enumerate accounts. Do not impose permanent hard lockouts that attackers can weaponize; use progressive delay, temporary/risk-based challenges, and notify users.

Credential stuffing defenses include passkeys/WebAuthn or MFA (prefer phishing-resistant methods), breached-password checks at password creation, strong salted adaptive password hashing, rotating refresh tokens/session revocation, impossible-travel/device-risk analysis, bot detection, and monitoring one IP-many-accounts plus one account-many-IPs. CAPTCHA is friction and can be bypassed; use it adaptively. Never log credentials.

For an application-level distributed limiter, Redis plus atomic scripts can coordinate instances, but define outage behavior and protect Redis itself. Trust proxy-derived client IP headers only from known load balancers; otherwise attackers spoof `X-Forwarded-For`. Spring Security/Express middleware should enforce authorization after gateway checks because gateways can be bypassed by misconfiguration.

Prepare operationally: baseline traffic, WAF rules in count mode before blocking, autoscaling/cost alarms, origin allowlisting so only the CDN reaches it, multi-region/provider runbooks, emergency coarse rules, and tested contact/escalation. Observe requests/bytes/connections, TLS handshakes, WAF actions, unique sources, cache hit, auth success/failure by risk cohort, queue/DB saturation, 429/5xx, and cost. Preserve privacy and avoid automatically blocking legitimate corporate/mobile networks.

**References:** [AWS DDoS resilience best practices](https://docs.aws.amazon.com/whitepapers/latest/aws-best-practices-ddos-resiliency/welcome.html), [AWS WAF rate-based rules](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-rate-based.html), [OWASP Credential Stuffing Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Credential_Stuffing_Prevention_Cheat_Sheet.html)

### 96. Explain the difference between symmetric and asymmetric encryption. How is TLS/SSL initialized under the hood during a client-server web handshake?

**Interview-ready summary.** Symmetric cryptography uses the same secret key for encryption/decryption and is fast for bulk data (for example, AES-GCM or ChaCha20-Poly1305). Asymmetric cryptography uses a public/private key pair and is costlier; it supports signatures, identity, and key agreement. Modern TLS combines them: certificates/signatures authenticate, ephemeral asymmetric key agreement derives shared secrets, and symmetric AEAD protects records. “SSL” is obsolete terminology; current secure deployments use TLS, commonly TLS 1.3.

Symmetric encryption's key-distribution problem is that both parties need the secret. Asymmetric encryption permits public-key distribution, but public keys must be authenticated—normally through X.509 certificates and trusted certificate authorities. Encryption and signing are different operations: a signature uses a private key to prove integrity/authorship and is verified by the public key.

Simplified full TLS 1.3 handshake:

```text
Client -> Server: ClientHello
  supported versions/cipher suites, random, extensions (SNI/ALPN),
  ephemeral ECDHE key share

Server -> Client: ServerHello + selected suite + ephemeral key share
  [both derive handshake secrets from ECDHE via HKDF]
Server -> Client (encrypted): EncryptedExtensions, Certificate,
  CertificateVerify signature, Finished

Client validates hostname, chain, validity/policy and CertificateVerify
Client -> Server (encrypted): Finished
<-> application data protected with symmetric AEAD traffic keys
```

Both ephemeral key shares produce the same ECDHE shared secret without transmitting it. HKDF derives separate directional handshake/application keys and IVs. The server signs the handshake transcript with the certificate private key; the client validates the chain to a trust anchor and matches the requested hostname. Finished messages authenticate the transcript and derived keys. TLS 1.3 therefore should not be explained using the old RSA “client encrypts a premaster secret with the server certificate” flow; that describes legacy TLS modes and lacks modern forward secrecy.

Ephemeral Diffie-Hellman provides forward secrecy: later compromise of the certificate private key does not decrypt previously recorded sessions. AEAD gives confidentiality and integrity; nonces must never repeat for the same key. Keys rotate according to the protocol/implementation. Mutual TLS adds a client Certificate/CertificateVerify for machine identity, but authorization remains an application decision.

Session tickets/PSKs allow resumption with fewer round trips. TLS 1.3 0-RTT early data can be replayed, so do not allow non-idempotent money/state-changing requests in early data unless there is an explicit anti-replay design. SNI selects the virtual host; ALPN negotiates protocols such as HTTP/2. DNS/TCP or QUIC setup surrounds the TLS exchange depending on HTTP version.

Operational pitfalls: disabled certificate validation, hostname mismatch, stale trust stores, weak legacy protocols/ciphers, private keys in images, no rotation, terminating TLS at a proxy while leaving an untrusted internal hop plaintext, and confusing transport encryption with end-to-end authorization. Automate certificate issuance/renewal, protect keys in a managed KMS/HSM where appropriate, prefer modern library defaults, enable TLS 1.2 only when compatibility requires it, test configuration, and monitor expiry/handshake failures.

**References:** [IETF RFC 8446—TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446), [OWASP Transport Layer Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Security_Cheat_Sheet.html), [NIST cryptographic standards and guidelines](https://csrc.nist.gov/projects/cryptographic-standards-and-guidelines)

## Part 5: Behavioral and Leadership Scenarios

### 97. Tell me about the most technically complex feature you designed and shipped end-to-end in your 5-6 years career. What were the specific bottlenecks?

**How to answer.** Pick one feature where *you* made consequential design decisions, shipped it, and measured the result. “Complex” should come from constraints and tradeoffs—not a list of technologies. In roughly two minutes, use STAR, then be ready to whiteboard the architecture and defend alternatives.

**Customizable STAR outline**

- **Situation (15–20 seconds):** `[business/customer problem]`, old scale/SLA, and why the previous system failed.
- **Task (10 seconds):** your role and decision ownership: “I owned `[scope]`, working with `[teams]`, with `[deadline/constraint]`.”
- **Action (60–90 seconds):** requirements, options considered, chosen architecture, the two or three specific bottlenecks, tests/rollout/observability, and how you influenced others.
- **Result (20–30 seconds):** quantified before/after metrics, business outcome, incidents avoided, and one lesson or later improvement.

**Illustrative sample only—replace it with your real experience and numbers:**

> Our checkout flow synchronously called inventory, payment, and notification services. At `[peak traffic]`, provider latency caused thread-pool exhaustion, duplicate retries, and a p99 of `[before]`; the launch target was `[SLO/throughput]`. I owned the workflow redesign from RFC through rollout, with three service teams and SRE.
>
> I first converted assumptions into a load model and traced the critical path. The major bottlenecks were `[for example: a database hot row]`, `[for example: synchronous provider calls holding connections]`, and `[for example: retry amplification]`. We considered `[2PC/direct calls/event workflow]`; I chose `[your actual choice]` because `[tradeoff tied to invariant]`. I implemented local transactions plus an outbox, idempotency keys and conditional state transitions, partitioned consumers by `[key]`, and bounded concurrency per provider. We added circuit breakers, exponential backoff with jitter, DLQ/reconciliation tooling, and dashboards for queue age, duplicate suppression, provider latency, and business success. A shadow run and `[5% -> 25% -> 100%]` canary let us compare behavior, and the database migration used expand-contract so rollback remained possible.
>
> The result was `[p99 before -> after]`, `[throughput increase]`, `[error/duplicate reduction]`, and `[business result]`. One initial partition key still concentrated `[tenant/event]` traffic; we detected it from per-partition lag and changed to `[bucketing/virtual partitions]`. That taught me to capacity-test skew, not only uniform traffic.

The bottleneck explanation should be mechanical: “threads waited on a 2-second dependency, so at 500 requests/s Little's Law implied about 1,000 concurrent waits against a 200-thread pool,” or “all writes used the current date as a range key, so one partition received 70% of traffic.” Explain evidence—profiles, traces, `EXPLAIN`, heap dumps, queue lag—not intuition alone.

**Prompts to personalize:** What did *you* decide versus the team? What was the rejected alternative and why? Which constraint was correctness, latency, cost, compliance, or deadline? What numbers can you safely disclose (percentages are fine)? What failed during testing? How did you roll back? What would you change now?

Avoid claiming the whole team's work, saying only “we used Kafka/Kubernetes,” inventing exact numbers, or presenting a flawless story. A credible senior answer includes one discovered mistake, the feedback loop, and a measurable outcome.

### 98. Describe a scenario where you had a major technical disagreement with a Senior Architect or Tech Lead. How did you structure your data to resolve it?

**How to answer.** The interviewer is testing judgment, influence without authority, listening, and whether you optimize for the product rather than “winning.” Choose a genuine disagreement with material tradeoffs and a professional resolution. Make the decision data reproducible and show that you could commit to the final choice even if it was not yours.

**Customizable STAR structure**

- **Situation:** decision, stakes, people involved, and shared goal.
- **Task:** your responsibility and the decision deadline.
- **Action:** restate the other position fairly; define decision criteria; gather production evidence; run a bounded experiment; document results/risks; decide through the agreed mechanism.
- **Result:** decision and measured outcome, relationship/team impact, and what later evidence changed.

**Illustrative sample only—do not present it as personal history:**

> We disagreed about whether `[high-traffic read path]` should remain a synchronous database query or add `[cache/materialized view/new service]`. The architect favored the new layer for scale; I was concerned about invalidation and operational complexity. We agreed that the shared goal was `[p99 SLO]` under `[peak]` while keeping stale data below `[bound]`, not defending either design.
>
> I wrote a one-page decision record with weighted criteria: p99/p99.9 latency, sustainable throughput, correctness/staleness, failure behavior, implementation time, run cost, and on-call burden. I pulled four weeks of traces/query statistics to build representative traffic distributions, including the top-tenant skew. We implemented two small prototypes and ran the same replay/load test. I published raw scripts, environment, confidence intervals, flame graphs, and failure-injection results—not just an average-latency chart. I also asked the architect to review the test for assumptions that favored my proposal.
>
> The data showed `[actual result]`: option A met normal load but crossed the SLO under `[condition]`; option B handled peak but served stale data during `[failure]`. We chose `[decision or hybrid]` with explicit guardrails: `[TTL/invalidation/circuit breaker/capacity threshold]`, an owner, and a revisit date. After rollout, `[metric]` changed from `[before]` to `[after]`, and the ADR helped adjacent teams reuse the reasoning. I learned `[real lesson—for example, model skew and failure behavior earlier]`.

Good decision data includes:

```text
Hypothesis and success threshold
Representative workload + skew + dataset
Same hardware/config and reproducible scripts
p50/p95/p99, throughput, errors, resource/cost
Correctness and chaos/failover outcomes
Delivery/operational complexity and reversible path
Decision owner, deadline, dissent, and revisit trigger
```

**Prompts to personalize:** What was the strongest argument for the other side? Did you discover you were partly or fully wrong? Who was the decision owner? What evidence changed a mind? Was the decision reversible? How did you prevent the debate delaying delivery? What happened to trust afterward?

Avoid describing the senior person as incompetent, escalating before trying alignment, cherry-picking a benchmark, or saying “I convinced them” without shared criteria. If the lead chose the other option, a strong ending is: “I documented the risk, committed to the decision, added monitoring, and we revisited when the agreed trigger occurred.”

### 99. Tell me about a time a critical system crashed in a production environment under your watch. Walk through your immediate debugging, mitigation, and post-mortem analysis.

**How to answer.** Use a real incident and separate **stabilization**, **diagnosis**, and **prevention**. Demonstrate calm incident command, customer communication, reversible mitigation, evidence-based debugging, and blameless follow-through. Do not expose confidential customer/security details.

**Customizable incident timeline**

```text
T+0–5 min: acknowledge page, declare severity, open incident channel,
           assign incident commander/comms/operations/scribe, freeze deploys
T+5–15:   assess blast radius/SLO, compare recent changes, mitigate safely
T+15–__:  validate recovery; preserve logs/metrics/traces/dumps; test hypotheses
After:    reconcile data, communicate closure, write postmortem and track actions
```

**Illustrative STAR sample only—replace every fact with your incident:**

> At `[time/context]`, alerts showed `[user-visible symptom]` across `[scope]`; error rate reached `[x]` and `[business operation]` was affected. I was `[on-call/feature owner]`. I declared a `[severity]` incident, established one incident commander, assigned a communications owner and scribe, and paused deployments so responders did not make conflicting changes.
>
> Our first goal was reducing harm, not proving a root cause. The golden-signal dashboard and release markers showed `[evidence]`, so we `[rolled back/disabled a flag/shifted traffic/reduced concurrency]`, a known reversible action. We verified recovery through both telemetry and a synthetic/customer journey. In parallel, we preserved `[thread/heap dump, logs, traces, DB plans]`. A time-correlated comparison showed `[mechanism—for example, retries multiplied traffic after a provider slowdown, exhausting the DB pool]`; controlled reproduction/failure injection confirmed it. We then capped concurrency, disabled unsafe retries, and reconciled ambiguous `[orders/messages]` using idempotency records rather than replaying blindly.
>
> Service recovered in `[MTTM]` and fully stabilized in `[MTTR]`; impact was `[honest quantified impact]`. I facilitated a blameless postmortem with a timeline, contributing conditions, detection/response gaps, and why safeguards did not catch it. Actions had owners and dates: `[code fix]`, `[canary/load/chaos test]`, `[SLO alert/runbook]`, and `[architectural change]`. We tracked completion and demonstrated `[subsequent drill/no recurrence/metric improvement]`. My key lesson was `[specific change in operating practice]`.

The root cause should explain a causal chain, not “human error”:

```text
trigger -> latent condition -> resource/invariant failure -> retry/fan-out amplifier
        -> user impact
controls expected to stop it -> why each failed or was absent
```

**Prompts to personalize:** What was the first reliable symptom? What did you do in the first five minutes? Which mitigation was reversible and why chosen? What hypotheses did evidence reject? Was data inconsistent? How did you communicate status? What action did *you* own after the review? What measurable prevention proves learning?

Avoid narrating frantic command-by-command debugging, blaming a deployer, changing multiple variables at once, or claiming “zero impact” without evidence. If you caused the trigger, say so plainly while focusing on systemic guardrails. If you were not incident commander, accurately state your role and decisions.

### 100. How do you handle managing technical debt while product owners are aggressively pushing for rapid business feature delivery schedules? Provide an example.

**Interview-ready summary.** Treat debt as an economic/risk portfolio, not a moral complaint or an invisible cleanup backlog. Make its interest measurable, distinguish urgent risk from aesthetic preference, give product owners options with impact, reserve capacity, and attach debt retirement to the feature paths that create or suffer from it. Escalate only when reliability, security, compliance, or data integrity crosses an explicit threshold.

Classify and quantify debt:

- **Critical:** known security/compliance/data-loss risk—must block or receive explicit accountable risk acceptance.
- **Reliability/performance:** incidents, error-budget burn, latency/capacity ceiling.
- **Delivery friction:** build time, flaky tests, manual releases, change lead time.
- **Maintainability:** duplication/obsolete components with evidence of change cost.

For each item record owner, affected capability, probability/impact, “interest” (hours/incidents/cloud cost), remediation size, dependencies, and revisit trigger. Offer options rather than a binary argument: full fix now, bounded enabling refactor, isolate behind an interface/flag with a dated follow-up, or consciously accept risk. Do not hide refactoring inside estimates; explain it as work needed to deliver safely.

**Illustrative STAR sample only—replace it with your real example:**

> Product needed `[feature]` by `[date]`, but the affected `[module/pipeline]` had `[measured debt: deployment failures, 40-minute builds, incident rate, capacity ceiling]`. A full rewrite would take `[estimate]` and miss the market window; adding the feature unchanged would raise `[specific risk]`.
>
> I mapped the dependency path and proposed three costed options. We agreed on a thin-slice plan: spend `[bounded time/capacity]` extracting `[seam]`, add characterization/contract tests around current behavior, implement the feature behind a flag, and instrument `[risk metric]`. I created a debt item with an owner, acceptance criteria, and trigger—`[for example, before traffic exceeds X or by date Y]`—and showed it in the same product roadmap, not a hidden engineering board. We released the business slice on `[outcome]`, while the seam reduced subsequent change time from `[before]` to `[after]`. We retired the remaining debt over `[iterations]`; `[incidents/build failures/cost]` fell by `[real measure]`.

A sustainable operating model combines:

1. A small, explicit capacity budget (often adjusted by current error-budget health, not a magical fixed percentage).
2. “Boy Scout” cleanup only for safe local improvements; architectural changes get tests/review.
3. Debt acceptance criteria in feature definitions and post-incident actions tracked like product commitments.
4. Quarterly portfolio review to delete stale items and rank by risk/interest.
5. Fitness functions—build time, dependency age, coverage/contract checks, complexity boundaries, SLOs—so debt becomes visible early.

**Prompts to personalize:** Which number made product care? What business deadline was real? What did you deliberately *not* fix? How was risk accepted? Did the follow-up actually ship—what mechanism ensured it? What was the measurable delivery/reliability outcome? When have you said no because the risk was unacceptable?

Avoid “20% tech debt” as an unsupported slogan, calling every disliked code path debt, proposing rewrites without incremental value, or promising to clean it “later” without owner/date/trigger. A senior answer shows joint prioritization: product owns value and timing, engineering makes technical risk legible, and both agree on a reversible, observable plan.
