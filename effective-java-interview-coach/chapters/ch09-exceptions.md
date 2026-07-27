# Chapter 10: Exceptions

## Item 69: Use exceptions only for exceptional conditions

### Opening scenario
"Here's a loop that iterates an array by looping `while(true)` and catching `ArrayIndexOutOfBoundsException` to terminate. Why would someone write this, and what's wrong with it?"

### Follow-up probes
- Isn't skipping the bounds-check test a real perf win since the JVM already checks bounds on every array access?
- What happens to debugging if a genuine bug throws that same exception type inside the loop body?
- How does this same anti-pattern show up at the API design level, not just in loops?
- If you were designing `Iterator`, why does `hasNext()` exist at all instead of just calling `next()` until it throws?
- When would you use a distinguished return value (null/empty Optional) instead of a state-testing method?

### Naive attempt
```
try { int i = 0; while(true) range[i++].climb(); } catch (ArrayIndexOutOfBoundsException e) { }
```
Candidate defends it as "avoiding the redundant hidden bounds check the for-each loop pays for anyway."

### What breaks
The reasoning is triple wrong: exceptions have no incentive to be fast (JVMs don't optimize them like explicit tests), try-catch blocks inhibit JIT optimizations, and the standard idiom's bounds checks are frequently eliminated by the JVM anyway — the exception-based version measured about twice as slow on 100-element arrays. Worse, it masks bugs: if some unrelated out-of-bounds access happens inside the loop body, the exception-based version silently swallows it as "normal termination" instead of producing a stack trace and halting.

### The recommended approach & why
Exceptions are, by name, for exceptional conditions — never for ordinary control flow. Use the standard `for (Mountain m : range) m.climb();` idiom. At the API level, a well-designed API must not force clients into exception-driven control flow: pair a state-dependent method (`next()`) with a state-testing method (`hasNext()`), or have the state-dependent method return an empty optional / distinguished value (e.g., null) instead. Prefer a state-testing method when the object isn't subject to concurrent/external state changes and performance parity holds — it's slightly more readable and misuse (forgetting to check) throws obviously rather than silently. If the object can be mutated concurrently or externally, you must use an optional/distinguished value instead, since state could change between the test and the action.

### Panel-ready answer checklist
- Exceptions are for exceptional conditions only — never as a substitute for `if`/loop-termination logic.
- Three concrete reasons exception-based control flow is slower: no JIT incentive to optimize exceptions, try-catch blocks block JIT optimizations, and standard-idiom bounds checks are often already eliminated.
- Exception-based control flow can mask unrelated bugs by misinterpreting them as normal termination.
- API design corollary: pair state-dependent methods with state-testing methods (`hasNext`/`next`), or return Optional/null.
- Use optional/distinguished value instead of a state-testing method when concurrent access or externally-induced state change is possible.

---

## Item 70: Use checked exceptions for recoverable conditions and runtime exceptions for programming errors

### Opening scenario
"A teammate wants to make every exception in the codebase checked 'to be safe, so nobody forgets to handle errors.' What's your pushback?"

### Follow-up probes
- What's the "cardinal rule" for choosing checked vs. unchecked?
- Resource exhaustion could be a bug (huge array allocation) or a genuine transient shortage — how do you decide which kind of exception to use?
- Why shouldn't you ever define your own `Error` subclass, or throw `Error` yourself (barring `AssertionError`)?
- Is it ever appropriate to define a throwable that's neither `Exception`/`RuntimeException` nor `Error`?
- Why does it matter that checked exceptions provide accessor methods, and why is parsing an exception's string representation "extremely bad practice"?

### Naive attempt
"Make everything checked so the compiler forces callers to handle failures — that's inherently safer."

### What breaks
Overusing checked exceptions doesn't map to "recoverable vs. not recoverable" — it just burdens every caller with a `catch` or a `throws` declaration regardless of whether recovery is actually possible. Callers of unrecoverable conditions end up writing meaningless catch blocks (item 71's territory), and API users lean on parsing an exception's string representation for extra detail when no real accessor exists — fragile, since string representations aren't specified and can change release to release.

### The recommended approach & why
Cardinal rule: use checked exceptions for conditions from which the caller can reasonably be expected to recover; use runtime exceptions to indicate programming errors (typically precondition violations, e.g., `ArrayIndexOutOfBoundsException`). A checked exception is a mandate to the caller to handle or propagate it — a strong signal the condition is a possible, recoverable outcome. Unchecked throwables (runtime exceptions and errors) generally mean recovery is impossible and continued execution would do more harm than good; if uncaught, they halt the thread with a message. `Error` is reserved by strong convention for the JVM itself (resource deficiencies, invariant failures) — don't subclass or throw it (except `AssertionError`). Never define a throwable that's neither `Exception`/`RuntimeException` nor `Error` — it behaves as an ordinary checked exception but confuses API users with no benefit. When it's unclear whether a condition is recoverable, default to unchecked (ties to Item 71's reasoning). Checked exceptions should expose accessor methods (e.g., query the shortfall amount on a failed gift-card purchase) so callers can actually act on the failure, not just report it.

### Panel-ready answer checklist
- Cardinal rule: checked = recoverable condition; unchecked = programming error.
- When in doubt about recoverability, use unchecked (segues into Item 71).
- Never subclass or throw `Error` (except `AssertionError`) — that's JVM territory by convention.
- Never define throwables outside the `Exception`/`RuntimeException`/`Error` hierarchy — no benefit, only confusion.
- Checked exceptions should carry accessor methods for recovery info, not just a message string.

---

## Item 71: Avoid unnecessary use of checked exceptions

### Opening scenario
"A method throws one single checked exception, and every caller's catch block does nothing but `e.printStackTrace(); System.exit(1);`. Is this checked exception pulling its weight?"

### Follow-up probes
- What's the litmus test for whether a checked exception is justified?
- Why did Java 8 streams make this problem worse?
- What's the "single checked exception" burden specifically, versus a method that already throws several?
- Name two ways to eliminate an unnecessary checked exception without going fully unchecked-and-undocumented.
- When is the `actionPermitted()`/`action()` two-method refactor NOT appropriate?

### Naive attempt
Defend the checked exception on the grounds that "checked exceptions force programmers to deal with problems, so more checked exceptions = more reliability."

### What breaks
If the caller can't do anything useful except rethrow as `AssertionError` ("can't happen!") or log-and-exit, the checked exception has added zero value and pure friction: it forces a `try`/`catch` or a `throws` declaration, and as of Java 8, a method throwing a checked exception can't be used directly in streams at all. If it's the sole checked exception a method throws, it's literally the only reason that method needs to live inside a `try` block or can't be passed into a stream — the burden is disproportionate to the value.

### The recommended approach & why
Litmus test: ask how the programmer will actually handle the exception, and whether that's the best they can do. If callers won't be able to recover from failure, throw unchecked. If recovery may be possible and you want to force handling, first consider returning an Optional of the result type instead of throwing — trades away the ability to carry extra failure detail, but eliminates the checked-exception burden entirely. Alternatively, split the method in two: a boolean state-testing method (`actionPermitted(args)`) plus the original unchecked action — this converts:
```
try { obj.action(args); } catch (TheCheckedException e) { ... }
```
into:
```
if (obj.actionPermitted(args)) { obj.action(args); } else { ... }
```
This refactor also permits the trivial `obj.action(args);` call when the programmer is confident of success. It's inappropriate under the same conditions as Item 69's state-testing caveat: concurrent access without synchronization or externally induced state changes (state may change between the two calls), or when `actionPermitted` would duplicate `action`'s work (performance-prohibitive).

### Panel-ready answer checklist
- Litmus test: "how would the programmer handle this — is that the best they can do?"
- Checked exceptions can't be used directly in streams (Java 8+) — a real, not just stylistic, cost.
- Prefer returning Optional first; only throw a checked exception if Optional loses needed failure detail.
- Two-method refactor (`actionPermitted`/`action`) turns a checked exception into unchecked while preserving forced-handling semantics.
- Refactor is invalid under concurrent/externally-mutable state or when the test would duplicate the action's work.

---

## Item 72: Favor the use of standard exceptions

### Opening scenario
"A codebase has custom exceptions like `InvalidArgumentValueException` and `ObjectNotReadyException` scattered everywhere, each with no extra fields beyond a message. What's the problem?"

### Follow-up probes
- What's the difference in intent between `IllegalArgumentException` and `IllegalStateException`?
- A caller passes `null` where it's disallowed — `IllegalArgumentException` or something else?
- Why is `ConcurrentModificationException` described as "at best a hint"?
- Is it ever OK to throw `RuntimeException` or `Exception` directly?
- Deck-of-cards example: dealing a hand bigger than the remaining deck — `IllegalArgumentException` or `IllegalStateException`? What's the deciding rule?

### Naive attempt
Define a bespoke exception class per validation failure, each a bare-bones subclass of `RuntimeException` with just a message field, reasoning "explicit names are more descriptive."

### What breaks
Reuse of standard exceptions makes an API easier to learn and read because it matches conventions programmers already know; bespoke one-off exception classes clutter the API, require the reader to go look up unfamiliar types, and add memory/class-loading overhead for no semantic gain. Worse, reuse based only on a name match rather than documented semantics is itself a bug — an exception must be thrown consistent with its documented contract, not just because the name sounds right.

### The recommended approach & why
Reuse standard exceptions per their documented occasions for use:

| Exception | Occasion for Use |
|---|---|
| IllegalArgumentException | Non-null parameter value is inappropriate |
| IllegalStateException | Object state is inappropriate for method invocation |
| NullPointerException | Parameter value is null where prohibited |
| IndexOutOfBoundsException | Index parameter value is out of range |
| ConcurrentModificationException | Concurrent modification detected where prohibited |
| UnsupportedOperationException | Object does not support the method (e.g., append-only List's `remove`) |

Treat `Exception`, `RuntimeException`, `Throwable`, and `Error` as abstract — never throw them directly, since callers can't reliably test for them (they're superclasses of everything). For the deck-of-cards ambiguity: throw `IllegalStateException` if no argument value would have worked (deck too small regardless of hand size requested), otherwise `IllegalArgumentException` (a smaller hand size would have worked). Domain-specific reuse (e.g., `ArithmeticException`, `NumberFormatException` for a complex-number type) is fine too, as long as the throwing conditions match the exception's documented semantics. Subclassing a standard exception to add detail is fine (Item 75) — but exceptions are serializable, so don't create your own class without good reason.

### Panel-ready answer checklist
- `IllegalArgumentException` = bad argument value; `IllegalStateException` = bad receiver state; both are the two most commonly reused.
- Special cases override the general rule by convention: null → `NullPointerException`, bad index → `IndexOutOfBoundsException`.
- Never throw `Exception`/`RuntimeException`/`Throwable`/`Error` directly — treat as abstract, uncatchable-specifically.
- Reuse must follow documented semantics, not just a name that "sounds right."
- Deciding rule for ambiguous cases: `IllegalStateException` if no argument would have worked, else `IllegalArgumentException`.

---

## Item 73: Throw exceptions appropriate to the abstraction

### Opening scenario
"A low-level `SQLException` leaks out of a repository's public method — `findUserById()` throws `SQLException` straight to the controller layer. What's wrong with this, and what should happen instead?"

### Follow-up probes
- Why is this worse than just "an odd exception type" — what happens when the persistence layer implementation changes later?
- What's the difference between exception translation and exception chaining, and when do you need the latter?
- Isn't the "best" fix here just to catch `SQLException` at the higher layer and prevent it from ever occurring? Why can't you always do that?
- If exception translation isn't feasible either, what's the fallback?
- Show me how you'd write the chaining-aware constructor.

### Naive attempt
`catch (SQLException e) { throw new RuntimeException(e.getMessage()); }` — rethrow as a generic unchecked exception, discarding the original exception object.

### What breaks
Two failures: (1) it still pollutes the caller with implementation detail conceptually (a generic `RuntimeException` about SQL is meaningless at the repository abstraction level, and if the persistence tech changes, e.g., swapping JDBC for a different driver, all client code coupled to that shape breaks); (2) by passing only `e.getMessage()` instead of `e` itself as the cause, the original stack trace and exception object are lost — impossible to do root-cause debugging from the trace alone.

### The recommended approach & why
Higher layers should catch lower-level exceptions and, in their place, throw exceptions explainable in terms of the higher-level abstraction — exception translation:
```
try { ... } catch (LowerLevelException e) { throw new HigherLevelException(...); }
```
Example from `AbstractSequentialList.get`: catches `NoSuchElementException` from the iterator, throws `IndexOutOfBoundsException` instead, per the `List` interface's own specification. When the lower-level exception carries debugging value, use exception chaining — pass the lower exception as the cause via a chaining-aware constructor (`super(cause)`), which both exposes it via `getCause()` and splices its stack trace into the higher exception's:
```
try { ... } catch (LowerLevelException cause) { throw new HigherLevelException(cause); }
```
But translation shouldn't be overused — the best option, where feasible, is to prevent lower-layer exceptions entirely (e.g., validate higher-level parameters before delegating down). If prevention isn't possible, the next-best option is to have the higher layer silently work around the exception and just log it, insulating the caller entirely.

### Panel-ready answer checklist
- Never let a lower-layer exception leak unmodified through a higher-level abstraction's API.
- Exception translation: catch low-level, throw a higher-level exception that matches the abstraction.
- Exception chaining: pass the original exception as `cause` (chaining-aware constructor / `initCause`) when it aids debugging — preserves both `getCause()` access and the original stack trace.
- Preference order: prevent the lower-layer exception first; translate/chain second; silently log-and-insulate as last resort.
- Don't overuse translation — only apply it where the lower-level method doesn't already guarantee exceptions appropriate to the higher level.

---

## Item 74: Document all exceptions thrown by each method

### Opening scenario
"A public method's signature reads `public void process(Data d) throws Exception`. What's wrong with this declaration, beyond style?"

### Follow-up probes
- Why is declaring `throws Exception` worse than just "vague" — what does it actively hide?
- Should unchecked exceptions be documented too, even though the compiler doesn't require it?
- Why do interface methods have an extra obligation to document unchecked exceptions?
- How do you signal in Javadoc that an exception is unchecked rather than checked?
- Is it realistic to promise "every unchecked exception a method can throw is documented"? What breaks that promise over time?

### Naive attempt
Declare `throws Exception` (or worse, `throws Throwable`) on a public method "to cover all bases," and skip documenting unchecked exceptions since the compiler doesn't enforce them.

### What breaks
Declaring a supertype like `Exception`/`Throwable` gives zero guidance to the caller about which exceptions are actually possible, and it obscures any other exception that might legitimately be thrown in the same context — the caller can't distinguish "expected" from "unexpected" failures at all. Skipping documentation of unchecked exceptions denies callers the preconditions of the method (unchecked exceptions largely describe precondition violations), making the method's contract incomplete.

### The recommended approach & why
Always declare checked exceptions individually in the `throws` clause, and document precisely, via Javadoc `@throws`, the condition under which each is thrown — never take the shortcut of a shared superclass declaration (the sole exception: `main`, which may safely declare `throws Exception` since only the VM calls it). Document unchecked exceptions too, with `@throws` but without adding them to the `throws` clause — the absence of the `throws` keyword alongside a documented `@throws` tag is itself the visual cue that the exception is unchecked. This is especially critical for interface methods, since that documentation becomes part of the interface's general contract shared by every implementation. If an exception (typically `NullPointerException`) is common to many methods in a class for the same reason, document it once at the class level instead of repeating it per method. Note the ideal isn't always fully achievable — if a class you depend on is revised to throw new unchecked exceptions, your undocumented code can start propagating them without you knowing.

### Panel-ready answer checklist
- Declare checked exceptions individually with `@throws`; never declare a shared supertype like `Exception`/`Throwable` (main() is the sole exception).
- Document unchecked exceptions too — via `@throws` only, never added to the `throws` clause; the missing `throws` keyword is the signal it's unchecked.
- Undocumented unchecked exceptions effectively describe undocumented preconditions — bad for the same reason undocumented preconditions are bad.
- Interface methods carry extra weight: their exception documentation is part of the shared contract across all implementations.
- Class-level documentation is fine for exceptions common to many methods for the same reason (e.g., blanket `NullPointerException` note).

---

## Item 75: Include failure-capture information in detail messages

### Opening scenario
"Production incident: all you have is a stack trace reading `IndexOutOfBoundsException: Index out of range`. No index value, no bounds. What should the exception message have contained, and why does it matter that it didn't?"

### Follow-up probes
- Why is the exception's `toString()` — class name plus detail message — sometimes literally the only diagnostic information available?
- What's the difference between a detail message and a user-facing error message?
- Should the detail message include a paragraph of prose explaining the failure?
- Is there any category of information you should deliberately keep out of a detail message?
- How do you make it structurally hard for a caller to forget to include failure-capture info?

### Naive attempt
`throw new IndexOutOfBoundsException("Index out of range");` — a human-readable but data-free message.

### What breaks
If the failure isn't easily reproducible, this message is the entire diagnostic record. It doesn't reveal which of three possible root causes applies: a fencepost error (index one below lower bound, or equal to upper bound), a wildly out-of-range index, or a serious internal invariant failure (lower bound greater than upper bound) — each demands a different fix, and without the actual values, the engineer is guessing blind.

### The recommended approach & why
The detail message should capture the failure: include the values of all parameters and fields that contributed to the exception (for `IndexOutOfBoundsException`: lower bound, upper bound, and the failing index value). Skip lengthy prose — the stack trace already pins the exact file/line, so a wall of explanatory text is superfluous once you have the data and can read the source. Never include security-sensitive data (passwords, encryption keys) since stack traces circulate widely during debugging. Keep detail messages conceptually distinct from user-level error messages: detail messages serve programmers/SREs and prioritize information density over readability or localization; user-level messages must be intelligible to end users and are often localized. The most reliable enforcement mechanism: require the failure-capture values as constructor parameters instead of accepting a free-text `String` message, and generate the message internally:
```
public IndexOutOfBoundsException(int lowerBound, int upperBound, int index) {
    super(String.format("Lower bound: %d, Upper bound: %d, Index: %d", lowerBound, upperBound, index));
    this.lowerBound = lowerBound;
    this.upperBound = upperBound;
    this.index = index;
}
```
This makes it hard for the throwing code not to capture the failure. Provide accessor methods for this data too — more important on checked exceptions (recovery-relevant) than unchecked, but advisable on both as general principle.

### Panel-ready answer checklist
- Detail message must include every parameter/field value relevant to the failure — not prose, values.
- The stack trace's `toString()` is frequently the only evidence available for a non-reproducible failure.
- Never leak security-sensitive data (passwords, keys) into a detail message.
- Detail message (programmer-facing, dense) is a distinct concept from user-level error message (localized, must be intelligible to end users).
- Best structural fix: accept failure-capture data as constructor parameters, not a free-text message, and expose accessors for it.

---

## Item 76: Strive for failure atomicity

### Opening scenario
"`Stack.pop()` throws `EmptyStackException` when called on an empty stack — but suppose the size check were removed and it just let the array-index bug happen naturally. What's wrong with that version, beyond the wrong exception type?"

### Follow-up probes
- What does "failure-atomic" mean precisely?
- Why is failure atomicity "free" for immutable objects?
- What's the general-purpose technique of "ordering computation" to get failure atomicity almost for free, and how does `TreeMap` naturally benefit from it?
- Name a technique that gets failure atomicity as a side effect of a performance optimization.
- Is failure atomicity always achievable or desirable? Give a case where it isn't.

### Naive attempt
Skip the size check in `pop()`:
```
public Object pop() {
    Object result = elements[--size];
    elements[size] = null;
    return result;
}
```
"It'll still throw an exception on an empty stack, so we're fine."

### What breaks
It does throw — but the wrong one (`ArrayIndexOutOfBoundsException`, inappropriate to the `Stack` abstraction per Item 73), and worse, it leaves `size` in a corrupted negative state, so every subsequent call on that object fails too. The object is no longer usable after the failed call — that's the definition of a failure-atomicity violation.

### The recommended approach & why
A failed method invocation should, as a rule, leave the object in the state it was in prior to the invocation — that property is called failure-atomic. Several ways to get there, in order of frequency/ease: (1) design immutable objects — failure atomicity is then free, since object state is fixed at construction and a failed operation just prevents a new object from existing; (2) check parameters for validity before any modification begins (the actual fix for `pop()`: check `size == 0` and throw `EmptyStackException` before touching the array); (3) order the computation so any part that can fail runs before any part that mutates state — `TreeMap.put` naturally gets this because the `ClassCastException` from an incomparable element surfaces during the tree search, before any modification; (4) perform the operation on a temporary copy and swap it in only on success (e.g., sorting algorithms that copy into an array for the sort, incidentally leaving the input list untouched if the sort throws); (5) write explicit recovery/rollback code — rare, mostly used for durable/disk-based structures. Failure atomicity isn't always achievable (e.g., after a `ConcurrentModificationException`, the object's state can't be assumed intact) or always worth the cost/complexity it can add — but it's frequently free once you're deliberately aware of the issue.

### Panel-ready answer checklist
- Failure-atomic = a failed call leaves the object exactly as it was before the call.
- Immutability gets failure atomicity for free — no existing object state can be mutated at all.
- Validate-before-mutate (or naturally order fail-prone work before mutating work) is the standard technique for mutable objects.
- Copy-then-swap and explicit rollback are viable but less common techniques.
- Not always achievable (concurrent modification) or worth the cost — but often cheap once you design for it deliberately.

---

## Item 77: Don't ignore exceptions

### Opening scenario
"Code review: you spot `try { ... } catch (SomeException e) { }` — an empty catch block. Why not just approve it and move on?"

### Follow-up probes
- Isn't an empty catch block sometimes legitimate — e.g., closing a stream you're done reading from?
- If you do decide to ignore an exception, what should the catch block still contain?
- Does this advice apply differently to checked versus unchecked exceptions?
- What's the actual failure mode of a silently-swallowed exception, concretely — how does the program eventually fail?
- What's the fire-alarm analogy driving at?

### Naive attempt
```
try { ... } catch (SomeException e) { }
```
"It compiles, it doesn't crash, ship it."

### What breaks
An empty catch block defeats the entire purpose of exceptions — forcing you to handle exceptional conditions — and lets the program continue silently in the face of an actual error. The failure surfaces later, at an arbitrary point in the code bearing no apparent relation to the real problem, making root-causing far harder than if the exception had simply been allowed to propagate and fail fast.

### The recommended approach & why
Treat an empty catch block as inherently suspect — an API designer who declared a method to throw an exception is telling you something specific; ignoring it is like disabling a fire alarm so nobody else can check whether there's a real fire. There are legitimate ignore cases (e.g., closing a `FileInputStream` after you've already read everything you needed — no state changed, nothing to recover, no reason to abort), but even then: log the exception if it might recur, comment the catch block explaining why ignoring is safe, and name the exception variable `ignored`:
```
Future<Integer> f = exec.submit(planarMap::chromaticNumber);
int numColors = 4; // Default; guaranteed sufficient for any map
try {
    numColors = f.get(1L, TimeUnit.SECONDS);
} catch (TimeoutException | ExecutionException ignored) {
    // Use default: minimal coloring is desirable, not required
}
```
This advice applies equally to checked and unchecked exceptions — swallowing either produces the same silent-continuation risk. Properly handling the exception can avert failure entirely; at minimum, letting it propagate causes a fast, informative failure instead of a delayed, mysterious one.

### Panel-ready answer checklist
- An empty catch block defeats the purpose of exceptions and should always trigger scrutiny in review.
- Legitimate ignore cases exist (e.g., cleanup after you've already gotten what you needed) but are the exception, not the norm.
- If ignoring is justified: comment why, name the variable `ignored`, and consider logging for recurrence detection.
- Applies equally to checked and unchecked exceptions — no special exemption for either.
- Swallowed exceptions cause delayed, hard-to-trace failures; propagating them causes fast, diagnosable ones.
