# Chapter 2: Creating and Destroying Objects

## Item 1: Consider static factory methods instead of constructors

### Opening scenario
"Here's a class `BigInteger` that needs a constructor `BigInteger(int, int, Random)` to return a probably-prime number. Also, I need two ways to build a `Currency`-like value object, and both constructors would take `(int, int)` — one for `(dollars, cents)`, one for `(euros, centimes)`. How do you design the API?"

### Follow-up probes
- Why not just add a boolean flag parameter to disambiguate the two constructors instead of writing separate methods?
- If `Boolean.valueOf(boolean)` never allocates, how does that interact with `==` comparisons — is that safe to rely on?
- Your service returns different concrete collection implementations depending on size. How would a constructor-based API even express that?
- What's the cost of choosing this approach — what can a client no longer do that they could do with a public constructor?
- If this is so much better, why does `Boolean` still also have a public constructor (pre-Java 9)?

### Naive attempt
Candidate proposes two overloaded constructors differing only in parameter order, or a single constructor with a `boolean isEuro` flag: `new Money(int, int, boolean)`.

### What breaks
Two constructors with reordered parameter types of the same arity are indistinguishable at the call site without reading class docs — callers will eventually transpose arguments and get a compiling, silently-wrong program (e.g., pass euros where dollars expected). The boolean-flag approach means every call site is a puzzle: `new Money(240, 8, true)` — true meaning what? Neither approach scales as more variants are needed, and neither documents intent in the API surface itself.

### The recommended approach & why
Use a `public static factory method` — a static method returning an instance — instead of, or alongside, constructors: `BigInteger.probablePrime(...)`, `Money.euros(int, int)`, `Money.dollars(int, int)`. Mechanism and payoff, per Bloch:
1. **They have names** — unlike constructors, so intent is legible at the call site (`BigInteger.probablePrime`).
2. **Not required to create a new object each call** — enables caching/reuse of preconstructed instances (`Boolean.valueOf(boolean)` never allocates), which is the basis for **instance-controlled** classes (singletons, noninstantiable classes, `a.equals(b)` iff `a==b` guarantees for immutable value types — enums get this for free).
3. **Can return any subtype of the declared return type** — lets an API return objects "without making their classes public," which is how `java.util.Collections` exports 45 implementations via one noninstantiable class with all impl classes package-private.
4. **The returned class can vary from call to call** based on input parameters — e.g. `EnumSet` static factories return `RegularEnumSet` (backed by a `long`) for ≤64 elements or `JumboEnumSet` (backed by a `long[]`) for 65+, invisibly to the client.
5. **The returned class need not exist when the method is written** — the basis of **service provider frameworks** (JDBC: `Connection` = service interface, `DriverManager.registerDriver` = provider registration API, `DriverManager.getConnection` = service access API, `Driver` = service provider interface). Java 6+ ships `java.util.ServiceLoader` so you generally shouldn't roll your own.

Limitations (the disadvantages the panel wants named): (a) classes with only static factories and no public/protected constructor **cannot be subclassed** — arguably a blessing, since it forces composition over inheritance and is required for immutability; (b) static factories are **hard to find** — they don't stand out in Javadoc the way constructors do, so follow naming conventions: `from`, `of`, `valueOf`, `instance`/`getInstance`, `create`/`newInstance`, `getType`/`newType`, `type`.

### Panel-ready answer checklist
- States the core idea: a static method returning an instance, distinct from the GoF Factory Method pattern.
- Names at least 3 of the 5 advantages (naming, non-required allocation/instance-control, subtype flexibility, call-to-call variance, provider-framework flexibility) with a concrete example each.
- Names both disadvantages: no subclassing without a public/protected constructor; discoverability problem in docs.
- Cites at least one naming convention (`of`, `valueOf`, `getInstance`, etc.).
- Connects to service provider frameworks / JDBC or `ServiceLoader` if pushed on "why does this matter at scale."

---

## Item 2: Consider a builder when faced with many constructor parameters

### Opening scenario
"This `NutritionFacts` class has 3 required fields and over 20 optional ones. Someone calls the constructor as `new NutritionFacts(240, 8, 100, 0, 35, 27)`. Walk me through what's wrong here and how you'd redesign the constructor(s)."

### Follow-up probes
- Why not just use a no-arg constructor plus setters (the JavaBeans pattern) — isn't that more readable than counting constructor positions?
- Your builder is a nested static class — why static and not an inner class?
- How would this pattern extend to a class *hierarchy* — say an abstract `Pizza` with `NyPizza` and `Calzone` subtypes each needing their own builder methods?
- Where exactly do you put parameter validation, and why there specifically (constructor vs builder setter)?
- Isn't creating a builder object just extra allocation overhead for no reason?

### Naive attempt
Telescoping constructors: a chain of overloaded constructors, from required-only params up to all params, each calling `this(...)` with a default for the next. Or, when pushed on readability, the candidate pivots to the JavaBeans pattern: no-arg constructor + `setCalories()`, `setSodium()`, etc.

### What breaks
Telescoping constructors: at 6+ params, `new NutritionFacts(240, 8, 100, 0, 35, 27)` is unreadable — the reader must count positions and consult docs to know what `0` and `35` mean; transposed same-typed params (e.g. swapping `sodium` and `carbohydrate`) compile cleanly and silently corrupt data at runtime. JavaBeans pattern is worse structurally: construction is split across multiple calls, so the object exists in an **inconsistent state partway through construction** — nothing stops a consumer from reading a `NutritionFacts` after `setServingSize` but before `setCalories`. It also **precludes immutability** (Item 17) and forces the class to manage thread-safety itself, since mutation can happen at any time from any thread.

### The recommended approach & why
The **Builder pattern**: client calls a constructor/static factory with only the *required* params to get a `Builder`, chains setter-like methods (each returning `this` for a fluent API) for optional params, then calls a no-arg `build()` that returns the (typically immutable) object:
```java
NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8)
    .calories(100).sodium(35).carbohydrate(27).build();
```
Mechanism: the builder accumulates state across chained calls but the target object is only constructed once, atomically, inside `build()` — so the final object is immutable and never observably half-built. Validate individual params in the builder's constructor/setters; validate **cross-field invariants** in the constructor invoked by `build()`, and do those checks on the *copied* fields (post-copy from builder), not the builder's own fields, to defend against race conditions (Item 50). Throw `IllegalArgumentException` naming the bad param on failure.

For **class hierarchies**, use a parallel hierarchy of builders with a recursive generic type bound and simulated self-type:
```java
abstract static class Builder<T extends Builder<T>> {
    EnumSet<Topping> toppings = EnumSet.noneOf(Topping.class);
    public T addTopping(Topping topping) {
        toppings.add(Objects.requireNonNull(topping));
        return self();
    }
    abstract Pizza build();
    protected abstract T self();
}
```
Subclass builders override `build()` with **covariant return typing** (`NyPizza.Builder.build()` returns `NyPizza`, not `Pizza`) so chained calls avoid casts entirely.

Payoff: combines the safety of telescoping constructors (validated, immutable end result) with the readability of JavaBeans (named, chainable calls) — "simulates named optional parameters as found in Python and Scala." Tradeoff to acknowledge: building the builder costs an extra allocation (rarely material) and the pattern is more verbose than a plain constructor, so Bloch's threshold is **~4+ parameters, especially with optional or same-typed ones** — and since retrofitting later makes old constructors "stick out like a sore thumb," it's often better to just start with a builder.

### Panel-ready answer checklist
- Names both rejected alternatives (telescoping constructor, JavaBeans) and their specific failure modes (readability/transposition bugs vs. inconsistent-state/mutability).
- Describes the builder mechanics: required params in builder constructor, optional via chained setters returning `this`, single `build()` call producing an immutable object.
- States where validation belongs (per-field in builder, cross-field invariants in the constructor invoked by `build()`, on copied fields).
- Can describe the hierarchical-builder variant: recursive type bound (`Builder<T extends Builder<T>>`), `self()` method, covariant return types.
- Gives the "~4 or more parameters" heuristic and the builder's own cost (extra object, verbosity).

---

## Item 3: Enforce the singleton property with a private constructor or an enum type

### Opening scenario
"We have a `Elvis` class meant to be instantiated exactly once system-wide — a connection manager, say. Show me how you'd guarantee that, and how you'd make it work if we ever need to serialize it."

### Follow-up probes
- What stops someone from calling the private constructor via reflection?
- If this needs to implement `Serializable`, what specifically goes wrong if you just add `implements Serializable` and nothing else?
- Why would you ever choose the static-factory-method form over the public-final-field form, given the field form is "simpler"?
- How does making it a singleton affect testability for the class's clients?
- Can an enum singleton extend an arbitrary superclass? What does that constrain you to?

### Naive attempt
`public class Elvis { public static Elvis instance = new Elvis(); public Elvis() {...} }` — public (non-final, or with a public constructor) field, or a lazy `getInstance()` with no protection at all.

### What breaks
A public non-final field, or a public constructor left in place, means nothing stops `new Elvis()` from being called again elsewhere in a large codebase — multiple instances now exist, silently breaking every invariant the "singleton" was supposed to provide. Separately: naive `implements Serializable` on any singleton form means **every deserialization manufactures a brand-new instance** — "spurious Elvis sightings" — because deserialization bypasses constructors entirely, breaking the one-instance guarantee.

### The recommended approach & why
Two constructor-based approaches, both keeping the constructor `private`:
```java
// public final field
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
}
// static factory
public class Elvis {
    private static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
    public static Elvis getInstance() { return INSTANCE; }
}
```
Field form: simpler, and the API itself documents the singleton guarantee (`public static final`). Factory form trades that clarity for flexibility: you can change your mind later about being a singleton without changing the API (e.g., return a per-thread instance), you can write a *generic* singleton factory, and the method reference works as a `Supplier<Elvis>` (`Elvis::instance`). Bloch: **prefer the public field approach unless one of the factory's specific advantages is relevant.**

Reflection caveat: a privileged caller can still invoke the private constructor via `AccessibleObject.setAccessible` — if you must defend against that, make the constructor throw on a second invocation.

Serialization fix: mark all instance fields `transient` and supply:
```java
private Object readResolve() {
    return INSTANCE; // let the impersonator be garbage collected
}
```

**Preferred approach overall: a single-element enum**:
```java
public enum Elvis {
    INSTANCE;
    public void leaveTheBuilding() { ... }
}
```
This is "often the best way to implement a singleton" — more concise, gets serialization machinery for free, and is an **ironclad guarantee** against multiple instantiation even under sophisticated reflection or serialization attacks (enum deserialization is handled specially by the JVM so it can't fabricate extra instances the way ordinary classes can). Constraint: unusable if your singleton must extend some class other than `Enum` (it can still implement interfaces).

### Panel-ready answer checklist
- Presents both classic forms (public final field, static factory) and correctly states the field form is preferred absent a specific reason for the factory form.
- Names the reflection attack via `AccessibleObject.setAccessible` and the defense (throw on second construction).
- Explains precisely *why* naive `Serializable` breaks singleton-ness (deserialization fabricates new instances) and the fix (`transient` fields + `readResolve`).
- States the enum-singleton as the generally preferred idiom and why (concise, free serialization, immune to reflection/serialization attacks) plus its one real constraint (can't extend another superclass).
- Notes the testability cost of singletons for their clients (can't substitute a mock unless the singleton exposes an interface type).

---

## Item 4: Enforce noninstantiability with a private constructor

### Opening scenario
"You've got a `MathUtils` class — nothing but static methods like `java.lang.Math`. A teammate writes `MathUtils utils = new MathUtils();` somewhere and it compiles fine. What's actually going on, and how do you stop it?"

### Follow-up probes
- Why not just make the class `abstract` to prevent instantiation?
- If you make the constructor `private`, don't you also need to guard against the class calling its own constructor internally by mistake?
- What's a side effect of the noninstantiability idiom on subclassing?
- With Java 8+ allowing static interface methods, when would you still want a separate utility class at all?

### Naive attempt
Candidate declares the class `abstract` with no explicit constructor, assuming that blocks instantiation.

### What breaks
`abstract` only prevents direct instantiation of *that* class — nothing stops someone from writing `class Sub extends MathUtils {}` and instantiating the subclass, `new Sub()`. It also actively misleads future maintainers into thinking the class was designed for inheritance/extension (Item 19), when a stateless utility grouping was never meant to be extended at all.

### The recommended approach & why
The compiler synthesizes a public, parameterless default constructor **only when a class declares no explicit constructor**. So the fix is a single explicit **private** constructor:
```java
public class UtilityClass {
    // Suppress default constructor for noninstantiability
    private UtilityClass() {
        throw new AssertionError();
    }
    ...
}
```
Mechanism: the private constructor is inaccessible from outside the class, so external code cannot instantiate it; the `AssertionError` is insurance against accidental *internal* invocation (e.g., another constructor in the same class calling this one). Side effect (worth calling out proactively): this also prevents subclassing entirely, since every constructor must invoke a superclass constructor and there's no accessible one to invoke — which is exactly right for a class that's just a namespace for static members (`java.lang.Math`, `java.util.Arrays`, or a pre-Java-8-style companion class for an interface's static factory methods, à la the historical `java.util.Collections`).

### Panel-ready answer checklist
- Explains *why* the bug exists at all: the compiler's implicit no-arg public constructor is what makes the class instantiable by default.
- Correctly rejects `abstract` as insufficient and explains the subclass-instantiation loophole plus the misleading "designed for inheritance" signal.
- Gives the private-constructor idiom including the `AssertionError` guard against accidental internal calls.
- States the side effect: noninstantiability also forecloses subclassing, and frames that as desirable here.
- Can mention that Java 8+ static interface methods reduce (but don't eliminate) the need for this pattern.

---

## Item 5: Prefer dependency injection to hardwiring resources

### Opening scenario
"A `SpellChecker` class hardcodes `private static final Lexicon dictionary = ...;` as a static field, exposed via static methods. Now product wants per-language dictionaries and QA wants to swap in a test dictionary. What's your redesign?"

### Follow-up probes
- Could you just add a `setDictionary(Lexicon)` method to the existing static/singleton class instead of a redesign?
- How does this generalize when a class needs many dependencies, not just one?
- What's the difference between passing a resource directly versus passing a `Supplier<T>`/factory for it?
- Doesn't manually wiring dozens of dependencies through constructors get unwieldy at scale — what's the answer to that?
- Does dependency injection conflict with keeping a class immutable?

### Naive attempt
Static utility class (`private static final Lexicon dictionary`, private constructor, static `isValid`/`suggestions` methods) or a singleton (`private final Lexicon dictionary` set once in a private constructor, exposed via `public static INSTANCE`).

### What breaks
Both bake in the **assumption that exactly one dictionary will ever be needed** — but each language needs its own dictionary, special vocabularies need special dictionaries, and tests need a fake one. Neither design supports multiple simultaneous instances behaving differently. Making the field non-final and adding a mutator method to "fix" this is worse: it's error-prone (any code path might mutate shared state mid-use) and "unworkable in a concurrent setting" — one thread swaps the dictionary while another is mid-lookup.

### The recommended approach & why
**Dependency injection**: pass the resource (the dependency) into the constructor when creating each instance, rather than the class reaching out and grabbing/constructing it itself:
```java
public class SpellChecker {
    private final Lexicon dictionary;
    public SpellChecker(Lexicon dictionary) {
        this.dictionary = Objects.requireNonNull(dictionary);
    }
    public boolean isValid(String word) { ... }
    public List<String> suggestions(String typo) { ... }
}
```
Mechanism: each `SpellChecker` instance now owns its own reference to whichever `Lexicon` it was given at construction — multiple instances can coexist with different dictionaries, including a test double, with zero shared mutable state. This preserves immutability (Item 17), so multiple clients can safely share the same dependent objects, and it generalizes to an arbitrary number of resources and arbitrary dependency graphs. Applies equally to constructors, static factories, and builders.

**Variant**: pass a resource *factory* (a `Supplier<T>`, bounded with a wildcard per Item 31) instead of the resource itself, when the class needs to create new instances of the resource repeatedly rather than share one:
```java
Mosaic create(Supplier<? extends Tile> tileFactory) { ... }
```
This embodies the GoF Factory Method pattern.

At scale, manual DI "can clutter up large projects" with thousands of dependencies — that's what **DI frameworks** (Dagger, Guice, Spring) exist to eliminate; APIs designed for manual DI adapt trivially to these frameworks, so designing for manual injection first is not wasted work.

### Panel-ready answer checklist
- Names both rejected patterns (static utility, singleton) and states precisely *why* both fail: the single-dictionary assumption plus the concurrency/mutability trap of a "just add a setter" patch.
- Presents the fix as passing the dependency (or a `Supplier`/factory for it) into the constructor — names it "dependency injection."
- Notes this preserves immutability and thread-safety and scales to arbitrary dependency graphs.
- Mentions the `Supplier<T>` factory variant and when to prefer it over injecting the resource directly.
- Brings up DI frameworks (Dagger/Guice/Spring) as the answer to manual-wiring clutter at scale, without overstating it as required for small projects.

---

## Item 6: Avoid creating unnecessary objects

### Opening scenario
"A hot-path method does `String s = new String("bikini");` inside a loop that runs millions of times. Separately, another method validates Roman numerals with `s.matches(regex)` and profiling shows it's a bottleneck. Diagnose both."

### Follow-up probes
- If reusing objects is good, why not maintain your own object pool for these frequently-created objects?
- What's actually expensive about `String.matches` — walk me through what happens under the hood on each call?
- Where's the counterargument — when does creating *more* objects actually make code better, not worse?
- Autoboxing: what specifically goes wrong if `sum` in a summing loop is declared `Long` instead of `long`?
- Is it ever fine to have multiple instances of the same conceptual "view" object, like `Map.keySet()`? Why or why not create a fresh one each call?

### Naive attempt
`String s = new String("bikini");` treated as harmless; and for the regex check:
```java
static boolean isRomanNumeral(String s) {
    return s.matches("^(?=.)M*(C[MD]|D?C{0,3})(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
}
```
plus a summing loop using `Long sum = 0L;` accumulated with `+=`.

### What breaks
`new String("bikini")` allocates a brand-new `String` instance functionally identical to the literal argument every single call — "millions of String instances can be created needlessly" in a loop or hot method, all pure garbage-collector churn with zero benefit (string literals are already interned/reused per the JLS). `String.matches` silently **compiles a new `Pattern` (regex → finite state machine) on every single call**, uses it once, and discards it — measured by Bloch at 1.1 µs vs. 0.17 µs for a cached `Pattern`, a 6.5x hit, worse the hotter the path. The autoboxing bug — `Long sum` instead of `long sum` in a loop adding `int`s up to `Integer.MAX_VALUE` — silently boxes ~2^31 unnecessary `Long` objects, one per iteration, turning a 0.59s loop into 6.3s: "a one-character typographical error."

### The recommended approach & why
Reuse instead of reallocate wherever objects are functionally identical:
- `String s = "bikini";` — one instance, and the JLS guarantees literal reuse across the same JVM.
- Prefer static factories over constructors on immutable classes offering both (`Boolean.valueOf(String)` over the deprecated `Boolean(String)` constructor) since factories aren't *required* to allocate.
- Cache the expensive object once, as a `static final` field, and reuse it:
```java
public class RomanNumerals {
    private static final Pattern ROMAN = Pattern.compile(
        "^(?=.)M*(C[MD]|D?C{0,3})(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
    static boolean isRomanNumeral(String s) {
        return ROMAN.matcher(s).matches();
    }
}
```
This also *improves clarity* — a named constant beats an inline literal regex. (Lazy-initializing that field to defer the cost is explicitly **not recommended** — it complicates the code for no measurable win, per Item 83/67.)
- Recognize non-obvious reuse cases: **adapters/views** (e.g. `Map.keySet()`) have no state beyond their backing object, so returning the same instance repeatedly is safe and there's no benefit to allocating fresh ones each call.
- Fix autoboxing bugs by preferring primitives to boxed types in hot code and watching for unintentional boxing.

Counterpoint (must be stated to avoid over-applying this item): **don't avoid object creation reflexively.** Small objects with cheap constructors are cheap to create and reclaim on modern JVMs; creating objects to improve clarity, simplicity, or correctness is "generally a good thing." Maintaining your own object pool is a bad idea *unless objects are extremely heavyweight* (Bloch's canonical counter-example: database connections) — modern GC easily outperforms hand-rolled pools for lightweight objects, and pools clutter code and increase memory footprint. There's also a named counterpoint item: Item 50 (defensive copying) says the opposite thing in the opposite direction — reusing an object when you should have defensively copied is a *worse* bug (security holes, insidious corruption) than needlessly allocating.

### Panel-ready answer checklist
- Identifies both concrete failure modes precisely: needless `String` reallocation, and `Pattern` recompilation hidden inside `String.matches`, with the actual mechanism (regex → FSM compile cost) not just "it's slow."
- Gives the fix as caching in a `static final` field and explains why lazy-init isn't worth it here.
- States the counterpoint explicitly: don't over-apply this — small cheap objects are fine, custom object pools are usually harmful except for genuinely heavyweight resources (DB connections).
- Explains the autoboxing trap with the primitive-vs-boxed distinction and a concrete cost (measured slowdown).
- Can name the Item 50 tension (reuse-when-you-should-copy is worse than allocate-when-you-need-not) if pushed on "when does this rule reverse."

---

## Item 7: Eliminate obsolete object references

### Opening scenario
"Here's a hand-rolled `Stack` backed by an `Object[]`, with `push`/`pop`/`ensureCapacity`. It passes every unit test. In production, after heavy push/pop churn, the app eventually dies with `OutOfMemoryError`. Find the bug."

### Follow-up probes
- Since Java is garbage collected, why doesn't the GC just reclaim the popped elements automatically?
- Should you null out every reference the moment you're done with it, as a general policy?
- What's the analogous leak risk in a cache, and how do you fix that one differently from the Stack case?
- What about listener/callback registries — same fix?
- What's the actual mechanism by which nulling a reference helps you *catch* bugs, not just avoid the leak?

### Naive attempt
```java
public Object pop() {
    if (size == 0) throw new EmptyStackException();
    return elements[--size];
}
```
Candidate sees no bug: it compiles, passes tests, decrements size correctly.

### What breaks
`elements[--size]` is returned but the array slot at the old index still **holds a reference** to the popped object even though it's logically outside the "active portion" (`index < size`). The GC has no concept of "active portion" — to it, every slot in `elements` is an equally valid live reference, so those objects (and everything *they* transitively reference) can never be collected. This is an **unintentional object retention** ("memory leak" in a GC'd language) — it manifests as increased GC activity/memory footprint, and in extreme cases disk paging or `OutOfMemoryError`, potentially years after the class was written, discoverable typically only via heap profiler or careful inspection.

### The recommended approach & why
Null out the reference the instant it becomes obsolete:
```java
public Object pop() {
    if (size == 0) throw new EmptyStackException();
    Object result = elements[--size];
    elements[size] = null; // Eliminate obsolete reference
    return result;
}
```
Mechanism: an **obsolete reference** is one that will never be dereferenced again; nulling it removes the GC root path to that object (and transitively, anything only it references). Bonus payoff: if the nulled slot is mistakenly dereferenced later, you get an immediate `NullPointerException` instead of silent wrong behavior — fail fast beats fail quiet.

Critical scoping rule (don't over-apply): **this is the exception, not the norm.** Nulling every reference as soon as you're "done" with it clutters code for no benefit. The default, correct pattern is to let variables **fall out of scope naturally** by declaring them in the narrowest possible scope (Item 57) — that alone eliminates most obsolete references without explicit nulling.

**When you specifically need to null things out**: whenever a class **manages its own memory** — here, the `Stack`'s backing array is a private storage pool the GC can't reason about, since only the class itself knows which slots are "allocated" (active) vs. "free" (inactive-but-still-referenced). Any class doing manual memory management like this should be audited for this exact leak pattern.

Two other named leak sources beyond self-managed arrays:
- **Caches** — an entry can be forgotten and outlive its relevance. If an entry's usefulness is tied strictly to external references to its *key*, use `WeakHashMap` (entries auto-removed once the key becomes otherwise unreachable — only useful when lifetime is key-driven, not value-driven). Otherwise, periodically purge stale entries via a background thread (`ScheduledThreadPoolExecutor`) or as a side effect of insertion (`LinkedHashMap.removeEldestEntry`); for advanced cases, use `java.lang.ref` directly.
- **Listeners/callbacks** — clients that register but never deregister accumulate forever unless you store only **weak references** to them (e.g., as keys in a `WeakHashMap`).

### Panel-ready answer checklist
- Correctly identifies the leaked reference (the stale array slot beyond `size`) and explains *why the GC can't help*: it has no notion of "active portion," only the programmer does.
- Gives the fix (null the slot on pop) and states the bonus benefit (fail-fast `NullPointerException` on misuse).
- States the scoping caveat explicitly: this is the exception, not a general policy — narrow variable scope is the primary defense.
- Names the general trigger condition: classes that manage their own memory need this scrutiny.
- Covers at least one of the two other leak sources (caches → `WeakHashMap` or eviction; listeners → weak references) with the correct applicability condition for `WeakHashMap` specifically (key-driven lifetime, not value-driven).

---

## Item 8: Avoid finalizers and cleaners

### Opening scenario
"Someone added a `finalize()` method to a class that wraps a native file handle, reasoning 'it'll close the file when the object's garbage collected, so we can't leak file descriptors.' Six months later, production starts failing with 'too many open files' under load. Explain what's going on and what you'd do differently."

### Follow-up probes
- Isn't a finalizer basically Java's version of a C++ destructor — why wouldn't that reasoning hold?
- Could you just call `System.gc()` before checking resource state, to force finalizers to run?
- What happens if the resource-holding constructor throws partway through — is there any residual risk from having a finalizer at all, even a "harmless" one?
- If cleaners are "less dangerous," why not just always use them instead of the AutoCloseable/close() pattern?
- What are the *only* legitimate uses of a finalizer or cleaner, if any?

### Naive attempt
Add a `finalize()` (or, in modern code, a `Cleaner`-based safety net) as the *primary* mechanism for releasing a scarce resource like a file descriptor or a lock, with no explicit `close()` required from callers.

### What breaks
No promptness guarantee: "It can take arbitrarily long between the time that an object becomes unreachable and the time its finalizer or cleaner runs" (JLS 12.6) — for a limited resource like file descriptors, this means a program can exhaust available descriptors and fail to open new files purely because GC hasn't gotten around to running finalizers yet, even though logically nothing is using those files anymore. Worse: **no guarantee they run at all** — a program can terminate having never finalized some unreachable objects, so relying on a finalizer to release a persistent lock on a shared resource (e.g., a DB lock) "is a good way to bring your entire distributed system to a grinding halt." Additional dangers: an uncaught exception during finalization is silently swallowed (no stack trace, no warning) and finalization of that object simply stops, potentially leaving other objects corrupted; finalizers impose a ~50x construct/destroy performance penalty (12ns via try-with-resources vs. 550ns with a finalizer) because they inhibit efficient GC; and nonfinal classes with finalizers are exposed to **finalizer attacks** — a malicious subclass's finalizer can run on a partially-constructed object (one whose constructor threw, which should never have come into existence) and stash a reference to it in a static field, permanently defeating garbage collection and allowing arbitrary method calls on an object that should be dead.

### The recommended approach & why
Implement `AutoCloseable`, require clients to call `close()`, and use `try`-with-resources (Item 9) so `close()` runs even under exceptions. The instance should track its own closed-state in a field and throw `IllegalStateException` from other methods if called post-close.

Finalizer attack defense: final classes are automatically immune (no subclass possible); for nonfinal classes, write a `final finalize()` method that does nothing, blocking any subclass override.

**The only two legitimate uses**, per Bloch:
1. **Safety net** — in case the resource owner forgets to call `close()`. Late cleanup beats none; some JDK classes (`FileInputStream`, `FileOutputStream`, `ThreadPoolExecutor`, `java.sql.Connection`) do exactly this. Weigh the (real, ~5x when used purely as a safety net — about 66ns vs 12ns) cost against the protection before adding one.
2. **Native peers** — a native (non-Java) object a Java object delegates to via native methods; the GC can't see or reclaim it since it isn't a normal Java object. Only appropriate if performance is acceptable and the native peer holds no resources that must be reclaimed promptly (otherwise, require `close()` instead).

Cleaner mechanics example (`Room`/`State`/`Cleaner.Cleanable`):
```java
public class Room implements AutoCloseable {
    private static final Cleaner cleaner = Cleaner.create();
    private static class State implements Runnable {
        int numJunkPiles;
        State(int numJunkPiles) { this.numJunkPiles = numJunkPiles; }
        @Override public void run() {
            System.out.println("Cleaning room");
            numJunkPiles = 0;
        }
    }
    private final State state;
    private final Cleaner.Cleanable cleanable;
    public Room(int numJunkPiles) {
        state = new State(numJunkPiles);
        cleanable = cleaner.register(this, state);
    }
    @Override public void close() { cleanable.clean(); }
}
```
Critical detail: `State` **must not hold a reference back to `Room`** — that would create a cycle keeping `Room` reachable forever, defeating the entire mechanism. That's why `State` must be a `static` nested class (non-static nested classes implicitly hold an enclosing-instance reference, Item 24) — and why lambdas are similarly dangerous here (they can silently capture enclosing references). Demonstrated behavior: a well-behaved caller using try-with-resources reliably prints "Cleaning room"; an ill-behaved caller that just does `new Room(99)` and exits may **never** print it — cleaner-during-`System.exit` behavior is explicitly implementation-specific with no guarantees, illustrating the unpredictability directly.

### Panel-ready answer checklist
- Rejects the C++-destructor mental model explicitly: GC reclaims memory automatically; finalizers/cleaners are not the Java analog for releasing non-memory resources.
- States both core guarantees that don't exist: no promptness guarantee, no guarantee-of-execution-at-all — with the file-descriptor-exhaustion or DB-lock consequence as concrete stakes.
- Names the finalizer attack mechanism (constructor throws → subclass finalizer runs anyway → reference smuggled into a static field) and the defense (final class, or a final no-op `finalize()`).
- Gives the correct replacement: `AutoCloseable` + `close()` + try-with-resources, with a closed-state field guarding post-close calls.
- Names both legitimate exceptions (safety net; native peers) and the condition attached to each.
- If discussing the cleaner code specifically: explains why the cleanable state object must not reference the enclosing object (reachability cycle) and why it must be a static nested class.

---

## Item 9: Prefer try-with-resources to try-finally

### Opening scenario
"Code review: a method opens two resources — an `InputStream` and an `OutputStream` — copies bytes between them, and closes both in nested `try/finally` blocks. It looks correct and passes all tests. What's still wrong with it, and what happens the day the disk starts failing intermittently?"

### Follow-up probes
- The nested try-finally *does* close both resources correctly in the happy path and in most failure paths — what specifically is the remaining defect?
- What actually happens to the exception from the `try` block if `close()` in the `finally` block *also* throws?
- Can you still catch exceptions cleanly when using try-with-resources, or do you lose that?
- Why does this matter enough that Bloch calls out that "two-thirds of the uses of the close method in the Java libraries were wrong in 2007" — what does "wrong" mean there if the code still "works"?
- What must a resource class implement to be usable here, and whose responsibility is that?

### Naive attempt
```java
static void copy(String src, String dst) throws IOException {
    InputStream in = new FileInputStream(src);
    try {
        OutputStream out = new FileOutputStream(dst);
        try {
            byte[] buf = new byte[BUFFER_SIZE];
            int n;
            while ((n = in.read(buf)) >= 0)
                out.write(buf, 0, n);
        } finally {
            out.close();
        }
    } finally {
        in.close();
    }
}
```

### What breaks
Both the `try` block (`in.read`/`out.write`) and the `finally` block (`close()`) are capable of throwing. If the physical device fails, `in.read` throws — and then `in.close()` in `finally` *also* throws, for the same underlying reason. The second exception **completely obliterates the first one** in the stack trace: there is no record at all of the original, diagnostically-useful failure. This isn't a hypothetical edge case — it's exactly the scenario where you most need good diagnostics (a failing device), and it's exactly when you get none. Bloch notes even experienced authors get this pattern wrong in practice ("I got it wrong on page 88 of Java Puzzlers... no one noticed for years"; "two-thirds of the uses of the close method in the Java libraries were wrong in 2007") — meaning "wrong" here isn't a crash, it's silently swallowed root causes making production incidents nearly undiagnosable.

### The recommended approach & why
Use `try`-with-resources (Java 7+, JLS 14.20.3). The resource type must implement `AutoCloseable` (a single `void close()` method) — if you write a class representing a closeable resource, implement `AutoCloseable` yourself.
```java
static void copy(String src, String dst) throws IOException {
    try (InputStream in = new FileInputStream(src);
         OutputStream out = new FileOutputStream(dst)) {
        byte[] buf = new byte[BUFFER_SIZE];
        int n;
        while ((n = in.read(buf)) >= 0)
            out.write(buf, 0, n);
    }
}
```
Mechanism: the compiler-generated close logic, when both the body and the implicit `close()` throw, **automatically suppresses** the `close()` exception in favor of the one from the body — but it does **not discard** it. Suppressed exceptions are attached to the primary exception, printed in the stack trace with a "Suppressed:" annotation, and retrievable programmatically via `Throwable.getSuppressed()` (added Java 7). You keep the diagnostically important exception as primary *and* keep the secondary one, instead of silently losing one entirely.

You can still attach `catch` clauses directly to a try-with-resources statement, avoiding an extra nesting layer for error handling:
```java
static String firstLineOfFile(String path, String defaultVal) {
    try (BufferedReader br = new BufferedReader(new FileReader(path))) {
        return br.readLine();
    } catch (IOException e) {
        return defaultVal;
    }
}
```
Payoff, stated plainly: shorter, more readable code, and *strictly better* diagnostics than even a correctly-written try-finally can produce — because try-finally has no suppression mechanism at all, correctness there is bounded by "doesn't lose the first exception's real cause," which try-with-resources gets automatically and try-finally cannot.

### Panel-ready answer checklist
- Identifies the exact defect: when both body and `close()` throw, `finally`-based cleanup lets the `close()` exception overwrite/hide the original one in the stack trace, with zero record of the first.
- States the fix: `try`-with-resources, and that resources used this way must implement `AutoCloseable`.
- Explains the suppression mechanism specifically — the primary exception is preserved, secondary ones are attached as "Suppressed" and accessible via `getSuppressed()` — not just "it closes things for you."
- Confirms `catch` clauses still work directly on a try-with-resources statement, without extra nesting.
- Frames the conclusion correctly: prefer try-with-resources for *all* closeable-resource code, not merely as a style preference — it is strictly safer for diagnostics even when the try-finally was written correctly.
