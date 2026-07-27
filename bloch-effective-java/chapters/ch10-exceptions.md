# Chapter 10: Exceptions (Items 69–77)

## Core Idea
Exceptions are for exceptional conditions, not control flow. Choose the right kind (checked vs unchecked), throw at the right abstraction level, carry enough diagnostic info, and never swallow them silently.

## Frameworks Introduced
- **Item 69 — Use exceptions only for exceptional conditions**: never for ordinary control flow (e.g. terminating a loop by catching `ArrayIndexOutOfBounds`). A well-designed API needs no exception-based looping; provide a state-testing method or return an `Optional`/distinguished value.
- **Item 70 — Use checked exceptions for recoverable conditions and runtime exceptions for programming errors**: checked = caller can reasonably recover; unchecked (`RuntimeException`) = precondition violations / programming bugs. Errors are reserved for the JVM. Don't subclass `Error`; don't throw `Throwable` directly.
- **Item 71 — Avoid unnecessary use of checked exceptions**: they burden callers. If recovery is impossible, use unchecked. Turn a checked exception into an `Optional` return or a state-testing-method + unchecked pair when it eases the API.
- **Item 72 — Favor the use of standard exceptions**: reuse `IllegalArgumentException`, `IllegalStateException`, `NullPointerException`, `IndexOutOfBoundsException`, `ConcurrentModificationException`, `UnsupportedOperationException`. Don't reuse `Exception`/`RuntimeException`/`Throwable`/`Error` directly.
- **Item 73 — Throw exceptions appropriate to the abstraction**: **exception translation** — catch a lower-layer exception and rethrow one meaningful at your layer; use **exception chaining** (`new HigherEx(cause)`) to preserve the underlying cause.
- **Item 74 — Document all exceptions thrown by each method**: `@throws` for every checked *and* unchecked exception (unchecked = the method's precondition); never document a group by declaring `throws Exception`.
- **Item 75 — Include failure-capture information in detail messages**: the message should contain the values of all parameters/fields that contributed to the failure (but not passwords/keys). Enables post-mortem debugging from logs.
- **Item 76 — Strive for failure atomicity**: a failed method should leave the object in its pre-call state. Achieve via immutability, up-front validity checks, ordering (do failure-prone work before mutation), recovery code, or operating on a copy.
- **Item 77 — Don't ignore exceptions**: an empty `catch` block defeats the purpose. If ignoring is genuinely correct, name the variable `ignored` and comment *why*.

## Key Concepts
- **Checked exception**: compiler-enforced; signals a recoverable condition the caller must handle or propagate.
- **Unchecked exception (`RuntimeException`)**: programming error / precondition violation; not required to be caught.
- **Error**: abnormal JVM condition; don't create or catch (generally).
- **Exception translation / chaining**: rethrow a higher-level exception wrapping the low-level cause.
- **Failure atomicity**: object state unchanged after a failed operation.

## Mental Models
- Exceptions describe *what went wrong*, not *what to do next* — don't use them as a `goto`.
- Ask "can a well-written caller recover?" → yes ⇒ checked (sparingly); no ⇒ unchecked.
- Each layer should throw in *its own vocabulary*; wrap lower-level failures, keep the cause.
- A good exception message is a debugging note to your future self reading a production log.
- Prefer designs where the failure-prone step happens *before* any state change.

## Anti-patterns
- **Exceptions for control flow**: slow, obscures real errors, defeats JVM optimizations.
- **Overusing checked exceptions**: forces `try/catch` clutter or reflexive rethrows.
- **Custom exception duplicating a standard one**: use the standard.
- **Leaking low-level exceptions** (`SQLException` from a business method): breaks abstraction; translate.
- **`throws Exception`** / undocumented exceptions: uninformative contract.
- **Detail message without the failing values**: unusable in a post-mortem.
- **Empty `catch {}`**: silent failure — the worst exception-handling bug.

## Code Examples
```java
// Item 73 — exception translation with chaining
try {
    return lowLevelDao.load(id);
} catch (SQLException cause) {
    throw new AccountNotFoundException("id=" + id, cause); // higher abstraction + preserved cause
}

// Item 77 — if you must ignore, say so
try {
    numColors = f.get(1L, TimeUnit.SECONDS);
} catch (TimeoutException | ExecutionException ignored) {
    // Use default: minimal coloring is fine here — documented WHY it's safe to ignore
}
```
- **What it demonstrates**: translate to your layer's vocabulary while chaining the cause; and the only acceptable form of "ignoring."

```java
// Item 76 — failure atomicity via up-front check
public Object pop() {
    if (size == 0) throw new EmptyStackException(); // check BEFORE mutating
    Object result = elements[--size];
    elements[size] = null;
    return result;                                   // object unchanged if it threw
}
```

## Reference Tables
| Condition | Exception kind | Item |
|---|---|---|
| Caller can recover | checked (use sparingly) | 70/71 |
| Programming error / bad precondition | unchecked (`RuntimeException`) | 70 |
| JVM/system failure | `Error` (don't create/catch) | 70 |
| Bad argument | `IllegalArgumentException` | 72 |
| Bad object state | `IllegalStateException` | 72 |
| Null arg where disallowed | `NullPointerException` | 72 |
| Unsupported operation | `UnsupportedOperationException` | 72 |

## Worked Example
Exception translation across layers (Item 73). A `List`'s `get(index)` sits atop an abstract `AbstractSequentialList` that iterates. If the underlying `listIterator` throws `NoSuchElementException` for an out-of-range index, that's *wrong vocabulary* for `get` — callers expect `IndexOutOfBoundsException`. So `get` translates:
```java
public E get(int index) {
    ListIterator<E> i = listIterator(index);
    try {
        return i.next();
    } catch (NoSuchElementException e) {
        throw new IndexOutOfBoundsException("Index: " + index, e); // translate + chain
    }
}
```
The caller sees an exception meaningful at the `List` abstraction, while the chained cause preserves the real origin for debugging. Rule of thumb: translate when the lower exception would leak implementation details or confuse the caller — but don't translate *everything*; sometimes propagating unchanged is right.

## Key Takeaways
1. Exceptions are for exceptional conditions only — never control flow.
2. Checked for recoverable, unchecked for programming errors; use checked sparingly.
3. Reuse standard exceptions; never reuse `Exception`/`Throwable`/`Error` directly.
4. Translate exceptions to your abstraction level and chain the cause.
5. Document every thrown exception; put failure-capture values in detail messages (never secrets).
6. Strive for failure atomicity; never silently ignore an exception (if you must, name it `ignored` + comment).

## Connects To
- **Ch8 (Item 49)**: parameter checks throw `IllegalArgumentException`/`NPE`.
- **Ch2 (Item 9)**: try-with-resources cleans up regardless of exceptions.
- **Ch4 (Item 17)**: immutability gives failure atomicity for free.
- **Ch11 (Item 79)**: don't hold locks while code that may throw runs unexpectedly.
