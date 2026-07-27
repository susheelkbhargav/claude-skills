# Chapter 11: Concurrency (Items 78–84)

## Core Idea
Correct concurrent code requires *both* mutual exclusion and inter-thread communication (visibility). Prefer high-level `java.util.concurrent` utilities over raw `wait`/`notify` and threads; keep shared mutable state minimal and well-documented.

## Frameworks Introduced
- **Item 78 — Synchronize access to shared mutable data**: synchronization guarantees *mutual exclusion* AND *visibility* (memory effects). Reading/writing shared mutable data without it is unsafe even for atomic operations (no visibility guarantee). Both readers and writers must synchronize. Alternatively use `volatile` (visibility only) or immutability (no sharing needed).
- **Item 79 — Avoid excessive synchronization**: never call an *alien method* (client-provided callback, overridable method) while holding a lock — risk of deadlock, corruption, or reentrancy surprises. Do as little as possible inside `synchronized` blocks; snapshot then act outside the lock. Prefer `CopyOnWriteArrayList` for observer lists.
- **Item 80 — Prefer executors, tasks, and streams to threads**: use the `Executor` framework (`ExecutorService`, `ThreadPoolExecutor`, `ForkJoinPool`) instead of managing `Thread`s by hand. Submit `Runnable`/`Callable` tasks; let the framework manage lifecycle and pooling.
- **Item 81 — Prefer concurrency utilities to `wait` and `notify`**: `wait`/`notify` are error-prone (missed signals, spurious wakeups). Use higher-level tools: `ConcurrentHashMap`, `BlockingQueue`, `CountDownLatch`, `Semaphore`, `CyclicBarrier`. If you must use `wait`, always in a `while` loop testing the condition.
- **Item 82 — Document thread safety**: state each class's thread-safety level in its doc — **immutable**, **unconditionally thread-safe**, **conditionally thread-safe** (some sequences need external sync), **not thread-safe**, or **thread-hostile**. Don't rely on `synchronized` in the signature (it's an implementation detail); use a private lock object for unconditional thread safety.
- **Item 83 — Use lazy initialization judiciously**: usually initialize eagerly. If lazy: for instance fields use the *double-check idiom* (with a `volatile` field); for static fields use the *lazy initialization holder class idiom* (`private static class Holder`); for repeatable initialization use the single-check idiom.
- **Item 84 — Don't depend on the thread scheduler**: correctness must not hinge on thread priorities or scheduler timing. Don't busy-wait; don't "fix" races with `Thread.yield`/priorities/`sleep`. Keep the runnable-thread count low.

## Key Concepts
- **Mutual exclusion**: only one thread in a critical section at a time.
- **Visibility / memory effects**: guarantee that one thread sees another's writes.
- **`volatile`**: guarantees visibility (and ordering) but *not* atomicity of compound ops.
- **Alien method**: a method you don't control (callback/override) — dangerous under a lock.
- **Liveness failures**: deadlock, livelock, starvation.
- **Safe publication**: making an object visible to other threads correctly (via final field, volatile, synchronized, or concurrent collection).

## Mental Models
- Synchronization is *not just locking* — it's also a memory barrier. Both reader and writer must synchronize to communicate.
- "Do less inside the lock": gather what you need, release, then do the slow/alien work.
- Reach for `java.util.concurrent` first; hand-rolled `wait`/`notify` and raw threads are last resorts.
- Immutable objects are inherently thread-safe and freely shareable — the cheapest concurrency strategy.
- If correctness depends on timing/priorities, it's *already broken* — just latently.

## Anti-patterns
- **Unsynchronized shared mutable data** (even a `boolean` stop flag): the reader may loop forever (no visibility) — use `volatile` or synchronize.
- **`while(!stopRequested) i++;` with plain field**: JVM may hoist the read → infinite loop.
- **Calling alien methods under a lock**: deadlock / `ConcurrentModificationException` / reentrancy corruption.
- **Managing raw `Thread`s** and `Thread.run()` instead of executors.
- **`wait` outside a `while` loop / raw `notify`**: missed and spurious wakeups.
- **Relying on priorities/`yield`/`sleep`** to order threads: non-portable, fragile.
- **`volatile` for compound actions** (`count++`): not atomic → use `AtomicLong`/synchronization.

## Code Examples
```java
// Item 78 — visibility: volatile fixes the "stop flag never seen" bug
private static volatile boolean stopRequested;
// writer: stopRequested = true;
// reader loop terminates because volatile guarantees the write is visible

// Item 78 — atomicity needs more than volatile
private static final AtomicLong nextSerial = new AtomicLong();
public static long generateSerialNumber(){ return nextSerial.getAndIncrement(); } // atomic
```
- **What it demonstrates**: `volatile` for visibility; `Atomic*` (or synchronization) for compound atomicity.

```java
// Item 83 — lazy initialization holder class idiom (static field)
private static class FieldHolder { static final FieldType field = computeFieldValue(); }
static FieldType getField(){ return FieldHolder.field; } // JVM initializes Holder on first use, no sync cost

// Item 83 — double-check idiom (instance field)
private volatile FieldType field;
FieldType getField() {
    FieldType result = field;
    if (result == null) synchronized(this) {
        if (field == null) field = result = computeFieldValue();
    }
    return result;
}
```

## Reference Tables
| Need | Tool | Item |
|---|---|---|
| Visibility only | `volatile` | 78 |
| Atomic compound op | `synchronized` / `Atomic*` | 78 |
| Task execution | `ExecutorService` | 80 |
| Producer/consumer | `BlockingQueue` | 81 |
| Wait for N events | `CountDownLatch` | 81 |
| Concurrent map | `ConcurrentHashMap` | 81 |
| Lazy static field | holder class idiom | 83 |
| Lazy instance field | double-check (`volatile`) | 83 |

## Worked Example
The alien-method deadlock (Item 79). An observable set holds a list of observers and, inside a `synchronized` block, notifies each on add. One observer's callback, while being notified, calls back into the set to *remove itself* — re-entering the lock the current thread already holds, mutating the list mid-iteration → `ConcurrentModificationException`. A background-thread variant deadlocks instead. Fix: snapshot the observers, then notify *outside* the lock:
```java
private void notifyElementAdded(E element) {
    List<SetObserver<E>> snapshot;
    synchronized (observers) { snapshot = new ArrayList<>(observers); } // copy under lock
    for (SetObserver<E> observer : snapshot) observer.added(this, element); // alien calls OUTSIDE lock
}
// Or use CopyOnWriteArrayList and skip the manual snapshot entirely.
```
Rule: never yield control to a method you don't control while holding a lock.

## Key Takeaways
1. Synchronize both reads and writes of shared mutable data — for exclusion *and* visibility.
2. `volatile` gives visibility, not atomicity; use `Atomic*`/synchronization for compound ops.
3. Do minimal work inside locks; never call alien methods while holding one.
4. Prefer executors/tasks over raw threads, and `java.util.concurrent` utilities over `wait`/`notify`.
5. Document each class's thread-safety level; use a private lock for unconditional safety.
6. Initialize eagerly unless proven otherwise; never depend on the scheduler/priorities for correctness.

## Connects To
- **Ch4 (Item 17)**: immutable objects are automatically thread-safe.
- **Ch2 (Item 3)**: enum singletons are thread-safe by construction.
- **Ch7 (Item 48)**: parallel streams sit on the fork/join framework.
- **Ch9 (Item 59)**: `java.util.concurrent` is "know the libraries" applied to threads.
