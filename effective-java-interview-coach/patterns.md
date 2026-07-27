# Effective Java — Design Patterns & Techniques Quick Reference

## Static Factory Method
**When**: Need named constructors, instance-control/caching, or to return a subtype/interface without exposing the impl class (Item 1).
**How**: Static method (`of`, `from`, `valueOf`, `getInstance`) returns an instance instead of `new`.
**Trade-offs**: No-public-constructor classes can't be subclassed (often a feature); factories don't stand out in Javadoc like constructors.

## Builder Pattern
**When**: 4+ constructor params, especially optional/same-typed ones — telescoping ctors unreadable, JavaBeans setters allow inconsistent state (Item 2).
**How**: Static nested `Builder` with chained setters returning `this`, terminal `build()`. Hierarchies: generic self-type `Builder<T extends Builder<T>>` + abstract `self()` so subclass builders chain without casts; validate after copying fields off the builder.
**Trade-offs**: Verbose for few params; extra allocation to build the builder itself.

## Singleton: enum vs private constructor
**When**: Exactly-one-instance components (Item 3).
**How**: Single-element enum (`enum Elvis { INSTANCE; }`) — preferred, free serialization + reflection-attack immunity. Private ctor + public static final field is simpler but needs `readResolve` for serialization and is breakable via `setAccessible`.
**Trade-offs**: Enum singleton can't extend a class; singletons are hard to mock unless typed to an interface.

## Noninstantiable Utility Class
**When**: Grouping static methods/constants, no object identity (Item 4).
**How**: Single `private` ctor throwing `AssertionError` if invoked reflectively.
**Trade-offs**: Also blocks subclassing (no accessible super ctor) — usually desired.

## Dependency Injection
**When**: Behavior parameterized by a resource — not a static-utility/singleton problem (Item 5).
**How**: Pass the resource, or a `Supplier<T>` factory for it, into the constructor/static factory/builder.
**Trade-offs**: Manual DI clutters large graphs — reach for Dagger/Guice/Spring at scale; preserves immutability + testability.

## Try-With-Resources
**When**: Anything `AutoCloseable` (Item 9).
**How**: `try (Resource r = ...) { }` — closes in reverse order even on exception.
**Trade-offs**: Strictly better than try-finally: preserves the first exception, suppresses the close-time one instead of masking it.

## Defensive Copying
**When**: Ctors/accessors touching mutable objects a client could later mutate (Item 50); `readObject` for classes with mutable private fields (Item 88).
**How**: Copy on the way in (before validity checks — closes a TOCTOU window) and on the way out; don't use `clone()` if the class is subclassable by untrusted code.
**Trade-offs**: Costs a copy; skip only with mutual trust across a package boundary, documented.

## Copy Constructor / Copy Factory (vs Cloneable)
**When**: Need to duplicate an object; `Cloneable`/`clone()` is extralinguistic, fragile with `final` fields (Item 13).
**How**: `public Yum(Yum yum)` or `static Yum newInstance(Yum yum)`; can take an interface type for a "conversion" copy (`new TreeSet<>(hashSet)`).
**Trade-offs**: Arrays remain the one case where `clone()` is still right.

## Composition + Forwarding (Wrapper/Decorator)
**When**: Extend a concrete class's behavior across package boundaries without a true "is-a" relationship (Item 18).
**How**: New class holds a private reference to the wrapped instance, implements the same interface, forwards each method, adds behavior around the calls.
**Trade-offs**: Breaks in callback frameworks (SELF problem — wrapped object hands out `this`, bypassing the wrapper).

## Skeletal Implementation (Template Method)
**When**: Publishing a nontrivial interface most implementors would repeat logic for (Item 20).
**How**: `AbstractInterface` implementing non-primitive methods atop a few abstract primitives (`AbstractList`, etc.).
**Trade-offs**: Can't override `Object` methods via interface defaults — those still need the abstract class.

## Class Hierarchy over Tagged Class
**When**: A class switches behavior on a tag field (Item 23).
**How**: Abstract root with one abstract method per tag-dependent behavior; one concrete subclass per flavor.
**Trade-offs**: More types, but kills irrelevant fields, missing-case bugs, enables natural subtyping.

## Static Member Class over Nonstatic
**When**: Nested class doesn't need the enclosing instance (Item 24).
**How**: Add `static` to the nested class.
**Trade-offs**: Omitting it silently retains a hidden outer-instance ref — a common memory leak (Item 7).

## PECS / Bounded Wildcards
**When**: API params that are producer or consumer collections (Item 31).
**How**: **P**roducer-**E**xtends, **C**onsumer-**S**uper: `List<? extends E>` to read from, `Collection<? super E>` to write to; never wildcard a return type; `Comparable<? super T>` since comparators are consumers.
**Trade-offs**: If a param is both read and written, no wildcard works — need exact type `E`.

## Typesafe Heterogeneous Container
**When**: Unbounded number of typed values per instance where normal generics cap you at one type param (Item 33).
**How**: Parameterize the key, not the container — `Map<Class<?>, Object>` internally, `<T> T getFavorite(Class<T> type)` using `Class.cast` to restore type safety.
**Trade-offs**: Raw-typed `Class` can corrupt it (unchecked warning); doesn't work for non-reifiable types (`List<String>.class` is illegal).

## Enum Strategy / Constant-Specific Method Bodies
**When**: Each constant needs different behavior (Item 34).
**How**: Declare method `abstract` in the enum, override per-constant instead of switching on `this`. If only *some* constants share logic, delegate to a private nested "strategy" enum passed to each constant's ctor.
**Trade-offs**: Switch-on-enum is fine for augmenting an enum you don't own; constant bodies make cross-constant sharing awkward — that's when the strategy-enum variant earns its keep.

## EnumSet / EnumMap over Bit Fields / Ordinal Indexing
**When**: Sets/maps keyed by enum constants (Item 36, 37).
**How**: `EnumSet.of(...)` (bitmask-fast, typesafe); `new EnumMap<>(Key.class)` instead of `array[e.ordinal()]`; nest `EnumMap<..., EnumMap<...>>` for multi-dim relations.
**Trade-offs**: Never index on `Enum.ordinal()` — reordering/adding constants silently breaks it.

## Standard Functional Interfaces + Lambdas
**When**: Small function/strategy objects (Item 42, 44).
**How**: Lambdas over anonymous classes; `java.util.function`'s standard interfaces (`Function`, `Predicate`, `Supplier`, `Consumer`, `UnaryOperator`, `BinaryOperator` + primitive/Bi variants) over custom ones.
**Trade-offs**: Roll your own only for a documented contract/custom defaults/self-documenting name (like `Comparator`). Lambdas can't self-reference or serialize reliably; never overload two functional-interface params in the same slot — ambiguous call sites.

## Optional Return
**When**: A method may conceptually have no result (Item 55).
**How**: `Optional.empty()`/`Optional.of(v)`; return empty collections/arrays instead of wrapping them in Optional (Item 54); never return `null` from an Optional method.
**Trade-offs**: Skip for perf-critical paths (allocation cost) and never use as a field type/collection element.

## Executor Framework over Raw Threads
**When**: Any background/async work (Item 80).
**How**: `Executors.newFixedThreadPool` (avoid `newCachedThreadPool` under heavy load — unbounded thread growth) or `ThreadPoolExecutor` directly; submit `Runnable`/`Callable`, always `shutdown()`.
**Trade-offs**: Fork-join/parallel streams sit atop this but are tricky to tune correctly.

## Concurrency Utilities over wait/notify
**When**: Any new concurrent coordination (Item 78, 79, 81).
**How**: `ConcurrentHashMap`/`CopyOnWriteArrayList` over `Collections.synchronizedX`; `CountDownLatch`/`Semaphore` over raw `wait`/`notify`; `AtomicLong` for lock-free counters; `volatile` only for visibility, not atomicity.
**Trade-offs**: Never call an "alien" (overridable/client-supplied) method from inside a synchronized block — take a snapshot or use an open call to avoid deadlock. Document thread-safety level explicitly; `synchronized` in source isn't part of the API.

## Serialization Proxy Pattern
**When**: Class implements `Serializable` and deserialization shouldn't be an unguarded "extra constructor" (Item 90).
**How**: Private static nested `SerializationProxy` mirrors logical state; `writeReplace()` emits the proxy; proxy's `readResolve()` rebuilds via the real public constructor/factory (re-validates invariants); enclosing `readObject()` throws to reject non-proxy streams.
**Trade-offs**: ~14% slower than defensive-copy `readObject`; incompatible with client-extendable classes and circular object graphs — but the most robust option, and enables truly `final` fields.

## Instance Control via Enum (not readResolve)
**When**: Enforcing a fixed/singleton instance set across serialization (Item 89).
**How**: Represent as an enum. If it must be a non-enum class, provide `readResolve` and declare every reference field `transient` (else an attacker can steal a reference before `readResolve` runs).
**Trade-offs**: `readResolve` alone is fragile against a crafted stream; enum sidesteps the attack for free.

## Custom Serialized Form
**When**: Physical representation differs from logical data (e.g. linked list, hash table) (Item 87).
**How**: Mark fields `transient`; `writeObject`/`readObject` call `defaultWriteObject`/`defaultReadObject` first, then explicitly write/read only logical data.
**Trade-offs**: Default form permanently exposes private fields as API, can be slow/huge, and can stack-overflow on deep graphs. Always declare an explicit `serialVersionUID` regardless of form chosen.
