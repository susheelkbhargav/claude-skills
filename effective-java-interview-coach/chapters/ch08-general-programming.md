# Chapter 9: General Programming

## Item 57: Minimize the scope of local variables

### Opening scenario
"Here's a 40-line method. Halfway through code review, someone asks: 'why is `result` declared at the top when it's not touched until line 30?' Walk me through why that's a problem beyond style."

### Follow-up probes
- Is there ever a legitimate reason to declare a variable before you can sensibly initialize it? (try-catch)
- Why does a `for` loop beat a `while` loop here, beyond fewer keystrokes?
- I have two nearly identical while-loops back to back, each with its own `Iterator`. What's the classic bug lurking there?
- Why does the equivalent code written with `for` loops fail to compile instead of silently misbehaving?
- What's the "final technique" for scope minimization that has nothing to do with variable declarations at all?

### Naive attempt
Declare all variables at the top of the method "C-style," including loop iterators reused loop after loop, e.g. two consecutive `while` loops sharing variable name `i` where the second loop actually needs a fresh iterator `i2` but the programmer copy-pastes and forgets to update a reference inside the loop condition.

### What breaks
```
Iterator<Element> i = c.iterator();
while (i.hasNext()) {
    doSomething(i.next());
}
...
Iterator<Element> i2 = c2.iterator();
while ( i .hasNext()) { // BUG!
    doSomethingElse(i2.next());
}
```
The second loop's condition still references the stale `i`, which is still in scope and already exhausted. The code compiles, runs without exception, and silently does nothing — `c2` appears empty when it isn't. The bug is invisible because there's no crash, just wrong behavior.

### The recommended approach & why
Declare each variable at the point of first use, with an initializer, so its scope is exactly as large as needed. Use `for` loops (traditional or for-each) instead of `while` for loop variables — the scope of the loop variable is limited to the loop body plus the parenthesized header, so a copy-paste error becomes a compile-time "cannot find symbol" error instead of a silent runtime bug. A special idiom, `for (int i = 0, n = expensiveComputation(); i < n; i++)`, caches an expensive, loop-invariant limit check as a second loop variable. Finally, keep methods small and focused — combining two activities in one method spreads one activity's variables into the other's scope.

### Panel-ready answer checklist
- Declare where first used, always with an initializer (except try-catch's forced early declaration).
- Prefer `for`/for-each over `while` for loop variables — narrower scope turns silent bugs into compile errors.
- Use the two-variable `for (i, n = expensiveComputation(); ...)` idiom to avoid redundant per-iteration computation.
- Keep methods small and single-purpose to prevent variable scope creep across unrelated logic.

---

## Item 58: Prefer for-each loops to traditional for loops

### Opening scenario
"A junior engineer wrote nested loops to build a deck of cards from `Suit` and `Rank` enums using `Iterator`s directly, and it throws `NoSuchElementException` intermittently. Diagnose it."

### Follow-up probes
- Why doesn't the equivalent nested for-each version have this bug at all?
- Show me a variant of this bug that *doesn't* throw — what does it do instead?
- Name three situations where for-each genuinely cannot be used.
- Is there a performance penalty for for-each on arrays versus indexed access?
- What single-method interface makes a custom type work with for-each, and why is it non-trivial to implement well?

### Naive attempt
```
for (Iterator<Suit> i = suits.iterator(); i.hasNext(); )
    for (Iterator<Rank> j = ranks.iterator(); j.hasNext(); )
        deck.add(new Card(i.next(), j.next()));
```

### What breaks
`i.next()` (the outer iterator) is called once per inner-loop iteration instead of once per outer iteration — it's called from the inner loop's body. After `suits` runs out of elements, the loop throws `NoSuchElementException`. Worse variant: if both collections are the same size (e.g. iterating `Face` against itself for dice rolls), the loop terminates normally but prints only six "doubles" instead of the expected thirty-six combinations — no crash, just silently wrong output.

### The recommended approach & why
```
for (Suit suit : suits)
    for (Rank rank : ranks)
        deck.add(new Card(suit, rank));
```
The for-each loop (the "enhanced for statement") hides the iterator/index entirely, eliminating both the clutter and the opportunity to misuse it, with **no performance penalty** even for arrays — the generated code is essentially identical to hand-written indexed code. It applies uniformly to collections, arrays, and anything implementing `Iterable<E>` (one method: `Iterator<E> iterator()`). Three situations still require a traditional loop: destructive filtering (need `Iterator.remove`, or use `Collection.removeIf` since Java 8), transforming (need list iterator or index to replace elements), and parallel iteration (need explicit lockstep control over multiple iterators/indices).

### Panel-ready answer checklist
- For-each hides the iterator/index variable, eliminating the classic "wrong variable reused in nested loop" bug class entirely.
- Zero performance penalty versus hand-written indexed loops, even for arrays.
- Works over anything implementing `Iterable<E>` — consider implementing it for your own aggregate types.
- Three legitimate exceptions: destructive filtering, transforming, parallel iteration — use `removeIf` to avoid the first when possible.

---

## Item 59: Know and use the libraries

### Opening scenario
"Someone wrote their own `random(int n)` helper using `Random` and `Math.abs(rnd.nextInt()) % n` because they didn't trust the built-in bound-aware method. Code review flags it. What's wrong?"

### Follow-up probes
- Walk through exactly why `Math.abs(rnd.nextInt()) % n` is biased when `n` isn't a power of two.
- There's a catastrophic edge case buried in `Math.abs` here — what input triggers it?
- As of Java 7, what should be used instead of `Random`, and why?
- Beyond correctness, name three other advantages of preferring library code over hand-rolled solutions.
- When is it legitimate to write your own instead of using the library?

### Naive attempt
```
// Common but deeply flawed!
static Random rnd = new Random();
static int random(int n) {
    return Math.abs(rnd.nextInt()) % n;
}
```

### What breaks
Three flaws: (1) if `n` is a small power of two, the sequence repeats after a short period; (2) if `n` is not a power of two, some numbers are returned disproportionately more often — demonstrated by a test where two-thirds of a million generated numbers fall in the lower half of the range instead of the expected half; (3) it can fail catastrophically — if `rnd.nextInt()` returns `Integer.MIN_VALUE`, `Math.abs` returns `Integer.MIN_VALUE` unchanged (overflow), and `%` then yields a negative number, producing an out-of-range result that's hard to reproduce.

### The recommended approach & why
Use `Random.nextInt(int)` — designed, implemented, and battle-tested by an expert and used by millions of programmers for decades. As of Java 7, prefer `ThreadLocalRandom` for most uses (higher quality, ~3.6x faster than `Random` on the author's machine); use `SplittableRandom` for fork-join pools and parallel streams. More broadly: library code gets expert design, community-tested correctness, ongoing performance improvements "with no effort on your part," growing functionality over releases, and puts your code in the mainstream for readability/maintainability by other developers. Every programmer should know `java.lang`, `java.util`, `java.io` and their subpackages at minimum; check the library before writing ad hoc code, and fall back to high-quality third-party libraries (e.g., Guava) before rolling your own.

### Panel-ready answer checklist
- Hand-rolled `% n` bounding is measurably biased and can catastrophically fail on `Integer.MIN_VALUE` via `Math.abs` overflow.
- Use `Random.nextInt(int)`; prefer `ThreadLocalRandom` (Java 7+) generally, `SplittableRandom` for fork-join/parallel streams.
- Library code wins on correctness, performance-over-time, growing functionality, and mainstream maintainability.
- Check the JDK libraries first, then high-quality third-party (Guava), then write your own as a last resort.

---

## Item 60: Avoid float and double if exact answers are required

### Opening scenario
"A billing system prints `$0.09999999999999998` instead of `$0.10` in change-due output. The finance team is furious. What's actually going on, and what's the fix?"

### Follow-up probes
- Why can't rounding right before printing reliably fix this?
- Walk through the candy-shelf example: naive code says 3 items bought, correct answer is 4 — why?
- When would you choose `int`/`long` over `BigDecimal`, and what's the tradeoff?
- What's special about how `BigDecimal`'s constructor is invoked in the correct example, and why does it matter?
- Up to how many decimal digits can you safely use `int` vs `long` for money math?

### Naive attempt
```
// Broken - uses floating point for monetary calculation!
public static void main(String[] args) {
    double funds = 1.00;
    int itemsBought = 0;
    for (double price = 0.10; funds >= price; price += 0.10) {
        funds -= price;
        itemsBought++;
    }
    System.out.println(itemsBought + " items bought.");
    System.out.println("Change: $" + funds);
}
```

### What breaks
`float`/`double` perform binary floating-point arithmetic, which cannot represent 0.1 (or any negative power of ten) exactly. `System.out.println(1.03 - 0.42)` prints `0.6100000000000001`. `System.out.println(1.00 - 9 * 0.10)` prints `0.09999999999999998`. In the candy example, the accumulated binary rounding error causes the loop to report only 3 items bought with $0.3999999999999999 left over — the actual correct answer is 4 items with $0.00 left over.

### The recommended approach & why
Use `BigDecimal`, `int`, or `long` for monetary/exact calculations. Construct `BigDecimal` from its **String** constructor (`new BigDecimal(".10")`), not the double constructor, to avoid baking inaccurate binary values into the computation. `BigDecimal` gives full control over rounding via eight selectable rounding modes — valuable for legally mandated rounding in business calculations — at the cost of being less convenient and slower than a primitive type. Alternatively, use `int`/`long` and track the decimal point yourself (e.g., do everything in cents); this is faster and more convenient but requires manual bookkeeping. Use `int` if quantities won't exceed nine decimal digits, `long` up to eighteen digits, and `BigDecimal` beyond that.

### Panel-ready answer checklist
- `float`/`double` cannot exactly represent negative powers of ten — never use them where exact results are required, especially money.
- Rounding at print time doesn't fix accumulated error across a computation (candy-shelf example).
- Use `BigDecimal` (via its String constructor) for correctness and rounding-mode control; use `int`/`long` in cents for speed if you'll track the decimal point yourself.
- Digit budget: `int` ≤ 9 digits, `long` ≤ 18 digits, `BigDecimal` beyond that.

---

## Item 61: Prefer primitive types to boxed primitives

### Opening scenario
"This comparator passes tests on a million-element list, but `naturalOrder.compare(new Integer(42), new Integer(42))` returns 1 instead of 0. Find the bug."

### Follow-up probes
- Why does `i == j` misbehave here specifically, and when does `i < j` work fine?
- Now look at this: a static `Integer i` field, and `if (i == 42)` throws `NullPointerException`. Why?
- What's wrong performance-wise with declaring an accumulator as `Long sum = 0L` in a hot loop over `long`?
- Name three legitimate uses where you're *forced* to use boxed primitives.
- What's the actual fix for the broken comparator, without just calling `Comparator.naturalOrder()`?

### Naive attempt
```
// Broken comparator - can you spot the flaw?
Comparator<Integer> naturalOrder =
    (i, j) -> (i < j) ? -1 : (i == j ? 0 : 1);
```

### What breaks
`i < j` auto-unboxes correctly and works fine. But when it's false, `i == j` performs **identity comparison** on the boxed `Integer` references, not value comparison. Two distinct `Integer` instances holding the same value (e.g., two separately created `Integer(42)`) are `==`-unequal, so the comparator wrongly returns 1. Separately: `static Integer i;` defaults to `null`; evaluating `i == 42` auto-unboxes `i`, and unboxing `null` throws `NullPointerException`. And: declaring an accumulator as `Long` instead of `long` in a loop that runs ~2^31 times causes repeated boxing/unboxing, causing severe, silent performance degradation with no compiler warning.

### The recommended approach & why
Prefer primitives whenever you have a choice — they're simpler and faster, and boxed primitives introduce three hazards: distinct identity from value (breaks `==`), a non-functional `null` value (unboxing risk), and time/space cost. For natural ordering, call `Comparator.naturalOrder()`, or if writing your own comparator, unbox to local primitive variables first and compare those. Boxed primitives are legitimately required as elements/keys/values in collections, as type parameters in generics (primitives aren't allowed there), and for reflective method invocation.

### Panel-ready answer checklist
- `==` on boxed primitives is identity comparison, "almost always wrong" — unbox to primitives before comparing, or use `Comparator.naturalOrder()`.
- Mixed primitive/boxed operations trigger auto-unboxing; unboxing a `null` throws `NullPointerException` and can happen almost anywhere.
- Boxed primitives in hot loops cause silent, severe performance degradation via repeated box/unbox — compiles clean, runs slow.
- Legitimate boxed-primitive uses: collection elements, generic type parameters, reflective invocation — otherwise prefer primitives.

---

## Item 62: Avoid strings where other types are more appropriate

### Opening scenario
"Codebase has `String compoundKey = className + \"#\" + i.next();` used as a map key throughout the system. What's wrong with this design, beyond taste?"

### Follow-up probes
- What happens if the separator character (`#`) appears inside one of the fields being concatenated?
- Why can't you give this compound key sensible `equals`, `toString`, or `compareTo` behavior?
- Walk through the string-keyed `ThreadLocal` design — what's the actual security hole?
- How do you evolve that broken API step by step into something typesafe?
- Besides aggregate types, what other two categories of data are commonly, wrongly, represented as strings?

### Naive attempt
```
// Inappropriate use of string as aggregate type
String compoundKey = className + "#" + i.next();
```
And, for capability-style access control:
```
// Broken - inappropriate use of string as capability!
public class ThreadLocal {
    private ThreadLocal() { }
    public static void set(String key, Object value);
    public static Object get(String key);
}
```

### What breaks
For the compound key: if the separator character occurs inside a field's actual value, parsing becomes ambiguous ("chaos may result"); reading a field requires slow, tedious, error-prone string parsing; you're stuck with `String`'s `equals`/`toString`/`compareTo` semantics instead of ones meaningful to the aggregate. For the string-keyed `ThreadLocal`: the string keys form a shared global namespace — two independent clients choosing the same key name unintentionally share state and both fail; worse, a **malicious** client can deliberately reuse another client's key string to gain illicit access to that client's data — a real security hole, not just a naming collision.

### The recommended approach & why
Don't use strings as substitutes for value types, enum types, aggregate types, or capabilities. If numeric, translate to `int`/`float`/`BigInteger`; if a fixed enumeration, use an actual `enum`; if it has multiple components, write a class (often a private static member class) to represent it. For capabilities, replace the forgeable string key first with an unforgeable `Key` object, then recognize the static methods can become instance methods on the key itself — collapsing to a typesafe, parameterized `ThreadLocal<T>` with `set(T)`/`get()` — which is essentially `java.lang.ThreadLocal`'s actual design, and is both faster and more elegant than the string- or key-based intermediate designs.

### Panel-ready answer checklist
- Strings are poor substitutes for value types, enums, aggregate types, and capabilities — use the appropriate type instead, or write one.
- Aggregate-as-string breaks on separator collision and forces you to accept `String`'s equals/toString/compareTo.
- String-keyed capability APIs (e.g., early ThreadLocal designs) are a genuine security hole: a malicious client can reuse another's key to steal access.
- The fix path: string key → unforgeable `Key` object → instance methods on the key → typesafe generic class (`ThreadLocal<T>`).

---

## Item 63: Beware the performance of string concatenation

### Opening scenario
"This method that builds a billing statement by looping and doing `result += lineForItem(i)` runs fine in dev with 5 items, then times out in production with thousands of line items. Why?"

### Follow-up probes
- Why exactly is repeated `+` concatenation quadratic, not linear, in the number of strings?
- What's the actual mechanism — why does immutability of `String` force a full copy each time?
- Give me a concrete speedup number for the `StringBuilder` fix.
- Does preallocating the `StringBuilder`'s capacity matter, and by how much?
- When is plain `+` concatenation actually fine to use?

### Naive attempt
```
// Inappropriate use of string concatenation - Performs poorly!
public String statement() {
    String result = "";
    for (int i = 0; i < numItems(); i++)
        result += lineForItem(i); // String concatenation
    return result;
}
```

### What breaks
Because `String` is immutable, each `+=` concatenation copies the contents of both the accumulated result and the new line into a brand-new `String`. Concatenating `n` strings this way requires time **quadratic in n**. The method "performs abysmally" as the number of items grows — it's fine for a handful of items but doesn't scale.

### The recommended approach & why
```
public String statement() {
    StringBuilder b = new StringBuilder(numItems() * LINE_WIDTH);
    for (int i = 0; i < numItems(); i++)
        b.append(lineForItem(i));
    return b.toString();
}
```
Use `StringBuilder.append` instead — linear in `n`. With `numItems()` = 100 and 80-character lines, the `StringBuilder` version ran 6.5x faster on the author's machine (still 5.5x faster even with a default-sized `StringBuilder` instead of preallocating), and the gap widens as item count grows because one approach is quadratic and the other linear. Alternatively, use a character array or process strings one at a time instead of combining them. Plain `+` is fine only for a small, fixed number of concatenations (e.g., a single output line) where performance is irrelevant.

### Panel-ready answer checklist
- Repeated `+=` concatenation is quadratic in the number of strings because each op copies both operands (String is immutable).
- `StringBuilder.append` in a loop is linear — use it whenever concatenating more than "a few" strings.
- Preallocating the `StringBuilder`'s capacity helps further but isn't the main win — the algorithmic complexity change is.
- Plain `+` remains fine for small, fixed-size concatenations where performance doesn't matter.

---

## Item 64: Refer to objects by their interfaces

### Opening scenario
"A field is declared as `LinkedHashSet<Son> sonSet = new LinkedHashSet<>();` instead of `Set<Son>`. Six months later someone wants to swap to `HashSet` for a memory win and the change breaks unrelated call sites. Why did declaring it as the concrete class cause that?"

### Follow-up probes
- What's the one time you actually must name the concrete class rather than the interface?
- Give an example where switching implementation type is desirable — and one where it would be *unsafe* even though it compiles.
- Name a real caveat: when would swapping `LinkedHashSet` for `HashSet` silently change program behavior?
- Are there cases where there's no appropriate interface at all? Name the three categories.
- Why does `PriorityQueue` specifically belong to one of those exception categories?

### Naive attempt
```
// Bad - uses class as type!
LinkedHashSet<Son> sonSet = new LinkedHashSet<>();
```

### What breaks
Client code written against the concrete type may call methods present on `LinkedHashSet` but absent from a replacement like `HashSet`, or may pass the variable to another method whose parameter type demands the original concrete class. When you later change the implementation type, "there is no guarantee that this change will result in a program that compiles" — even though you also changed the declaration type at the same time. Separately, if surrounding code silently depended on `LinkedHashSet`'s predictable iteration order, swapping to `HashSet` (which makes no ordering guarantee) compiles fine but changes runtime behavior.

### The recommended approach & why
```
// Good - uses interface as type
Set<Son> sonSet = new LinkedHashSet<>();
```
Favor interface types for parameters, return values, variables, and fields wherever an appropriate interface exists; reserve the concrete class name for the constructor call itself. This lets you swap implementations (e.g. to `HashMap`→`EnumMap` for performance, or `HashMap`→`LinkedHashMap` for predictable iteration order) by changing only the constructor call — "declaring the variable with the interface type keeps you honest," since the compiler will catch incompatible usages immediately rather than silently compiling into behavior you didn't intend. Three legitimate exceptions to "always use an interface": (1) value classes like `String`/`BigInteger` with no interface and no intended multiple implementations; (2) class-based frameworks (e.g., `java.io`'s `OutputStream`) where the fundamental type is an abstract base class, not an interface; (3) classes offering extra methods beyond their interface (e.g., `PriorityQueue.comparator()` isn't on `Queue`) — use the class directly only if you truly rely on those extras.

### Panel-ready answer checklist
- Declare parameters/fields/variables with the interface type; the class name should appear only at the constructor call.
- This keeps implementation swaps compiler-checked instead of silently breaking or silently changing behavior (e.g., losing `LinkedHashSet`'s ordering guarantee).
- Three legitimate exceptions: value classes (String, BigInteger), class-based frameworks (OutputStream), classes with extra non-interface methods you actually need (PriorityQueue.comparator()).
- If no interface exists, use the least specific class in the hierarchy that still provides what you need.

---

## Item 65: Prefer interfaces to reflection

### Opening scenario
"A plugin-loading system uses `Class.forName` plus `Method.invoke` everywhere to call into plugin code, and the team wants to know why startup is slow and why a bad plugin JAR causes six different exception types instead of clean compile errors."

### Follow-up probes
- Name the three core costs of using reflection listed in the book.
- Given the toy `Set<String>` example that reflectively instantiates a class by name, how many exception types can it throw and why?
- Once the object is instantiated reflectively, is the rest of the program actually affected by reflection's costs?
- What's the one legitimate, narrow way to use reflection that the book actually recommends?
- Name the "legitimate, if rare" use case for reflection involving version compatibility.

### Naive attempt
Use `java.lang.reflect` pervasively: obtain `Class`, then `Constructor`/`Method`/`Field` instances, and invoke/construct/access everything through them, throughout the application rather than only at object-creation boundaries.

### What breaks
You lose all compile-time type checking, including exception checking — a reflective call to a nonexistent or inaccessible method fails only at **runtime**. The code is "clumsy and verbose... tedious to write and difficult to read." Performance suffers badly — the book measures reflective invocation of a no-arg, int-returning method as **eleven times slower** than normal invocation. The toy example that instantiates a `Set` implementation reflectively by class name can throw six different runtime exceptions (`ClassNotFoundException`, `NoSuchMethodException`, `IllegalAccessException`, `InstantiationException`, `InvocationTargetException`, `ClassCastException`) — all of which would have been ordinary compile-time errors with direct instantiation.

### The recommended approach & why
Use reflection, if at all, only in a very limited form: to **create instances** reflectively, then immediately access and use those objects through an ordinary compile-time-known interface or superclass (Item 64). In the book's example, a `Set<String>` instance is created reflectively from a class name given on the command line, but once constructed, "the set is indistinguishable from any other `Set` instance" — the reflection cost and risk is confined entirely to the instantiation step, and the bulk of the program is unaffected. A second legitimate, rare use: managing dependencies on classes/methods that may be absent at runtime (e.g., supporting multiple versions of another package), compiling against the oldest supported version and reflectively probing for newer APIs, with a fallback if they're absent.

### Panel-ready answer checklist
- Reflection costs: no compile-time type/exception checking, verbose/hard-to-read code, and measured ~11x slower invocation.
- Confine reflection to object creation; access the result through a compile-time-known interface or superclass afterward.
- The instantiate-by-name toy example demonstrates both disadvantages concentrated in ~25 lines — six possible runtime exceptions instead of compile errors.
- Legitimate rare use: reflectively probing for newer classes/methods to support multiple versions of a dependency, with a fallback.

---

## Item 66: Use native methods judiciously

### Opening scenario
"An engineer proposes rewriting a hot numeric loop in C via JNI for 'guaranteed' performance. What questions do you ask before approving that?"

### Follow-up probes
- Historically, what were the three legitimate uses of native methods — are all three still as relevant today?
- What happened when `BigInteger`'s native multiprecision arithmetic was reimplemented in pure Java in release 3?
- Since Java code isn't memory-safe when native methods are involved, what category of bug reappears?
- Why can native methods actually hurt garbage collector performance, not just add call overhead?
- Is there a modern, legitimate case where native methods for performance are still justified?

### Naive attempt
Reach for native methods (JNI) by default whenever a code path looks performance-critical, on the assumption that C/C++ is automatically faster than Java.

### What breaks
It's "rarely advisable to use native methods for improved performance" today — JVMs have matured to the point where comparable performance is usually achievable in pure Java. Concretely, when `java.math` was added in 1.1, `BigInteger` relied on a native multiprecision C library; in Java 3 it was reimplemented in Java and, once carefully tuned, ran **faster** than the original native implementation. Beyond that, native methods reintroduce memory corruption risk (native languages aren't safe), reduce portability (more platform-dependent), are harder to debug, can hurt performance because the garbage collector "can't automate, or even track, native memory usage," incur cost crossing the native/Java boundary, and require tedious, hard-to-read "glue code."

### The recommended approach & why
Think twice before using native methods; it's rare that you need them for performance. Legitimate uses remain: accessing genuinely platform-specific facilities not otherwise exposed (though the Java platform's process API and similar additions have closed most such gaps), and using existing native libraries with no Java equivalent — the book cites GMP (GNU Multiple Precision arithmetic library) as a case where truly high-performance multiprecision arithmetic still justifies native methods, since `BigInteger` itself has changed little since Java 8. If you must use native code, use as little as possible and test it thoroughly — "a single bug in the native code can corrupt your entire application."

### Panel-ready answer checklist
- Native methods are rarely justified for performance anymore — JVMs have closed most of that gap (BigInteger's Java reimplementation outperformed the native one).
- Real costs: memory corruption risk, reduced portability, harder debugging, GC/performance friction from untracked native memory, and tedious glue code.
- Legitimate uses remain: platform-specific facilities without a Java equivalent, and specialized native libraries with no Java counterpart (e.g., GMP for multiprecision arithmetic).
- If used: minimize the native surface area and test it exhaustively — a single native bug can corrupt the whole process.

---

## Item 67: Optimize judiciously

### Opening scenario
"A team spent two sprints hand-optimizing a module for speed before it even had correct behavior nailed down, and the result is both slower in the profiler and buggy. What's the actual failure here, structurally?"

### Follow-up probes
- What are Jackson's two rules of optimization, and who is "Rule 2" actually for?
- Give a concrete API-level example where a design decision (not an algorithm) permanently capped performance.
- Why is Java's "performance model" specifically harder to reason about than C/C++'s?
- What single step should come before writing a single line of optimization code?
- What's the one thing "no amount of low-level optimization" can ever fix?

### Naive attempt
Sacrifice clean architecture and API design early for speed, optimize based on intuition about where time is being spent, and skip measurement before/after changes.

### What breaks
"It is easy to do more harm than good, especially if you optimize prematurely" — you can end up with software that is neither fast nor correct and can't easily be fixed. A concrete cautionary example: `java.awt.Component.getSize()` returns a `Dimension`, and the decision to make `Dimension` mutable forces every call to allocate a new `Dimension` instance — a performance-limiting API decision baked in permanently, since preexisting client code still uses `getSize()` even after alternative methods were added later. More generally: attempted optimizations frequently have "no measurable effect on performance; sometimes, they make it worse," because it's difficult to guess where a program actually spends its time without measuring.

### The recommended approach & why
Strive to write good programs, not fast ones — speed follows from good architecture (information hiding, per Item 15), and a good program's design will allow later optimization if needed. But think about performance during design, especially for hard-to-change-later surfaces: APIs, wire-level protocols, and persistent data formats — these can place permanent limits on achievable performance (the `getSize()`/`Dimension` example). Once you have a clear, well-structured implementation, measure before deciding whether to optimize at all, and measure again after every attempted change — use a profiler (bigger codebase, more essential the "metal detector") and consider jmh for microbenchmarking. If a quadratic-or-worse algorithm is the bottleneck, no low-level tuning fixes it — you must replace the algorithm. Java's performance model is comparatively ill-defined and varies across implementations/releases/processors, making empirical measurement more essential than in C/C++, not less.

### Panel-ready answer checklist
- Jackson's rules: "Don't do it" / "(experts only) don't do it yet" — optimize only after a clear, correct, unoptimized solution exists.
- API/protocol/persistent-format decisions are the hardest to change later and can permanently cap performance (getSize()/mutable Dimension example).
- Measure before and after every optimization attempt — many "optimizations" have no effect or make things worse; use a profiler, consider jmh.
- No amount of low-level tuning fixes a bad algorithm — check algorithmic complexity first.

---

## Item 68: Adhere to generally accepted naming conventions

### Opening scenario
"A new hire submits a PR with a class named `HTTPURLConnectionMGR`, a constant field named `maxSize` (lowercase), and a package named `com.acme.Utils`. Before even reading the logic, what do you flag and why does it matter beyond aesthetics?"

### Follow-up probes
- Why does the book prefer `HttpUrl` over `HTTPURL` specifically?
- What's the single narrow exception to "constants use underscores"? What exactly qualifies a field as a "constant field"?
- A method returns a `boolean` — what naming pattern should it follow, and what's the alternative for non-boolean accessors?
- Type parameter conventions: what does `T`, `E`, `K`/`V`, `X`, and `R` each conventionally mean?
- Give the naming pattern difference between a method that converts to a new type versus one that returns a same-typed view.

### Naive attempt
Mix conventions ad hoc: package names with mixed case, constants without underscores, boolean-returning methods without an `is`/`has` prefix, acronyms in all-caps producing ambiguous runs like `HTTPURLConnection`.

### What breaks
Violating typographical conventions in an API "may be difficult to use"; violating them in an implementation "may be difficult to maintain." Either way, violations "have the potential to confuse and irritate other programmers" and cause faulty assumptions that lead to errors. Concretely, all-caps acronym runs are ambiguous about word boundaries — `HTTPURL` doesn't visibly separate "HTTP" from "URL" the way `HttpUrl` does.

### The recommended approach & why
Typographical conventions (rarely violate, never without very good reason): packages/modules lowercase, dot-separated, reverse-domain prefixed (`com.google`); classes/interfaces/enums/annotations `PascalCase` nouns (`FutureTask`), with acronyms capitalized only at their first letter (`HttpUrl`, not `HTTPURL`); methods/fields `camelCase` starting lowercase; **constant fields** (`static final` with a primitive or immutable reference type) use `ALL_CAPS_WITH_UNDERSCORES` — "the only recommended use of underscores"; local variables can be abbreviated (`i`, `denom`); type parameters are usually single letters: `T` arbitrary type, `E` collection element type, `K`/`V` map key/value, `X` exception, `R` function return type. Grammatical conventions (looser, more debated): instantiable classes get singular nouns (`Thread`), non-instantiable utility classes get plural nouns (`Collectors`), interfaces are nouns or `-able`/`-ible` adjectives (`Runnable`); boolean-returning methods use `is`/`has` prefixes (`isEmpty`, `hasSiblings`); non-boolean accessors are usually a bare noun/verb phrase (`size()`, `hashCode()`) rather than forced `getX()`, though the Beans-style `getX`/`setX` pair convention is acceptable, especially when both a getter and setter exist for the same attribute, or when tooling expects it. Special method-name patterns: `toType` converts to an independent object of a new type (`toString`); `asType` returns a view of a different type (`asList`); `typeValue` returns a same-value primitive (`intValue`); static factories favor `from`, `of`, `valueOf`, `instance`, `getInstance`, `newInstance`.

### Panel-ready answer checklist
- Capitalize only the first letter of acronyms in class/method names (`HttpUrl`, not `HTTPURL`) — improves word-boundary readability.
- Constant fields (`static final` + primitive or immutable reference type) are the sole sanctioned use of `ALL_CAPS_WITH_UNDERSCORES`.
- Boolean accessors: `is`/`has` prefix; non-boolean accessors: bare noun/verb phrase preferred over forced `getX`, though Beans-style `getX`/`setX` pairs are acceptable by convention.
- Type parameters follow single-letter conventions: `T`/`E`/`K`,`V`/`X`/`R` — deviating without reason reads as unconventional to reviewers.
- These are conventions, not laws — JLS itself says don't follow them "slavishly" against long-held conventional usage; use judgment.
