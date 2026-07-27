# Chapter 7: Lambdas and Streams

## Item 42: Prefer lambdas to anonymous classes

### Opening scenario
"Here's some pre-Java-8 code using an anonymous `Comparator` class to sort a list of strings by length. Rewrite it however you'd write it today, and tell me what actually changed under the hood — not just the syntax."

### Follow-up probes
- Why does the compiler let you drop the parameter types (`s1, s2`) in the lambda but not always?
- When would you be forced to specify the types explicitly?
- Is a lambda literally an anonymous class instance? What's different about `this` in each?
- Can you serialize a lambda? What would you do instead if you needed a serializable function object?
- When would you still reach for an anonymous class instead of a lambda?
- Why shouldn't a lambda body span many lines?

### Naive attempt
Candidate keeps using anonymous classes everywhere out of habit, or overcorrects and crams multi-line logic into lambdas because "lambdas are the new way":
```java
Collections.sort(words, new Comparator<String>() {
    public int compare(String s1, String s2) {
        return Integer.compare(s1.length(), s2.length());
    }
});
```

### What breaks
- Boilerplate obscures intent; readability suffers when anonymous classes are used purely as function objects.
- Type inference for lambdas depends on generics: `words` declared as raw `List` instead of `List<String>` means the snippet won't compile as a lambda — the compiler loses the type information it needs.
- Long or complex lambda bodies (more than ~3 lines) harm readability — "one line is ideal... three lines is reasonable maximum."
- Lambdas in enum constructors can't access instance members of the enum (constructor arguments are evaluated in a static context) — a candidate who tries this will hit a compile error.
- Assuming `this` inside a lambda refers to the lambda itself — it refers to the enclosing instance, unlike an anonymous class where `this` refers to the anonymous instance.
- Trying to serialize a lambda across JVM implementations — not reliable, same limitation as anonymous class instances.

### The recommended approach & why
As of Java 8, lambdas are by far the best way to represent small function objects. Interfaces with a single abstract method are "functional interfaces," and lambdas instantiate them concisely — e.g. `Collections.sort(words, (s1, s2) -> Integer.compare(s1.length(), s2.length()));`, shortenable further with method references: `words.sort(comparingInt(String::length));`.

Lambdas also make constant-specific enum behavior clean: store a `DoubleBinaryOperator` (or other functional interface) as an instance field per constant instead of overriding an abstract method with constant-specific class bodies — simpler and clearer than the classic pattern, but only when the logic is short and doesn't need instance state.

Anonymous classes remain necessary for: instantiating abstract classes (not just interfaces), interfaces with multiple abstract methods, and any case where the function object needs a reference to itself via `this`. Favor generic types/methods and avoid raw types when using lambdas, since the compiler depends on generic type information to perform type inference — omitting it forces manual type annotations and destroys the conciseness lambdas are for.

### Panel-ready answer checklist
- Lambdas are how you instantiate functional interfaces (single abstract method) since Java 8; far more concise than anonymous classes.
- Favor generics/parameterized types — raw types break lambda type inference.
- One line ideal, three lines max for a lambda body; anything longer should be refactored out.
- Lambdas can't self-reference via `this` (refers to enclosing instance) and can't be reliably serialized — anonymous classes still needed for abstract classes, multi-method interfaces, and self-reference.
- Enum constructors are evaluated in a static context — lambdas passed to them can't touch instance members.

---

## Item 43: Prefer method references to lambdas

### Opening scenario
"You see `map.merge(key, 1, (count, incr) -> count + incr);` in a PR. The reviewer says 'just use a method reference.' What would they write, and is that always the right call?"

### Follow-up probes
- Is there anything a method reference can do that a lambda can't, or vice versa?
- Give an example where the lambda is actually clearer than the method reference.
- What are the five kinds of method references? Give one example of each.
- Why might an IDE's "convert to method reference" suggestion sometimes make code worse?
- What's `Function.identity()` for, and why might a raw lambda `x -> x` sometimes be preferred anyway?

### Naive attempt
Sticking with a verbose lambda when a method reference already exists for exactly that operation:
```java
map.merge(key, 1, (count, incr) -> count + incr);
```

### What breaks
- Nothing breaks functionally — the failure mode here is stylistic/readability, not correctness. The parameters `count` and `incr` add visual clutter without adding meaning since the lambda is just delegating to a known static method.
- Conversely, blindly accepting every IDE method-reference suggestion can backfire: when the referenced method lives in the same (verbosely named) class, e.g. `service.execute(GoshThisClassNameIsHumongous::action);` vs. `service.execute(() -> action());` — the lambda is shorter and clearer here.

### The recommended approach & why
Use `Integer::sum` in place of `(count, incr) -> count + incr` — `map.merge(key, 1, Integer::sum);`. The more parameters a method has, the more boilerplate a method reference eliminates. There is nothing a method reference can do that a lambda can't (with one obscure exception per JLS 9.9-2), but method references usually produce shorter, clearer code, and provide an escape hatch: extract lambda logic into a well-named method and reference it.

The exception: when the lambda would just wrap a call to a method in the same class, especially with a long class name, the lambda reads better. Also prefer a bare lambda `x -> x` over `Function.identity()` when it's inline and equally clear.

Five kinds of method references (memorize the table):

| Method Ref Type | Example | Lambda Equivalent |
|---|---|---|
| Static | `Integer::parseInt` | `str -> Integer.parseInt(str)` |
| Bound | `Instant.now()::isAfter` | `Instant then = Instant.now(); t -> then.isAfter(t)` |
| Unbound | `String::toLowerCase` | `str -> str.toLowerCase()` |
| Class Constructor | `TreeMap<K,V>::new` | `() -> new TreeMap<K,V>()` |
| Array Constructor | `int[]::new` | `len -> new int[len]` |

### Panel-ready answer checklist
- Method references are the more succinct alternative to lambdas — use them where they're shorter and clearer.
- No functional gap between the two (with one obscure JLS exception); it's a readability call, item by item.
- Prefer the lambda when the method reference would be no shorter/clearer — e.g., calling a method in a class with a long name.
- Know the five kinds: static, bound instance, unbound instance, class constructor, array constructor.
- Bound vs. unbound distinction: bound captures the receiver at reference-creation time; unbound takes the receiver as the first parameter when invoked.

---

## Item 44: Favor the use of standard functional interfaces

### Opening scenario
"You need `LinkedHashMap` to evict entries once it exceeds 100 items, so you're adding cache behavior via a factory method that takes a function object instead of overriding `removeEldestEntry`. Someone proposes writing a custom `EldestEntryRemovalFunction` interface for this. Good idea?"

### Follow-up probes
- Why can't the function object just take `Map.Entry<K,V>` and return `boolean`? What extra input does it need, and why?
- Name the six basic functional interfaces in `java.util.function` from memory, with their signatures.
- Why does `Comparator<T>` deserve its own interface instead of reusing `ToIntBiFunction<T,T>`, which is structurally identical?
- What does `@FunctionalInterface` actually enforce, and why annotate every one?
- Why shouldn't you overload a method to take two different functional interfaces in the same argument position?
- Why avoid boxed-primitive versions of these interfaces (e.g., `Function<Integer,Integer>`) in bulk operations?

### Naive attempt
```java
// Unnecessary functional interface; use standard one instead.
@FunctionalInterface interface EldestEntryRemovalFunction<K,V>{
    boolean remove(Map<K,V> map, Map.Entry<K,V> eldest);
}
```

### What breaks
- Reinventing an interface that already exists in `java.util.function` (here, `BiPredicate<Map<K,V>, Map.Entry<K,V>>`) needlessly enlarges the API's conceptual surface area and forfeits the default methods (e.g., `Predicate`'s `and`/`or`/`negate`) that come free with standard interfaces.
- Using a basic functional interface with boxed primitives instead of the primitive-specialized variant (`IntPredicate`, `LongBinaryOperator`, etc.) works but incurs the performance cost of boxing in bulk operations (ties to Item 61: prefer primitives to boxed primitives).
- Overloading a method to accept, say, `Callable<T>` or `Runnable` in the same parameter slot (as `ExecutorService.submit` does) creates real ambiguity — client code sometimes needs an explicit cast to select the right overload (Item 52).

### The recommended approach & why
If a standard functional interface does the job, use it instead of a purpose-built one — it reduces API surface area and buys default methods for free. Know the six basic interfaces:

| Interface | Function Signature | Example |
|---|---|---|
| `UnaryOperator<T>` | `T apply(T t)` | `String::toLowerCase` |
| `BinaryOperator<T>` | `T apply(T t1, T t2)` | `BigInteger::add` |
| `Predicate<T>` | `boolean test(T t)` | `Collection::isEmpty` |
| `Function<T,R>` | `R apply(T t)` | `Arrays::asList` |
| `Supplier<T>` | `T get()` | `Instant::now` |
| `Consumer<T>` | `void accept(T t)` | `System.out::println` |

Of 43 total standard interfaces: three primitive variants of each basic interface (`int`/`long`/`double`), nine additional `Function` variants for primitive-to-primitive or primitive-to-object conversions, `BiPredicate`/`BiFunction`/`BiConsumer` two-arg forms plus their primitive-returning/primitive-accepting variants, and `BooleanSupplier`.

Write your own functional interface only when none of the standard ones fit, or when the interface (like `Comparator<T>`) has one or more of: a commonly-used, descriptive name; a strong contract; custom default methods that would benefit from it. Comparator is the canonical example that "deserved" its own interface despite being structurally equivalent to `ToIntBiFunction<T,T>`.

Always annotate a hand-written functional interface with `@FunctionalInterface`: it documents intent, forces a compile error if the interface doesn't have exactly one abstract method, and prevents future maintainers from accidentally breaking the contract by adding a second abstract method.

Never overload a method with two different functional-interface parameter types in the same position — it's a special case of Item 52 (use overloading judiciously) and creates genuine ambiguity requiring casts to disambiguate.

### Panel-ready answer checklist
- Default to `java.util.function`'s 43 standard interfaces before writing a custom one — smaller API surface, free default methods.
- Know the six basics cold: `UnaryOperator`, `BinaryOperator`, `Predicate`, `Function`, `Supplier`, `Consumer`.
- Write a custom interface only when it needs a descriptive name, a strong contract, or custom default methods (Comparator's justification).
- Always tag custom functional interfaces `@FunctionalInterface`.
- Avoid boxed-primitive basic interfaces in hot paths — use the primitive-specialized variant.
- Never overload on two different functional-interface types in the same argument slot — it forces awkward casts on callers.

---

## Item 45: Use streams judiciously

### Opening scenario
"You've rewritten an anagram-grouping program to squeeze the entire `main` method — file reading, grouping, filtering, printing — into one chained stream expression. A teammate says it's unreadable. Are they wrong just because they're not comfortable with streams?"

### Follow-up probes
- What's the actual failure mode of the "everything in one expression" version — is it a correctness bug or something else?
- Streams process `"Hello world!".chars()` — what does `forEach(System.out::print)` actually print, and why is that surprising?
- What can code blocks (for-loops) do that lambdas passed to stream operations fundamentally cannot?
- When is a stream-based solution clearly worse even though it "works," e.g. for the Cartesian product / card deck example?
- Why does naming a stream-returning method with a plural noun matter?
- Streams are lazy — what does that actually buy you, and what's the risk of a pipeline with no terminal operation?

### Naive attempt
```java
// Overuse streams - don't do this!
public class Anagrams {
    public static void main(String[] args) throws IOException {
        Path dictionary = Paths.get(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);
        try (Stream<String> words = Files.lines(dictionary)) {
            words.collect(
                groupingBy(word -> word.chars().sorted()
                    .collect(StringBuilder::new, (sb, c) -> sb.append((char) c),
                             StringBuilder::append).toString()))
                .values().stream()
                .filter(group -> group.size() >= minGroupSize)
                .map(group -> group.size() + ": " + group)
                .forEach(System.out::println);
        }
    }
}
```

### What breaks
- Technically correct, but "shorter... also less readable, especially to programmers who are not experts in the use of streams" — overusing streams harms readability and maintainability even without any bug.
- Using streams to process `char` values is a specific trap: `"Hello world!".chars().forEach(System.out::print)` prints `721011081081113211911111410810033`, not `Hello world!` — because `chars()` returns a stream of `int`, so the `int` overload of `print` is invoked, not the `char` one.
- Trying to force everything into streams hits a hard capability wall: from inside a lambda you can only read final/effectively-final local variables (never modify them), and you cannot `return` from the enclosing method, `break`/`continue` an enclosing loop, or throw a checked exception the method didn't declare.
- Once a value is `map`ped away in a pipeline, the original value is gone — accessing values from multiple pipeline stages simultaneously (e.g., needing both a Mersenne prime and its exponent `p`) isn't directly supported; the fix is to invert the mapping in a later stage rather than tuple everything together.

### The recommended approach & why
There are no hard rules, only heuristics. A "tasteful" middle ground uses streams for what they're good at and keeps iteration (or helper methods) for the rest:
```java
// Tasteful use of streams enhances clarity and conciseness
try (Stream<String> words = Files.lines(dictionary)) {
    words.collect(groupingBy(word -> alphabetize(word)))
         .values().stream()
         .filter(group -> group.size() >= minGroupSize)
         .forEach(g -> System.out.println(g.size() + ": " + g));
}
```
Extracting `alphabetize` into a named helper method is itself the recommendation — helper methods matter more for readability in streams than in iterative code because pipelines lack explicit type declarations and named temporaries.

Streams are a good fit for: uniformly transforming a sequence, filtering a sequence, combining a sequence with a single operation (sum, concatenation, min/max), accumulating into a collection (possibly grouped), and searching for a matching element. Streams are a poor fit whenever the computation needs to modify local variables, or needs to `return`/`break`/`continue`/throw a checked exception from within the loop body — that's the signal to stick with (or fall back to) iteration.

When choosing between an iterative and stream-based implementation of similar clarity (e.g., Cartesian product of two enums via `flatMap`), the book explicitly calls it "personal preference": default to the iterative version if you're unsure, since a broader set of programmers can maintain it.

### Panel-ready answer checklist
- No hard rule — heuristic-driven choice; refactor existing code to streams and use streams in new code only where it clearly helps.
- Streams excel at: transform, filter, combine (reduce), collect/group, search — one linear pipeline.
- Streams are a poor fit when the logic needs mutable local state, non-local control flow (`return`/`break`/`continue`), or checked exceptions.
- `chars()` on a `String` returns `int`, not `char` — a well-known gotcha that argues against processing char streams at all.
- Once mapped away, an earlier stage's value is lost — invert the mapping in the terminal stage to recover it, rather than smuggling a pair/tuple through the pipeline.
- Name stream-returning methods with a plural noun (e.g., `primes()`) to keep pipelines self-documenting.

---

## Item 46: Prefer side-effect-free functions in streams

### Opening scenario
"Someone wrote a stream pipeline using `forEach` to mutate an external map — a word-frequency counter — instead of collecting. What's wrong with this, given that it uses streams, lambdas, and gets the right answer?"

### Follow-up probes
- If it compiles and produces the correct frequency table, what's actually the problem?
- What is a "pure function" in this context, and why does the streams paradigm depend on it?
- Is `forEach` ever legitimately used for more than printing/reporting a result?
- Name the three basic collectors for gathering into a true `Collection`. When would you reach for each?
- Walk through what `toMap(keyMapper, valueMapper)` does when two elements map to the same key — what happens, and how do you control it?
- What's the difference between `groupingBy`'s classifier and its downstream collector?

### Naive attempt
```java
// Uses the streams API but not the paradigm--Don't do this!
Map<String, Long> freq = new HashMap<>();
try (Stream<String> words = new Scanner(file).tokens()) {
    words.forEach(word -> {
        freq.merge(word.toLowerCase(), 1L, Long::sum);
    });
}
```

### What breaks
- This is "iterative code masquerading as streams code" — it derives no benefit from the streams API, and is actually longer, harder to read, and less maintainable than the plain iterative equivalent.
- The root problem: all the work is done inside a terminal `forEach` using a lambda that mutates external state (the `freq` map). A `forEach` that does more than report a stream's result is explicitly called out as "a bad smell in code," as is any lambda that mutates state.
- `forEach` is described as "among the least powerful of the terminal operations and the least stream-friendly" — it's explicitly iterative and therefore not amenable to parallelization.

### The recommended approach & why
Replace side-effecting `forEach` with a proper collector:
```java
// Proper use of streams to initialize a frequency table
Map<String, Long> freq;
try (Stream<String> words = new Scanner(file).tokens()) {
    freq = words.collect(groupingBy(String::toLowerCase, counting()));
}
```
The core principle: structure computation as a sequence of transformations where each stage's result is as close as possible to a pure function of the previous stage's result — no dependence on or mutation of external mutable state.

Key collector factories to know:
- `toList()`, `toSet()`, `toCollection(collectionFactory)` — gather into a true `Collection`.
- `toMap(keyMapper, valueMapper)` — throws `IllegalStateException` on key collisions unless you use the 3-arg form with a `BinaryOperator<V>` merge function (e.g., last-write-wins via `(v1, v2) -> v2`, or picking a "best" element via `maxBy(comparing(...))`). A 4-arg form lets you specify the map implementation (e.g. `TreeMap`, `EnumMap`).
- `groupingBy(classifier)` — groups into a `Map` of `List`s by category; add a downstream collector (`toSet()`, `counting()`, `toCollection(...)`) to change the value shape; a 3-arg form adds a map factory (note: mapFactory precedes downstream, breaking the usual telescoping argument order).
- `toConcurrentMap`/`groupingByConcurrent` — parallel-friendly variants producing `ConcurrentHashMap`.
- `joining` — concatenates `CharSequence` elements, optionally with a delimiter and prefix/suffix.
- Statically import `Collectors` members — customary and improves pipeline readability (e.g., bare `toList()` instead of `Collectors.toList()`).

Ignore most of the other ~15 Collectors methods (`summing*`, `averaging*`, `reducing`, `filtering`, `mapping`, `collectingAndThen`) unless you have a specific need — their functionality often duplicates something already available directly on `Stream` (e.g., never write `collect(counting())` — just call `.count()`).

### Panel-ready answer checklist
- The essence of stream pipelines is side-effect-free function objects — pure functions of the prior stage's output.
- `forEach` that mutates external state is a "bad smell" — it's iterative code pretending to be a stream pipeline, and it's the least parallelizable terminal operation.
- `forEach` should be used only to report a result already computed by the stream.
- Know the core collector factories cold: `toList`, `toSet`, `toMap` (with merge-function overloads for collisions), `groupingBy` (with downstream collectors), `joining`.
- `toMap`'s 2-arg form throws `IllegalStateException` on duplicate keys — reach for the 3-arg merge-function form to handle collisions deliberately.

---

## Item 47: Prefer Collection to Stream as a return type

### Opening scenario
"You're designing a public API method that returns a sequence of elements. A teammate says 'just return a `Stream<T>` — it's the modern way.' What's the problem with committing to that as the return type?"

### Follow-up probes
- Why can't a caller just use a for-each loop directly over a `Stream`?
- Why doesn't casting a method reference to `Iterable` compile cleanly — what's the actual compiler error?
- What would an adapter method between `Stream` and `Iterable` look like, in both directions?
- If the sequence is huge (say, a power set), why can't you just materialize it into an `ArrayList` and return that?
- What's the size-limitation trade-off you accept by returning a `Collection` instead of a `Stream`/`Iterable`?
- When is it fine to just return a `Stream` and not worry about iteration users?

### Naive attempt
Return a `Stream<E>` from a public API and assume callers who want a for-each loop can just cast their way around it:
```java
// Won't compile, due to limitations on Java's type inference
for (ProcessHandle ph : ProcessHandle.allProcesses()::iterator) {
    // Process the process
}
```

### What breaks
- The above doesn't compile: `error: method reference not expected here`. Fixing it requires an ugly cast: `for (ProcessHandle ph : (Iterable<ProcessHandle>) ProcessHandle.allProcesses()::iterator)` — "too noisy and opaque to use in practice."
- `Stream` doesn't extend `Iterable`, even though `Stream`'s sole abstract method (`iterator`) is spec-compatible with `Iterable`'s — there's no clean built-in bridge, and the JDK provides no adapter.
- Returning only a `Stream` justifiably upsets users who want to iterate; returning only an `Iterable` justifiably upsets users who want to build a stream pipeline. An API that forces one camp to write a bridging adapter (which also has real overhead — an adapter measurably slowed a loop by 2.3x in the author's benchmark) is a worse API than one that just returns `Collection`.
- Naively materializing a huge or infinite sequence (e.g. a power set of size 2^n, or all sublists of a list — quadratic in list size) into a standard collection like `ArrayList` wastes enormous memory, or is outright infeasible.

### The recommended approach & why
`Collection` is a subtype of `Iterable` and has a `stream()` method, so it serves both camps at once — "Collection or an appropriate subtype is generally the best return type for a public, sequence-returning method." If the sequence fits comfortably in memory, return a standard implementation like `ArrayList` or `HashSet`. If it's large but can be represented concisely (like a power set), implement a custom `Collection` (e.g., atop `AbstractList`, encoding each subset via a bit vector over its index) rather than storing every element:
```java
// Returns the power set of an input set as custom collection
public class PowerSet {
    public static final <E> Collection<Set<E>> of(Set<E> s) {
        List<E> src = new ArrayList<>(s);
        if (src.size() > 30)
            throw new IllegalArgumentException("Set too big " + s);
        return new AbstractList<Set<E>>() {
            @Override public int size() { return 1 << src.size(); }
            @Override public boolean contains(Object o) {
                return o instanceof Set && src.containsAll((Set) o);
            }
            @Override public Set<E> get(int index) {
                Set<E> result = new HashSet<>();
                for (int i = 0; index != 0; i++, index >>= 1)
                    if ((index & 1) == 1) result.add(src.get(i));
                return result;
            }
        };
    }
}
```
Note the accepted trade-off: `Collection.size()` returns `int`, capping representable length at `Integer.MAX_VALUE` (2^31 - 1) — hence the explicit `size() > 30` guard above (2^31 elements would overflow).

If materializing isn't feasible even as a custom collection (e.g., sublists of a list — quadratic and awkward to encode losslessly), return whichever of `Stream` or `Iterable` feels more natural, and accept that some users will need a bridging adapter:
```java
// Adapter from Stream<E> to Iterable<E>
public static <E> Iterable<E> iterableOf(Stream<E> stream) {
    return stream::iterator;
}

// Adapter from Iterable<E> to Stream<E>
public static <E> Stream<E> streamOf(Iterable<E> iterable) {
    return StreamSupport.stream(iterable.spliterator(), false);
}
```
If you know for certain a method's result will only ever be used in a stream pipeline (or only ever iterated), it's fine to return `Stream` (or `Iterable`) directly and skip the accommodation.

### Panel-ready answer checklist
- `Stream` doesn't extend `Iterable`, so returning only a `Stream` forces for-each users into an ugly cast or a bridging adapter.
- `Collection` is the generally-best public return type — it's both `Iterable` and has `.stream()`, satisfying both camps at once.
- For huge-but-concisely-representable sequences (power set), implement a custom `Collection` rather than materializing every element — but note `size()` returning `int` caps it at `Integer.MAX_VALUE`.
- If it's infeasible to represent as a collection at all, return `Stream` or `Iterable`, whichever is more natural, and accept some users need an adapter (which has real runtime overhead).
- Returning only a `Stream`/`Iterable` when you know all callers use it one way is fine — the accommodation is only a public-API concern with unpredictable callers.

---

## Item 48: Use caution when making streams parallel

### Opening scenario
"A candidate parallelized a stream over a `Stream.iterate()` source with a `limit()` — computing the first 20 Mersenne primes — hoping for a speedup. Instead, it hung: CPU pegged at 90%, nothing printed even after 30 minutes. Why?"

### Follow-up probes
- What do `ArrayList`, `HashMap`, arrays, and int/long ranges have in common that makes them parallelize well, and why does `Stream.iterate` lack it?
- What's a spliterator, and why does writing a good one matter for custom `Stream`/`Iterable`/`Collection` types?
- Why does `limit()` make the parallelization problem worse, specifically for this Mersenne-prime pipeline?
- What kinds of terminal operations parallelize well versus poorly, and why is `collect()` (a "mutable reduction") a bad candidate?
- If a sequential pipeline is correct but you parallelize it and it starts producing wrong answers, what's really going on?
- Why is `SplittableRandom` preferred over `ThreadLocalRandom` or `Random` for a parallel random-number stream?
- Given how bad the odds sound, is there ever a case where `.parallel()` genuinely helps?

### Naive attempt
```java
public static void main(String[] args) {
    primes().map(p -> TWO.pow(p.intValueExact()).subtract(ONE))
            .filter(mersenne -> mersenne.isProbablePrime(50))
            .limit(20)
            .parallel()
            .forEach(System.out::println);
}
static Stream<BigInteger> primes() {
    return Stream.iterate(TWO, BigInteger::nextProbablePrime);
}
```

### What breaks
- This is a liveness failure: CPU spikes to 90% and nothing is ever printed — the author stopped it forcibly after 30 minutes without it finishing.
- The streams library "has no idea how to parallelize this pipeline" — parallelizing is unlikely to help when the source is `Stream.iterate` (not efficiently splittable) or when `limit` is used, and this pipeline has both problems simultaneously.
- The default strategy for handling `limit`'s unpredictability is to compute a few extra elements and discard the unneeded ones — but here each Mersenne prime costs roughly twice as long to find as the previous one, so computing "a few extra" costs as much as everything computed so far combined, bringing the whole parallelization scheme to its knees.
- Even when parallelization doesn't hang, it can silently produce wrong answers (a safety failure): the `Stream` spec requires accumulator/combiner functions passed to `reduce` (and similar) to be associative, non-interfering, and stateless. Violating this while running sequentially will likely still give correct results by luck; parallelizing the same violation will likely fail, possibly badly.
- Parallelizing also reorders output — `forEach` on a parallel stream does not preserve encounter order; you'd need `forEachOrdered` to keep the original (ascending) print order, at some cost to parallelism's benefit.

### The recommended approach & why
Do not parallelize a stream pipeline unless you have good reason to believe it will both preserve correctness and increase speed — treat it strictly as a performance optimization requiring before/after measurement (Item 67), ideally under realistic conditions, since all parallel pipelines in a program share one common fork-join pool and a single misbehaving pipeline can hurt unrelated code.

Parallelism performs best on sources that split cheaply and accurately into subranges — `ArrayList`, `HashMap`, `HashSet`, `ConcurrentHashMap`, arrays, and `int`/`long` ranges — because the streams library relies on `Spliterator` to divide work, and these sources also have good locality of reference (primitive arrays being the best case, since the data itself sits contiguously in memory) which keeps threads fed instead of idling on cache misses.

The terminal operation matters too: reductions (`reduce`, or prepackaged ones like `min`, `max`, `count`, `sum`) and short-circuiting operations (`anyMatch`, `allMatch`, `noneMatch`) parallelize well. `collect()`-based "mutable reductions" are poor candidates — the overhead of combining collections across threads is costly.

As a rough sizing heuristic: elements × lines-of-code-executed-per-element should be at least about 100,000 [Lea14] before parallelism is likely worth it. A counter-example that does work well:
```java
// Prime-counting stream pipeline - parallel version
static long pi(long n) {
    return LongStream.rangeClosed(2, n)
            .parallel()
            .mapToObj(BigInteger::valueOf)
            .filter(i -> i.isProbablePrime(50))
            .count();
}
```
This gave a 3.7x speedup on a quad-core machine (31s to 9.2s) — because the source is a cheaply-splittable `long` range and the terminal operation is `count()`, a genuine reduction. For parallel streams of random numbers, start from `SplittableRandom` rather than `ThreadLocalRandom` (single-thread-oriented, adapts but is slower) or `Random` (synchronizes on every call, killing parallelism via contention).

If you write your own `Stream`/`Iterable`/`Collection` and want it to parallelize well, you must override `spliterator()` and extensively test parallel performance — writing a high-quality spliterator is nontrivial.

### Panel-ready answer checklist
- Default posture: don't parallelize a stream pipeline without a specific, tested reason — it's an optimization, verify with before/after measurements under realistic conditions.
- Good parallel sources split cheaply and have good locality of reference: `ArrayList`, arrays, `HashMap`/`HashSet`, `ConcurrentHashMap`, `int`/`long` ranges. `Stream.iterate` and `limit` are red flags.
- Good parallel terminal operations: reductions (`reduce`, `min`/`max`/`count`/`sum`) and short-circuiting matches. `collect()` (mutable reduction) is a poor fit.
- Parallelizing a pipeline whose function objects violate the associative/non-interfering/stateless contract can silently produce wrong answers under parallel execution even though it worked sequentially.
- Rough sizing heuristic: elements × per-element work should be roughly ≥100,000 before parallelism pays off.
- Use `SplittableRandom` for parallel random-number streams, not `ThreadLocalRandom`/`Random`.
- Use `forEachOrdered` instead of `forEach` if encounter order must be preserved on a parallel stream.
