# Effective Java — Glossary

**Abstract skeletal implementation** — abstract class implementing an interface's non-primitive methods atop a few primitives, so subclasses fill in only primitives; Template Method pattern (Item 20).
**AutoCloseable / try-with-resources** — `close()`-based resource interface used with try-with-resources for guaranteed, exception-safe termination instead of finalizers (Item 9).
**Autoboxing** — automatic primitive↔boxed conversion; can silently create unnecessary objects and hurt performance (Item 6, Item 61).
**Bit field** — OR'd int constants for set membership; superseded by `EnumSet`, which packs the set into a bit vector (Item 36).
**Bounded wildcard** — `? extends T` / `? super T`, used per PECS for flexible, type-safe APIs (Item 31).
**Builder pattern** — chained-setter object that accumulates constructor parameters, then `build()`s the immutable object; replaces telescoping constructors (Item 2).
**Canonical form** — a normalized field representation cached so `equals` can do a cheap exact comparison (Item 10).
**Checked exception** — used for conditions the caller can reasonably recover from; forces the caller to handle or propagate it (Item 70).
**Cleaner / finalizer** — slow, unpredictable termination mechanisms to avoid; implement `AutoCloseable` instead (Item 8).
**Comparable / natural ordering** — single-method interface whose `compareTo` defines a type's default ordering (Item 14).
**compareTo contract** — must be antisymmetric and transitive, and equal-comparing objects must compare equally to all others; ideally consistent with `equals` (Item 14).
**Composition over inheritance** — hold and forward to another instance instead of extending it, avoiding fragility from a superclass's self-use (Item 18).
**Constant interface antipattern** — an interface holding only constants for classes to `implement`; misuses interfaces, which should define types (Item 22).
**Covariant return type** — an override may return a subtype of the overridden method's return type, e.g. `clone()` returning the concrete class (Item 13).
**Custom serialized form** — hand-designed `writeObject`/`readObject` layout representing an object's logical data, not its physical fields (Item 87).
**Decorator pattern** — a wrapper class that adds behavior to another instance while delegating the rest (Item 18).
**Defensive copy** — copying mutable inputs/outputs (constructors, accessors, `readObject`) so external code can't violate invariants post-validation (Item 50, Item 88).
**Dependency injection** — pass a resource (or its factory) into a constructor instead of hardwiring it, improving flexibility and testability (Item 5).
**Effectively immutable / safe publication** — an object mutated briefly then never again can be shared without further synchronization if its reference is safely published (`static`/`volatile`/`final` field) (Item 78).
**Encapsulation / information hiding** — hiding a component's internals behind its API, decoupling components (Item 15).
**Enum type** — type-safe replacement for int/String constant patterns; can carry per-instance data and behavior (Item 34).
**equals contract** — must be reflexive, symmetric, transitive, consistent, and `false` against `null`; violations break equals-dependent collections (Item 10).
**Extensible enum** — emulated via an interface implemented by multiple enum types, since enums can't be subclassed (Item 38).
**hashCode caching** — lazily computing and storing an immutable object's hash on first `hashCode()` call (Item 11).
**Failure atomicity** — a failed call should leave the object in its pre-call state; immutable objects get this for free (Item 76, Item 17).
**Flyweight pattern** — reusing preconstructed/cached instances instead of creating new ones, as in `Boolean.valueOf` (Item 1).
**Generic method** — a method whose type parameter is declared on the method itself (Item 30).
**Generic singleton factory** — one immutable object cast and dispensed as any requested type parameterization, relying on erasure (Item 30).
**Liveness / safety failure** — unsynchronized shared mutable state causes either no progress (liveness) or wrong results (safety) (Item 78).
**Heap pollution** — a parameterized-type variable refers to an object not actually of that type, often via a varargs array (Item 32).
**Holder class idiom** — lazy static-field init exploiting the JVM's guarantee a class isn't initialized until used, avoiding synchronization (Item 83).
**Immutable class** — no mutators, all fields final and private, no subclassing, exclusive access to mutable components (Item 17).
**Instance-controlled class** — guarantees no instances exist beyond those it creates; basis for singletons and Flyweight (Item 1).
**Marker interface** — a no-method interface designating a property (e.g. `Serializable`); preferred over marker annotations when used as a parameter type (Item 41).
**Memory leak / obsolete reference** — a held reference that will never be dereferenced again, blocking GC; common in self-managed arrays, caches, listeners (Item 7).
**Method reference** — compact lambda alternative referring directly to an existing method (Item 43).
**Monitor lock** — the intrinsic lock a `synchronized` method/block acquires, guaranteeing mutual exclusion and visibility (Item 78).
**Noninstantiability** — a utility class throws `AssertionError` from a private no-arg constructor to block instantiation (Item 4).
**Ordinal** — an enum constant's declaration position; fragile to use as an index/ID since reordering breaks it (Item 35).
**PECS** — Producer-Extends, Consumer-Super: a `T`-producing parameter uses `? extends T`; a `T`-consuming parameter uses `? super T` (Item 31).
**Raw type** — a generic type used without its type parameter (e.g. `List`); kept only for pre-generics compatibility, unsafe in new code (Item 26).
**Recursive type bound** — a type parameter bounded by an expression involving itself, e.g. `<T extends Comparable<T>>` (Item 30).
**Reified vs erased** — arrays are reified (enforce element type at runtime, covariant); generics are erased at runtime and invariant (Item 28).
**Self-type idiom** — a builder's recursive type parameter plus abstract `self()`, letting method chaining work across subclasses (Item 2).
**Serialization proxy pattern** — a private static nested class representing an object's logical state via `writeReplace`/`readResolve`, so the real class is never deserialized directly (Item 90).
**serialVersionUID** — explicit version identifier field for a serializable class; omitting it lets the JVM generate one, breaking compatibility on later changes (Item 87).
**Service provider framework** — decouples clients from service implementations via a service interface, registration API, and access API, e.g. JDBC (Item 1).
**Singleton** — instantiated exactly once, enforced via private constructor or (preferably) a single-element enum, which also handles serialization for free (Item 3).
**Static factory method** — a public static method returning an instance; can be named, need not create a new object each call, can return a subtype (Item 1).
**Static member class** — a nested class without an implicit outer-instance reference; preferred over nonstatic inner class to avoid a hidden reference that can leak memory (Item 24).
**Stream pipeline** — intermediate operations on a stream source followed by a terminal operation, ideally side-effect-free (Item 45, Item 46).
**Tagged class** — a class with a tag field for its "flavor"; verbose and inferior to a class hierarchy (Item 23).
**Telescoping constructor** — one constructor per parameter combination; unscalable and error-prone with same-typed parameters (Item 2).
**Template Method pattern** — a superclass/skeletal implementation defines an algorithm skeleton via abstract primitive methods subclasses override (Item 20).
**Typesafe heterogeneous container** — a container keyed by `Class<T>` letting clients store/retrieve arbitrarily many types with compile-time safety (Item 33).
**Unbounded wildcard** — `Collection<?>`, used instead of a raw type when the type parameter doesn't matter; only `null` insertable (Item 26).
**Unchecked exception** — signals a programming error the caller cannot be expected to handle (Item 70).
**Utility class** — solely static methods/fields, made noninstantiable via private constructor (Item 4).
**Varargs** — a `T...` parameter implicitly creating an array per call; combine with generics only under `@SafeVarargs` after verifying no array leak/store (Item 32, Item 53).
**Volatile** — guarantees cross-thread visibility of the latest write without mutual exclusion; misuse in read-modify-write still causes safety failures (Item 78).
**Weak reference** — doesn't prevent GC; used (e.g. `WeakHashMap`) so listener/callback entries reclaim automatically once otherwise unreferenced (Item 7).
**readObject defensive coding** — must validate invariants and defensively copy mutable fields, since deserialization is an attacker-drivable constructor (Item 88).
