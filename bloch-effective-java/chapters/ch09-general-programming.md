# Chapter 9: General Programming (Items 57–68)

## Core Idea
The day-to-day mechanics: scope things tightly, use the right library and type, avoid premature optimization, and follow conventions. Small, boring disciplines that compound into robust code.

## Frameworks Introduced
- **Item 57 — Minimize the scope of local variables**: declare where first used, with an initializer; prefer `for` (loop var scoped to loop) over `while`; keep methods small and focused.
- **Item 58 — Prefer for-each loops to traditional for loops**: clearer, no index/iterator errors. Can't use it only when you need the index, to modify via the iterator, or to iterate multiple collections in parallel.
- **Item 59 — Know and use the libraries**: don't reinvent (`Random` → `ThreadLocalRandom`; `Collections`, `Files`, etc.). Standard libraries are expert-written, tested, and maintained.
- **Item 60 — Avoid `float` and `double` if exact answers are required**: they're binary approximations; use `BigDecimal`, `int`, or `long` for money and exact arithmetic.
- **Item 61 — Prefer primitive types to boxed primitives**: boxed types have identity (`==` compares references!), allow `null` (auto-unboxing NPE), and cost performance. Use primitives unless you need collection elements, generics, or reflection.
- **Item 62 — Avoid strings where other types are more appropriate**: strings are poor substitutes for numbers, enums, aggregate types, and capabilities/keys.
- **Item 63 — Beware the performance of string concatenation**: `+` in a loop is O(n²); use `StringBuilder`.
- **Item 64 — Refer to objects by their interfaces**: declare variables/params/returns with interface types (`List` not `ArrayList`) for flexibility. Use the class type when no suitable interface exists.
- **Item 65 — Prefer interfaces to reflection**: reflection is powerful but costly, unsafe, and verbose; when you must use it, do so only to instantiate, then access via a known interface/superclass.
- **Item 66 — Use native methods judiciously**: JNI rarely pays off now; it's unsafe and unportable.
- **Item 67 — Optimize judiciously**: "premature optimization is the root of all evil." Design for good structure/APIs first; measure before and after; don't sacrifice sound architecture for speed guesses.
- **Item 68 — Adhere to generally accepted naming conventions**: typographical (`ClassName`, `methodName`, `CONSTANT`, `packagename`) and grammatical (nouns for classes, verbs for action methods, `is`/`has` for booleans).

## Key Concepts
- **Scope minimization**: narrowest lifetime for each variable.
- **Autoboxing/unboxing**: implicit primitive↔wrapper conversion — source of NPEs and silent slowdowns.
- **Interface-typed references**: program to interfaces for substitutability.
- **Premature optimization**: tuning before measuring, at the cost of clarity/design.
- **Naming conventions**: shared vocabulary making code readable across teams.

## Mental Models
- Declare a variable only when you have something to put in it, and let it die as soon as possible.
- Reach for the library first; assume the JDK already solved it well.
- Money and exactness ⇒ never `double`. Loop concatenation ⇒ always `StringBuilder`.
- "Measure, don't guess" — profile before optimizing; the bottleneck is rarely where you think.
- Type your variables by capability (interface), not implementation.

## Anti-patterns
- **Wide-scope / uninitialized locals**: bugs from stale values and misuse.
- **Indexed `for` where `for-each` fits**: off-by-one and wrong-iterator errors.
- **Rolling your own** date math, RNG, sorting: use the library.
- **`double` for money**: `1.03 - 0.42 == 0.6100000000000001`.
- **Boxed primitives in hot paths / with `==`**: `Long sum` in a loop = orders of magnitude slower; `Integer a == Integer b` compares identity.
- **String-typed enums/keys/aggregates**: stringly-typed code loses safety.
- **`+=` string concatenation in loops**: quadratic.
- **Declaring `ArrayList x` instead of `List x`**: locks the implementation.
- **Optimizing before measuring**: wasted effort, damaged design.

## Code Examples
```java
// Item 61 — the boxed-primitive performance trap
Long sum = 0L;                       // BOXED — allocates ~2^31 Long objects
for (long i = 0; i < Integer.MAX_VALUE; i++) sum += i;  // auto-box/unbox every iteration
// Fix: declare `long sum = 0L;` — a single primitive, ~6x+ faster.

// Item 61 — the == identity trap
Integer a = 1000, b = 1000;
System.out.println(a == b);          // false! compares references, not values
```
- **What it demonstrates**: boxed primitives silently kill performance and break `==` value comparison.

```java
// Item 63 — quadratic vs linear concatenation
String s = "";
for (String line : lines) s += line;           // O(n^2)
StringBuilder sb = new StringBuilder();
for (String line : lines) sb.append(line);     // O(n)
```

## Reference Tables
| Task | Prefer | Avoid |
|---|---|---|
| Exact/money math | `BigDecimal`, `int`, `long` | `float`, `double` |
| Numeric locals | primitive | boxed wrapper |
| String building in loop | `StringBuilder` | `+=` |
| Variable/param type | interface (`List`) | impl (`ArrayList`) |
| RNG | `ThreadLocalRandom` | `new Random()` per call |
| Iteration | `for-each` | index unless needed |

## Worked Example
The autounboxing NPE (Item 61). This comparator looks fine but throws on unboxing when the second `Integer` is involved:
```java
Comparator<Integer> naturalOrder = (i, j) ->
    (i < j) ? -1 : (i == j ? 0 : 1);   // i == j compares Integer IDENTITY
// naturalOrder.compare(new Integer(42), new Integer(42)) returns 1, not 0!
```
`i < j` auto-unboxes (fine), but `i == j` compares references and is false for distinct boxed objects. Fix by using primitives explicitly:
```java
Comparator<Integer> naturalOrder = (iBoxed, jBoxed) -> {
    int i = iBoxed, j = jBoxed;                  // unbox once, explicitly
    return (i < j) ? -1 : (i == j ? 0 : 1);
};
```
Lesson: whenever a boxed and unboxed type mix in an operation, the box is auto-unboxed — a `null` box there is an instant NPE, and `==` on two boxes is identity, not value.

## Key Takeaways
1. Declare locals late, narrow, and initialized; prefer `for-each`.
2. Learn and lean on the standard libraries before writing your own.
3. `double`/`float` are approximations — use exact types for money.
4. Prefer primitives; boxed primitives break `==` and wreck loop performance.
5. Don't stringly-type; don't concatenate strings in loops.
6. Program to interfaces; use reflection/native/optimization only with strong justification and measurement.

## Connects To
- **Ch2 (Item 6)**: unnecessary boxing = unnecessary objects.
- **Ch4 (Item 20/64)**: interface references enable substitutability.
- **Ch7 (Item 45)**: know the library = know Streams/Collectors.
- **Ch11 (Item 78/80)**: `ThreadLocalRandom`, executors are library concurrency tools.
