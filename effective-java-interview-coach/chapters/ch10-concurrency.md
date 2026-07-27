# Chapter 11: Concurrency

## Item 78: Synchronize access to shared mutable data

### Opening scenario
"Here's a class with a `private static boolean stopRequested` field. One thread sets it to `true` to stop a worker loop running in another thread. No exceptions are thrown, no compiler warnings — but the worker loop never terminates; it spins forever. The code has no synchronization errors in the traditional sense. Walk me through what's happening."

### Follow-up probes
- "The field is a `boolean` — reads and writes of it are atomic per the JLS. So why isn't that enough?"
- "Why not just make the field `volatile` instead of adding `synchronized` methods? What's the tradeoff?"
- "Now suppose instead of a stop flag, I have a serial-number generator using `nextSerialNumber++` on a `volatile int`. Is that thread-safe? Why not?"
- "What's the actual difference between synchronization used for mutual exclusion and synchronization used purely for communication?"
- "What does 'hoisting' mean here, and why is the JVM allowed to do it?"
- "If I don't want any synchronization overhead at all, what's my best option for something like a counter?"

### Naive attempt
```java
// Broken! - How long would you expect this program to run?
public class StopThread {
    private static boolean stopRequested;

    public static void main(String[] args) throws InterruptedException {
        Thread backgroundThread = new Thread(() -> {
            int i = 0;
            while (!stopRequested)
                i++;
        });
        backgroundThread.start();
        TimeUnit.SECONDS.sleep(1);
        stopRequested = true;
    }
}
```
The candidate reasons: "boolean reads/writes are atomic in Java, so there's no data race — this should work."

### What breaks
This is a **liveness failure**, not a safety failure: the program never terminates. The Java Memory Model (JLS 17.4) gives no guarantee about *when*, or even *whether*, a write made by one thread becomes visible to another thread in the absence of synchronization. Atomicity of the individual read/write is irrelevant — the problem is visibility. In the absence of synchronization, the JVM is free to hoist the read of `stopRequested` out of the loop, transforming:
```java
while (!stopRequested) i++;
```
into the semantic equivalent of:
```java
if (!stopRequested) while (true) i++;
```
This is exactly what the OpenJDK Server VM does with this code, per the book — the background thread caches the field's value in a register and never re-reads it from main memory. Interviewers should also probe the companion bug: `generateSerialNumber` using `nextSerialNumber++` on a `volatile int` is a **safety failure**, not a liveness failure — `++` is read-modify-write, not atomic, so two threads can race and return the same serial number even with `volatile`, because `volatile` gives visibility/ordering but not atomicity of compound operations.

### The recommended approach & why
Three fixes, escalating in nuance:

1. **Synchronize both the read and the write.** It is not sufficient to synchronize only the write method — synchronization must guard both sides or it isn't guaranteed to work at all (occasionally appears to work on some machines, which is worse than an outright failure):
```java
// Properly synchronized cooperative thread termination
public class StopThread {
    private static boolean stopRequested;
    private static synchronized void requestStop() { stopRequested = true; }
    private static synchronized boolean stopRequested() { return stopRequested; }

    public static void main(String[] args) throws InterruptedException {
        Thread backgroundThread = new Thread(() -> {
            int i = 0;
            while (!stopRequested())
                i++;
        });
        backgroundThread.start();
        TimeUnit.SECONDS.sleep(1);
        requestStop();
    }
}
```
Here the actions would be atomic even without synchronization — the sync is used **solely for its communication effect**, not mutual exclusion.

2. **Use `volatile` when you need communication but not atomicity.** `volatile` performs no mutual exclusion, but guarantees any reading thread sees the most recently written value — cheaper and less verbose than a monitor lock for this exact case:
```java
private static volatile boolean stopRequested;
```

3. **Use `java.util.concurrent.atomic` (e.g. `AtomicLong`) when you need atomicity, not just visibility** — as in the serial-number case. `volatile` alone is broken there because `++` is compound; `AtomicLong.getAndIncrement()` provides lock-free atomicity and outperforms a `synchronized` version:
```java
private static final AtomicLong nextSerialNum = new AtomicLong();
public static long generateSerialNumber() { return nextSerialNum.getAndIncrement(); }
```

The deepest mechanism: the JLS guarantees atomicity of reads/writes to all types except `long`/`double`, but atomicity of the individual operation says nothing about **cross-thread visibility** — that's a separate guarantee, governed by the memory model, that only synchronization (monitor locks, `volatile`, or `java.util.concurrent` primitives) provides. The best policy overall is to avoid sharing mutable data at all: share immutable data, confine mutable data to one thread, or use **safe publication** (static field at class-init time, `volatile`/`final` field, normal locking, or a concurrent collection) to hand off an "effectively immutable" object once, after which other threads may read it without further synchronization.

### Panel-ready answer checklist
- Names the actual guarantee: atomicity of primitive reads/writes (except `long`/`double`) is separate from cross-thread visibility, which requires synchronization/`volatile`/JMM happens-before edges.
- Explains hoisting/register-caching as the concrete mechanism producing the observed spin-forever symptom, not just "it's undefined behavior."
- Correctly classifies the failure as a liveness failure (never-terminates) vs. distinguishes it from a safety failure (wrong-value, e.g. the serial-number race from `++` not being atomic).
- States that synchronizing only the write (or only the read) is insufficient — both sides must synchronize.
- Chooses correctly between `synchronized` (mutual exclusion + communication), `volatile` (communication only, no atomicity), and `java.util.concurrent.atomic` (atomicity + lock-free) based on whether atomicity is required.
- Mentions the "don't share mutable data" / safe-publication alternative as the strongest structural fix, not just a locking fix.

---

## Item 79: Avoid excessive synchronization

### Opening scenario
"You have an `ObservableSet<E>` that wraps a `Set` and notifies registered `SetObserver` callbacks (via `synchronized` blocks around a `List<SetObserver<E>>`) whenever an element is added. A test adds integers 0 through 99. One observer, on seeing the value 23, calls `removeObserver(this)` to unsubscribe itself from inside its own `added()` callback. The program throws `ConcurrentModificationException`. Why, given that the whole thing is wrapped in `synchronized(observers)`?"

### Follow-up probes
- "The synchronized block should prevent concurrent modification — so why does removing from the list *during* iteration still blow up? Is this even a concurrency bug?"
- "Now: instead of calling `removeObserver` directly, the observer hands the removal off to a background thread via an executor and blocks on `.get()`. What happens now, and why is it worse?"
- "Reentrant locks are usually described as a convenience feature. How can reentrancy actively turn a liveness bug into a silent safety bug?"
- "What's an 'alien method' and why does the class have no way to defend against what it does?"
- "What's an 'open call', and why does moving a call outside a synchronized block fix both the exception and the deadlock?"
- "Why is `CopyOnWriteArrayList` a better fix here than taking a manual snapshot with `new ArrayList<>(observers)`?"
- "In a multicore world, what is the 'real cost' of oversynchronization, if it isn't CPU time spent acquiring locks?"

### Naive attempt
```java
// Broken - invokes alien method from synchronized block!
private void notifyElementAdded(E element) {
    synchronized(observers) {
        for (SetObserver<E> observer : observers)
            observer.added(this, element);
    }
}
```
The candidate reasons: "It's wrapped in `synchronized`, so it's thread-safe — case closed."

### What breaks
Two distinct, escalating failures, both caused by invoking an **alien method** (a method neither owned nor controlled by this class — here, the client-supplied `SetObserver.added`) from inside a synchronized region:

1. **`ConcurrentModificationException` (same thread, no other thread involved at all).** The observer's `added()` callback calls `s.removeObserver(this)`, which calls `observers.remove(...)` — mutating the very list `notifyElementAdded` is mid-iteration over. This isn't even a data race; it's the calling thread re-entering the guarded resource through its own reentrant lock while iterating.

2. **Deadlock (different thread).** If the observer instead offloads the removal to a background thread and blocks on `.get()` waiting for it, the background thread tries to acquire the lock on `observers` but can't — the main thread already holds it, and won't release it until the background thread (which it's now blocking on) finishes. Classic deadlock.

The deeper mechanism, per the book: because locks in Java are **reentrant**, calling back into your own class from an alien method invoked in a synchronized region does *not* deadlock — the calling thread just reacquires its own lock and proceeds, even though a conceptually unrelated operation is mid-flight on data the lock is supposed to be protecting. If the invariant was temporarily broken at that moment, this silently corrupts state instead of failing loudly. **Reentrant locks turn what would be a liveness failure (deadlock) into a much worse safety failure (silent data corruption)** — you don't get an exception at all, you just get wrong behavior.

### The recommended approach & why
Never cede control to the client from within a synchronized region — move alien method invocations outside the lock, producing an **open call**. First fix, take a defensive snapshot:
```java
// Alien method moved outside of synchronized block - open calls
private void notifyElementAdded(E element) {
    List<SetObserver<E>> snapshot = null;
    synchronized(observers) {
        snapshot = new ArrayList<>(observers);
    }
    for (SetObserver<E> observer : snapshot)
        observer.added(this, element);
}
```
Better: use a **concurrent collection** designed for this exact access pattern — `CopyOnWriteArrayList`, whose every mutation copies the underlying array, so iteration requires no locking at all:
```java
// Thread-safe observable set with CopyOnWriteArrayList
private final List<SetObserver<E>> observers = new CopyOnWriteArrayList<>();
public void addObserver(SetObserver<E> observer) { observers.add(observer); }
public boolean removeObserver(SetObserver<E> observer) { return observers.remove(observer); }
private void notifyElementAdded(E element) {
    for (SetObserver<E> observer : observers)
        observer.added(this, element);
}
```
This performs poorly for write-heavy lists, but observer lists are rarely modified and often traversed — a perfect fit.

Beyond correctness, there's a performance argument: keep the amount of work done inside a synchronized region to a minimum — acquire the lock, examine/transform shared state, release. In a multicore world, the real cost of oversynchronization isn't CPU cycles spent acquiring a lock, it's **contention**: lost parallelism opportunities and cross-core cache-coherency traffic needed to keep every core's view of memory consistent, plus a hidden cost — oversynchronization can limit the VM's ability to optimize code.

Design guidance for class authors: when writing a mutable class, choose between (a) no internal synchronization, requiring the client to synchronize externally (`java.util` collections), or (b) internal synchronization making the class thread-safe (`java.util.concurrent` collections) — choose (b) only if it buys *significantly* higher concurrency than external client-side locking would, since otherwise you're paying oversynchronization cost for nothing (see: `StringBuffer` → `StringBuilder`, synchronized `Random` → `ThreadLocalRandom`). When in doubt, don't synchronize — but document that the class isn't thread-safe (Item 82). One hard exception: a method that modifies a **static field** reachable by multiple threads must synchronize internally regardless, because unrelated clients have no way to externally synchronize on a field they don't jointly coordinate over.

### Panel-ready answer checklist
- Identifies "alien method invoked from a synchronized region" as the root cause, not "missing synchronization" — the code was already synchronized.
- Distinguishes the two failure modes produced by the same root cause: same-thread reentrant callback → `ConcurrentModificationException`; cross-thread callback → deadlock.
- Explains reentrancy correctly: it prevents self-deadlock but can silently let a thread observe/mutate an inconsistent state guarded by a lock it already holds — a safety failure disguised as "it didn't deadlock, so it's fine."
- Defines "open call" and explains why moving the alien call outside the lock fixes both failure modes simultaneously.
- Names `CopyOnWriteArrayList` (or equivalent concurrent collection) as superior to a manual snapshot-and-copy, and justifies it by the read-heavy/write-rare access pattern.
- Articulates the multicore contention argument for why "just synchronize everything" is not free, distinguishing lock-acquisition cost from contention cost.

---

## Item 80: Prefer executors, tasks, and streams to threads

### Opening scenario
"Someone on your team hand-rolls a background work queue: a class with a method to enqueue `Runnable`s and a shutdown method that lets in-flight work finish before the background thread exits. It's about a page of code. What's your review comment?"

### Follow-up probes
- "What's the actual abstraction difference between a raw `Thread` and the Executor Framework's task/executor split?"
- "When would you reach for `newCachedThreadPool` vs. `newFixedThreadPool` — what goes wrong with a cached pool on a heavily loaded server?"
- "What's a fork-join pool, and how do parallel streams relate to it?"
- "If I need to wait for a single submitted task's result, what do I call? What if I need all of a batch to finish?"
- "Why would forgetting to call `shutdown()` on an `ExecutorService` keep your JVM from exiting?"

### Naive attempt
A candidate proposes writing (or defends an existing) hand-rolled `Thread`-based work queue with manual start/stop/graceful-shutdown logic, treating `Thread` as the natural unit of both "what to run" and "how to run it."

### What breaks
Hand-written thread management of this kind is "little more than a toy" even when it looks complete — it requires a full page of subtle, delicate code prone to safety and liveness failures if not implemented exactly right. The conceptual bug is conflating the **unit of work** with the **execution mechanism**: a raw `Thread` serves as both, so every new execution policy (pooling, scheduling, bounded queues, rejection handling) has to be hand-coded again. On a heavily loaded server, a naively chosen `newCachedThreadPool` compounds this: tasks aren't queued, they're handed to a thread immediately, and if none is idle a new one is created — under CPU saturation this just creates more threads competing for the already-saturated CPUs, making things worse, not better.

### The recommended approach & why
Since `java.util.concurrent`, creating a robust work queue is one line:
```java
ExecutorService exec = Executors.newSingleThreadExecutor();
exec.execute(runnable);
exec.shutdown(); // omitting this can prevent the VM from exiting
```
The Executor Framework separates the **task** (`Runnable`, or `Callable` for a task that returns a value / throws checked exceptions) from the **executor service** that runs it — mirroring what the Collections Framework did for aggregation. This buys: waiting on one task (`get`), waiting on a batch (`invokeAll`/`invokeAny`), waiting for full shutdown (`awaitTermination`), pulling results as they complete (`ExecutorCompletionService`), and scheduled/periodic execution (`ScheduledThreadPoolExecutor`) — all without hand-written synchronization.

Sizing guidance: `Executors.newCachedThreadPool` is fine for small programs/lightly loaded servers (no configuration, generally "does the right thing"), but for a heavily loaded production server prefer `Executors.newFixedThreadPool` (bounded thread count) or configure `ThreadPoolExecutor` directly for full control.

Since Java 7, the framework extends to **fork-join tasks** run on a `ForkJoinPool`: threads in the pool don't just process their own subtasks, they *steal* work from one another to keep every thread busy, improving CPU utilization, throughput, and latency. Parallel streams (Item 48) are built atop fork-join pools, giving you these benefits with little manual effort when the task is a good fit for data-parallelism.

### Panel-ready answer checklist
- States the core abstraction shift explicitly: unit-of-work (task: `Runnable`/`Callable`) decoupled from execution mechanism (executor service), not just "use the library instead of rolling your own."
- Can name the concrete API calls for common needs: submit-and-forget (`execute`), wait-for-one (`get`), wait-for-many (`invokeAll`/`invokeAny`), scheduled (`ScheduledThreadPoolExecutor`).
- Explains precisely why `newCachedThreadPool` is dangerous under heavy load (unbounded thread creation under saturation) vs. `newFixedThreadPool`'s bounded behavior.
- Connects fork-join pools to work-stealing and to parallel streams as the higher-level, low-effort entry point.
- Knows that forgetting `shutdown()` can leave non-daemon threads alive and prevent JVM exit.

---

## Item 81: Prefer concurrency utilities to wait and notify

### Opening scenario
"You need one thread to wait until N worker threads have all finished a startup phase before proceeding — the classic 'wait for peers, then fire the starting gun' pattern. A candidate proposes implementing this directly with `wait`/`notifyAll` on a shared counter object. What's your first question?"

### Follow-up probes
- "What exactly can go wrong if you call `wait()` outside of a loop, or outside a `synchronized` block?"
- "Why must you re-check the condition *after* waking from `wait()` — isn't a notification proof the condition is now true?"
- "Give me two distinct reasons a thread might wake from `wait()` even though nobody who caused the condition to become true, called notify."
- "When would you choose `notify()` over `notifyAll()`, and what has to be true about your waiters for that to be safe?"
- "What's `putIfAbsent` actually buying you over separate `containsKey`/`put` calls on a `ConcurrentHashMap`, given the map is already thread-safe?"
- "Why can't you just wrap a `ConcurrentHashMap` in `Collections.synchronizedMap` and lock on it to get atomic compound operations?"

### Naive attempt
```java
// Candidate's hand-rolled coordination attempt
synchronized (lock) {
    if (!conditionHolds())
        lock.wait();
    // proceed
}
```
Reasoning: "I called `wait()` inside a `synchronized` block, so this is correct."

### What breaks
Two separate defects, both classic `wait`/`notify` traps:

1. **`if` instead of `while`** — testing the condition only once before waiting, and trusting it unconditionally after waking, breaks **safety**. A thread can wake from `wait()` with the condition still false for several reasons: another thread grabbed the lock and changed state between the `notify` and this thread's wakeup; another thread called `notify`/`notifyAll` accidentally or maliciously on a publicly accessible lock object; the notifying thread was "generous" and called `notifyAll` when only some waiters' conditions were satisfied; or a **spurious wakeup** occurred with no `notify` at all. Proceeding on a false condition can destroy the invariant the lock was meant to protect.

2. Skipping the pre-wait check (testing the condition only after entering the loop, not before waiting) breaks **liveness** — if the condition already holds and `notify` was already called before this thread started waiting, there's no guarantee it will ever wake up.

The correct low-level idiom (still worth knowing, for legacy code) is:
```java
// The standard idiom for using the wait method
synchronized (obj) {
    while (<condition does not hold>)
        obj.wait(); // (Releases lock, and reacquires on wakeup)
    ... // Perform action appropriate to condition
}
```

### The recommended approach & why
Since Java 5, prefer the higher-level `java.util.concurrent` utilities over hand-coded `wait`/`notify` — described in the book as the difference between "concurrency assembly language" and a proper high-level language. Three families:

- **Executor Framework** (Item 80).
- **Concurrent collections** — high-performance implementations (e.g. `ConcurrentHashMap`, `CopyOnWriteArrayList`) that manage their own internal synchronization, so external locking on them only slows the program down and cannot exclude concurrent activity. Because you can't lock out concurrent activity, these interfaces expose **state-dependent atomic compound operations** like `putIfAbsent`, which combine multiple primitives into one atomic step — impossible to build correctly by composing separate `get`/`put` calls externally:
```java
// Concurrent canonicalizing map atop ConcurrentMap - faster!
public static String intern(String s) {
    String result = map.get(s);
    if (result == null) {
        result = map.putIfAbsent(s, s);
        if (result == null)
            result = s;
    }
    return result;
}
```
  Prefer `ConcurrentHashMap` over `Collections.synchronizedMap` outright — the book reports the `intern` method above as over six times faster than `String.intern`.
- **Synchronizers** (`CountDownLatch`, `Semaphore`, `CyclicBarrier`, `Exchanger`, `Phaser`) let threads wait for one another. A `CountDownLatch` is a single-use barrier; `Phaser` is the most powerful/general. The "time concurrent execution" example in the book uses three latches (`ready`, `start`, `done`) to implement a start-gun/finish-line pattern that would be "messy to say the least" atop raw `wait`/`notify` but is straightforward atop `CountDownLatch` — and warns of a **thread-starvation deadlock** if the executor can't supply as many threads as the concurrency level requires.

If you must maintain legacy `wait`/`notify` code: always invoke `wait` inside a `while` loop using the standard idiom, and generally prefer `notifyAll` over `notify` — `notifyAll` is the conservative, always-correct choice since it wakes every waiter and lets each recheck its own condition; `notify` is only a safe optimization when every possible waiter is waiting on the *same* condition and only one waiter at a time can benefit, and even then `notifyAll` guards against an unrelated thread's stray `wait()` swallowing a critical notification meant for someone else.

### Panel-ready answer checklist
- Requires `while`, not `if`, around `wait()`, with a stated reason tied to both safety (recheck after wake) and liveness (check before wait).
- Can list at least two of the four reasons a thread may wake with a false condition (stolen notification, malicious/accidental notify on a public lock, over-generous notifyAll, spurious wakeup).
- Explains why concurrent collections can't be safely wrapped with external locking for compound operations, and why state-dependent methods like `putIfAbsent` exist as a result.
- Distinguishes when `notify` is a valid optimization vs. when `notifyAll` is required for correctness.
- Names at least one synchronizer (`CountDownLatch`, `Semaphore`, etc.) and describes a concrete coordination pattern it solves.
- Frames the overall recommendation as "concurrency utilities over hand-coded wait/notify," not as "avoid wait/notify because it's deprecated" (it isn't deprecated — it's just low-level).

---

## Item 82: Document thread safety

### Opening scenario
"A teammate says: 'I checked the Javadoc and none of this class's methods show the `synchronized` keyword, so it must not be thread-safe — let's wrap every call in external locking to be safe.' What's wrong with that reasoning, and what's wrong with the alternative belief that 'if I see `synchronized` in the signature, it's thread-safe'?"

### Follow-up probes
- "Why doesn't Javadoc even show the `synchronized` modifier in its generated output?"
- "Give me the full taxonomy of thread-safety levels a class can document, in order — where does `ConcurrentHashMap` fall vs. `ArrayList` vs. `String`?"
- "What does 'conditionally thread-safe' mean concretely, and what extra must the docs specify beyond just the word 'conditionally'?"
- "What's a thread-hostile class, and how does one usually come to exist?"
- "If a class exposes its lock publicly (e.g. via `synchronized` instance methods), what attack does that expose it to?"
- "Walk me through the private lock object idiom — why must the lock field be `final`, and why doesn't this idiom work for conditionally thread-safe classes?"

### Naive attempt
"I'll just look for `synchronized` in the method signatures in the generated docs to know if a class is safe to call from multiple threads without extra locking."

### What breaks
This fails on two independent counts. First, mechanically: Javadoc does not include the `synchronized` modifier in its output by design — the presence of `synchronized` is an **implementation detail**, not part of the API contract, so you can't even reliably observe it from outside the source. Second, conceptually: it treats thread safety as **all-or-nothing**, when in fact there's a spectrum, and knowing "this method is internally synchronized" tells a caller nothing about whether they still need external synchronization to compose multiple calls atomically, or whether they might be interfering with a private locking strategy by locking on the instance themselves.

A publicly-lockable class (one that synchronizes on `this` or the object itself, e.g. via plain `synchronized` methods) additionally exposes a **denial-of-service vector**: any client can grab the class's own lock and hold it indefinitely — accidentally or maliciously — freezing out every other caller.

### The recommended approach & why
Document the actual thread-safety level explicitly in prose or via annotation, choosing from (roughly, per *Java Concurrency in Practice*'s `Immutable`/`ThreadSafe`/`NotThreadSafe`):

- **Immutable** — instances appear constant, no external synchronization ever needed (`String`, `Long`, `BigInteger`).
- **Unconditionally thread-safe** — mutable, but sufficient internal synchronization that no external synchronization is ever needed (`AtomicLong`, `ConcurrentHashMap`).
- **Conditionally thread-safe** — like unconditional, except some methods need external synchronization for safe concurrent use; the docs must state *which invocation sequences* require it and *which lock* to acquire (e.g. `Collections.synchronizedMap`'s iterators require the client to synchronize on the returned map itself while iterating over any collection view — not on the view).
- **Not thread-safe** — mutable, every invocation (or invocation sequence) needs client-chosen external synchronization (`ArrayList`, `HashMap`).
- **Thread-hostile** — unsafe even with full external synchronization by every caller, typically from unsynchronized static-data mutation; nobody writes these on purpose, they result from failing to consider concurrency, and get fixed or deprecated once found.

For static factories, document the thread safety of the returned object unless it's obvious from the return type (e.g. `Collections.synchronizedMap`'s doc explicitly spelling out the iteration-locking requirement).

To avoid the public-lock denial-of-service exposure on an unconditionally thread-safe class, encapsulate a **private, final** lock object instead of synchronizing on `this`:
```java
// Private lock object idiom - thwarts denial-of-service attack
private final Object lock = new Object();
public void foo() {
    synchronized(lock) {
        ...
    }
}
```
`final` matters for the same reason as Item 78/17: an accidental reassignment of the lock field would mean different threads synchronizing on different objects, producing catastrophic unsynchronized access with no compiler warning. This idiom applies **only** to unconditionally thread-safe classes — a conditionally thread-safe class must expose which lock clients are meant to acquire for certain sequences, so hiding the lock defeats the whole documentation contract. It's especially valuable for classes designed for inheritance: if a class locked on its own instance, a subclass could unintentionally lock on the same object for an unrelated purpose and the two would "step on each other's toes" (this actually happened with `Thread` itself).

### Panel-ready answer checklist
- States plainly that `synchronized` in source is an implementation detail invisible to Javadoc consumers and is not part of the API contract.
- Recites the taxonomy (immutable / unconditionally thread-safe / conditionally thread-safe / not thread-safe / thread-hostile) with a correct example for at least three levels.
- For "conditionally thread-safe," specifies that the documentation obligation is naming both the invocation sequences requiring external sync *and* the specific lock to acquire.
- Explains the denial-of-service risk of a publicly-lockable class and how a private final lock object mitigates it.
- Correctly restricts the private-lock-object idiom to unconditionally thread-safe classes and explains why it's unsuitable for conditionally thread-safe ones.
- Ties `final` on the lock field back to preventing silent unsynchronized access from a reassigned lock reference.

---

## Item 83: Use lazy initialization judiciously

### Opening scenario
"A static field's initializer is expensive (say, precomputing a large table) and is only needed by a small fraction of instances/call paths. Someone proposes lazily initializing it for performance, using a plain `if (field == null) field = computeFieldValue();` check with no synchronization, on a field accessed from multiple threads. What's your first reaction, and what would you ask before even agreeing lazy init is warranted at all?"

### Follow-up probes
- "Before we even talk about thread safety — how do you know lazy initialization is a net performance win here rather than a net loss?"
- "For an instance field accessed by multiple threads, why can't you just use the naive unsynchronized check above?"
- "Walk me through the lazy initialization holder class idiom for a static field — why is the accessor not synchronized at all, and why is that safe?"
- "Now for an *instance* field — why does the holder-class idiom not apply, and what do you reach for instead?"
- "In the double-check idiom, why is the local variable `result` there — isn't `field` itself enough?"
- "What's the difference between the double-check idiom and the single-check idiom, and when is dropping the second check actually safe?"
- "What does the 'racy single-check idiom' trade away, and why would anyone accept that?"

### Naive attempt
```java
// Broken for multi-threaded access
private FieldType field;
private FieldType getField() {
    if (field == null)
        field = computeFieldValue();
    return field;
}
```
Reasoning: "It only computes the value once, so it should be fine — I'll just guard the null check."

### What breaks
This is unsynchronized shared mutable data (Item 78's failure mode transplanted here): without synchronization there is no guarantee one thread's write to `field` is visible to another, and worse, multiple threads can race past the `null` check simultaneously and each independently call `computeFieldValue()` — at best duplicated work, at worst (if `computeFieldValue` has side effects, or the type isn't safely publishable) actual corruption or an inconsistently observed half-constructed object. Lazy initialization in the presence of multiple threads is "tricky," and every technique the book endorses is deliberately thread-safe by construction — this naive version is not one of them.

### The recommended approach & why
Default position: **normal, eager initialization is preferable to lazy initialization** in most cases — lazy init is an optimization, and per Item 67, "don't do it unless you need to." It trades a cheaper class/instance-init cost for a more expensive per-access cost, and can easily net-negative performance depending on what fraction of instances need the field, how expensive computing it is, and how often it's accessed after that. Measure before committing to it.
```java
// Normal initialization of an instance field
private final FieldType field = computeFieldValue();
```

If lazy init is only needed to break an **initialization circularity**, use the simplest correct form — a synchronized accessor:
```java
// Lazy initialization of instance field - synchronized accessor
private FieldType field;
private synchronized FieldType getField() {
    if (field == null)
        field = computeFieldValue();
    return field;
}
```

If lazy init is needed **for performance on a static field**, use the **lazy initialization holder class idiom**, exploiting the JLS guarantee that a class isn't initialized until used:
```java
// Lazy initialization holder class idiom for static fields
private static class FieldHolder {
    static final FieldType field = computeFieldValue();
}
private static FieldType getField() { return FieldHolder.field; }
```
The accessor performs no synchronization and only a field access — the JVM only synchronizes to perform the one-time class initialization, then patches the generated code so later accesses involve no check or lock at all.

If lazy init is needed **for performance on an instance field**, use the **double-check idiom** — check unsynchronized first, and only take the lock and re-check if the field still looks uninitialized. The field must be `volatile` because there's no locking once it's initialized, so visibility must come from `volatile` alone:
```java
// Double-check idiom for lazy initialization of instance fields
private volatile FieldType field;
private FieldType getField() {
    FieldType result = field;
    if (result == null) {           // First check (no locking)
        synchronized(this) {
            if (field == null)      // Second check (with locking)
                field = result = computeFieldValue();
        }
    }
    return result;
}
```
The local `result` exists to guarantee `field` is read from the volatile location only once in the already-initialized common case — not strictly necessary, but measured at ~1.4x faster than the version without it, and considered more idiomatic for low-level concurrent code. The double-check idiom is not worth applying to static fields — the holder-class idiom is strictly better there.

Two variants for special cases: the **single-check idiom** drops the second check entirely, tolerable only when repeated initialization is harmless (field still `volatile`); and the **racy single-check idiom** additionally drops `volatile` (only valid for primitive types other than `long`/`double`), trading correctness guarantees for speed on some architectures at the cost of possibly re-initializing once per accessing thread — explicitly called out as "exotic," not for everyday use.

### Panel-ready answer checklist
- Leads with "don't do it unless you need to" and names the actual cost tradeoff (cheaper init, more expensive access) rather than assuming lazy is always a win.
- Correctly routes static-field lazy init to the holder-class idiom and instance-field lazy init to the double-check idiom — and can explain *why* the holder-class idiom doesn't apply to instance fields.
- Explains why `field` must be `volatile` in the double-check idiom specifically because there's no locking after initialization.
- Can explain the purpose of the local `result` variable in the double-check idiom.
- Distinguishes double-check / single-check / racy single-check by exactly what guarantee each one gives up (second check, then `volatile` itself) and under what conditions giving it up is acceptable.
- Ties any lazy-init-under-concurrency discussion back to Item 78's visibility/atomicity distinction rather than treating it as a new, unrelated concern.

---

## Item 84: Don't depend on the thread scheduler

### Opening scenario
"A service has hundreds of threads that spend most of their time in a spin loop repeatedly checking a shared flag for a state change, rather than blocking. Throughput is far worse than an equivalent design using a proper synchronizer, and it varies wildly between environments. What's going on, and why might 'just add `Thread.yield()` calls' or 'raise this thread's priority' seem to fix it locally but not solve the actual problem?"

### Follow-up probes
- "What's the actual design goal you're aiming for — 'more threads' or something else?"
- "Why specifically is busy-waiting bad beyond 'it wastes CPU' — what does it do to portability?"
- "If a program 'barely works' because some threads starve for CPU time relative to others, why is `Thread.yield()` the wrong fix?"
- "Same question for bumping thread priorities — when, if ever, is that legitimate?"
- "How does this connect to sizing thread pools under the Executor Framework?"
- "What's the distinction between 'number of threads' and 'number of runnable threads,' and why does only the latter matter here?"

### Naive attempt
```java
// Awful CountDownLatch implementation - busy-waits incessantly!
public void await() {
    while (true) {
        synchronized(this) {
            if (count == 0) return;
        }
    }
}
```
Then, when this performs badly or starves other threads, the candidate's fix is to sprinkle in `Thread.yield()` calls or tweak thread priorities until benchmarks on their machine look acceptable.

### What breaks
This is a correctness-adjacent design flaw dressed as a performance tuning problem. Busy-waiting — repeatedly polling shared state instead of blocking until notified — makes the thread perpetually runnable, so the program's behavior becomes hostage to the thread scheduler's policy, which varies across operating systems, JVMs, and even JVM versions. The book measured this exact `SlowCountDownLatch` at roughly 10x slower than the real `CountDownLatch` with 1,000 threads waiting on a latch. Reaching for `Thread.yield()` "fixes" the symptom only on the JVM/scheduler combination being tested: yield has no testable semantics, and the same calls that help throughput on one JVM can hurt it on another and do nothing on a third. Adjusting thread priorities to solve a genuine liveness problem is worse — it's one of the least portable Java facilities, and doesn't fix the underlying cause, so the starvation problem tends to resurface.

### The recommended approach & why
Design so the **average number of runnable threads is not significantly greater than the number of processors** — this leaves the scheduler with essentially no meaningful choice to make: it just runs whatever's runnable until it isn't, so behavior stays consistent across wildly different scheduling policies. Critically, "number of threads" and "number of *runnable* threads" are not the same measure — threads that are blocked/waiting don't count against this budget, only the ones actually contending for CPU. The concrete technique: give each thread a chunk of useful work, then have it **wait** (block) for more, rather than polling. In Executor Framework terms, this means sizing thread pools appropriately for the available parallelism and keeping tasks an appropriate length — long enough that per-task dispatch overhead doesn't dominate, short enough to avoid the very over-runnable-thread problem this item warns about.

`Thread.yield` and thread-priority tuning are, at most, hints to the scheduler, not levers with defined semantics — legitimate uses are limited to sparingly improving the quality-of-service of an *already-correct, already-working* program, never as a patch for a program whose correctness or throughput currently depends on scheduler whims.

### Panel-ready answer checklist
- Diagnoses busy-waiting (not "too many threads" per se) as the mechanism forcing scheduler-dependence, and connects it to non-portability across JVMs/OSes.
- States the target design invariant precisely: average runnable-thread count close to the processor count, not total thread count.
- Distinguishes "runnable" from merely "alive/waiting" threads.
- Rejects `Thread.yield()` and priority tuning as fixes for a liveness/starvation bug, citing their lack of testable/portable semantics.
- Identifies the legitimate, narrow use of thread priorities (minor QoS tuning of an already-working program) vs. the illegitimate use (patching a broken one).
- Connects the fix back to blocking synchronizers / proper Executor sizing rather than ad hoc spin-and-yield loops.
