# Chapter 7: Lambdas and Streams (Items 42–48)

## Core Idea
Java 8's functional tools (lambdas, method references, streams) express behavior concisely — but they're precision instruments. Use them where they clarify; keep them side-effect-free; know when a loop or `Collection` return is still better.

## Frameworks Introduced
- **Item 42 — Prefer lambdas to anonymous classes**: for function objects (single-method interfaces). Lambdas are terse but lack names/`this`-to-self and shouldn't exceed a few lines. Keep them self-documenting.
- **Item 43 — Prefer method references to lambdas**: when a reference is clearer/shorter (`Integer::parseInt`, `Map::merge`, `String::toLowerCase`). Five kinds: static, bound instance, unbound instance, class constructor, array constructor. Use a lambda when it's actually clearer.
- **Item 44 — Favor the use of standard functional interfaces**: use `java.util.function` (`Function`, `Supplier`, `Consumer`, `Predicate`, `UnaryOperator`, `BinaryOperator`) instead of custom ones. Write your own only when it adds a meaningful contract; annotate `@FunctionalInterface`.
- **Item 45 — Use streams judiciously**: streams shine for transformations/filters/aggregations; overusing them harms readability. Don't force everything into one giant pipeline; name intermediate results.
- **Item 46 — Prefer side-effect-free functions in streams**: the essence of streams is functional. Do work in the *terminal* collector (`toList`, `groupingBy`, `toMap`, `joining`), not via `forEach` mutating external state. `forEach` is for reporting results, not computing them.
- **Item 47 — Prefer `Collection` to `Stream` as a return type**: a `Collection` (or subtype) serves both `for-each` and stream users; a bare `Stream` forces callers into streams and can't be iterated with for-each directly.
- **Item 48 — Use caution when making streams parallel**: `parallel()` rarely helps and often hurts/breaks correctness. Best on `ArrayList`/`array`/`IntStream.range`/`ConcurrentHashMap` (easily splittable, good locality); never on stateful or order-dependent pipelines.

## Key Concepts
- **Lambda**: anonymous function implementing a functional interface.
- **Functional interface**: interface with one abstract method (SAM).
- **Method reference**: `::` shorthand for a lambda that just calls one method.
- **Stream / pipeline**: source → intermediate ops (lazy) → terminal op (eager).
- **Collector**: recipe for the terminal reduction (`toList`, `groupingBy`, `toMap`, `joining`, `counting`).
- **Side-effect-free**: pure functions; no mutation of external state mid-pipeline.

## Mental Models
- Lambda when short and the interface is obvious; anonymous class when you need state, a name, or multiple methods.
- Read a method reference like a sentence — if it doesn't read cleaner than the lambda, keep the lambda.
- A stream pipeline should read as a *data transformation*, not a disguised loop with side effects.
- Return the *richest* type callers might want to iterate or stream — usually a `Collection`.
- Treat `parallel()` as an optimization you *measure*, never a default.

## Anti-patterns
- **Overlong lambdas** (many lines): extract to a named method.
- **Lambda where a method reference is clearer** (or vice versa): pick the readable one.
- **Custom functional interface** duplicating a standard one: use `java.util.function`.
- **`forEach` doing the computation** (mutating a map/counter): use a collector instead — it's the tell of "streams as loops."
- **Returning `Stream` from a public API**: callers can't `for-each` it; return `Collection`.
- **Blind `parallel()`**: on `Stream.iterate` or limit-bound pipelines it can run forever or corrupt results.

## Code Examples
```java
// Item 46 — side-effect-based (bad) vs collector-based (good)
// BAD: forEach mutates external map
Map<String,Long> freq = new HashMap<>();
words.forEach(w -> freq.merge(w.toLowerCase(), 1L, Long::sum));
// GOOD: computation lives in the terminal collector
Map<String,Long> freq2 = words.collect(groupingBy(String::toLowerCase, counting()));
```
- **What it demonstrates**: `forEach` is for *presenting* results; the actual reduction belongs in a `Collector`.

```java
// Item 45 — readable pipeline with named intermediates
List<String> topTen = freq.keySet().stream()
    .sorted(comparing(freq::get).reversed())
    .limit(10)
    .collect(toList());
```

## Reference Tables
| Standard functional interface | Signature | Use |
|---|---|---|
| `Predicate<T>` | `T -> boolean` | filter/test |
| `Function<T,R>` | `T -> R` | transform |
| `Supplier<T>` | `() -> T` | produce/lazy |
| `Consumer<T>` | `T -> void` | side-effect sink |
| `UnaryOperator<T>` | `T -> T` | same-type map |
| `BinaryOperator<T>` | `(T,T) -> T` | reduce/combine |

## Worked Example
Refactoring an anagram grouper (Item 45). The imperative version builds a `Map<String,Set<String>>` with a loop and `computeIfAbsent`. An *over*-streamed version crams alphabetization, grouping, and filtering into one unreadable pipeline with helper lambdas inline. Bloch's balanced version keeps a small named helper and a clear pipeline:
```java
try (Stream<String> words = Files.lines(dictionary)) {
    words.collect(groupingBy(word -> alphabetize(word)))  // helper named, not inlined
         .values().stream()
         .filter(group -> group.size() >= minGroupSize)
         .forEach(g -> System.out.println(g.size() + ": " + g)); // forEach only to REPORT
}
```
The lesson: streams *can* express this, but readability comes from naming the key-extraction (`alphabetize`) and using `forEach` only for output, not computation.

## Key Takeaways
1. Lambdas over anonymous classes; method references over lambdas — when each is clearer.
2. Reuse the standard `java.util.function` interfaces; annotate custom ones `@FunctionalInterface`.
3. Streams are for transformation pipelines; don't force-fit all logic into one — favor readability.
4. Keep stream functions pure; do the work in collectors, not `forEach` side effects.
5. Return `Collection`, not `Stream`, so callers can iterate *or* stream.
6. Parallelize only measured, splittable, side-effect-free pipelines.

## Connects To
- **Ch5 (Item 44)**: functional interfaces are generic.
- **Ch8 (Item 51)**: `Collection` return type = careful signature design.
- **Ch9 (Item 57)**: minimize scope — named intermediates aid clarity.
- **Ch11 (Item 80/84)**: parallel streams tie to concurrency caveats.
