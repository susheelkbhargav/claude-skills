# Glossary — Effective Java (3rd ed.)

**Alien method** — a method you don't control (client callback or overridable method); calling one while holding a lock risks deadlock/corruption (Ch11, Item 79).

**Autoboxing / unboxing** — implicit conversion between primitives and their wrappers; source of NPEs and silent slowdowns (Ch9, Item 61).

**Bounded wildcard** — `? extends T` (upper) or `? super T` (lower); increases API flexibility (Ch5, Item 31).

**Builder pattern** — static nested `Builder` with fluent setters + `build()`; for many/optional constructor params (Ch2, Item 2).

**Checked exception** — compiler-enforced exception for recoverable conditions (Ch10, Item 70).

**Cleaner** — post-`finalize` cleanup mechanism (`java.lang.ref.Cleaner`); use only as a safety net (Ch2, Item 8).

**Collector** — recipe for a stream's terminal reduction: `toList`, `groupingBy`, `toMap`, `joining`, `counting` (Ch7, Item 46).

**Composition + forwarding** — wrapping an instance and delegating calls (decorator); the robust alternative to inheritance (Ch4, Item 18).

**Constant interface antipattern** — using an interface only to hold constants; leaks into API (Ch4, Item 22).

**Constant-specific method body** — per-constant override of an abstract method in an enum (Ch6, Item 34).

**Defensive copy** — copying mutable inputs/outputs to protect invariants; copy *before* validating (Ch8, Item 50; Ch12, Item 88).

**Dependency injection (DI)** — passing resources into a constructor rather than hardwiring them (Ch2, Item 5).

**Double-check idiom** — lazy instance-field init using a `volatile` field with two null checks (Ch11, Item 83).

**EnumMap / EnumSet** — array-backed, high-performance Map/Set for enum keys; replace ordinal indexing and bit fields (Ch6, Items 36–37).

**Erasure** — removal of generic type info at runtime; reason generics are invariant/unreified (Ch5, Item 28).

**Exception chaining** — constructing a higher-level exception with the low-level cause (`new Ex(cause)`) (Ch10, Item 73).

**Exception translation** — catching a low-level exception and rethrowing one meaningful at the current abstraction (Ch10, Item 73).

**Failure atomicity** — a failed method leaves the object in its pre-call state (Ch10, Item 76).

**Fail-fast** — detecting errors as early/close to the source as possible (Ch8, Item 49).

**Finalizer** — deprecated `finalize()` cleanup; unpredictable, avoid (Ch2, Item 8).

**Fluent API** — chained method calls returning `this` (Ch2, Item 2).

**Functional interface** — interface with a single abstract method (SAM), implementable by a lambda (Ch7, Item 44).

**Gadget chain / deserialization bomb** — crafted byte stream that executes code or exhausts resources during deserialization (Ch12, Item 85).

**General contract** — behavioral spec of `equals`/`hashCode`/`compareTo` that callers depend on (Ch3, Items 10–11, 14).

**Generic method** — method with its own type parameter(s), e.g. `<E> Set<E> union(...)` (Ch5, Item 30).

**Heap pollution** — a parameterized-type variable pointing at an object of a different type (Ch5, Item 32).

**Holder class idiom** — lazy static-field init via a nested class initialized on first use (Ch11, Item 83).

**Immutability** — state fixed at construction; inherently thread-safe and shareable (Ch4, Item 17).

**Information hiding / encapsulation** — decoupling components by minimizing accessibility (Ch4, Item 15).

**Marker interface** — empty interface defining a type/property (e.g. `Serializable`) (Ch6, Item 41).

**Method reference** — `::` shorthand for a lambda that just calls one method (Ch7, Item 43).

**Nonstatic member class** — inner class holding a hidden enclosing-instance reference; prefer `static` (Ch4, Item 24).

**Obsolete reference** — a reference the program never dereferences again yet still holds, blocking GC (Ch2, Item 7).

**Optional<T>** — container for present-or-absent single value; not for fields/params/collections (Ch8, Item 55).

**Ordinal** — an enum constant's declaration position; an implementation detail, never app data (Ch6, Item 35).

**PECS** — "Producer-`extends`, Consumer-`super`"; wildcard-selection mnemonic (Ch5, Item 31).

**Premature optimization** — tuning before measuring, at the cost of design/clarity (Ch9, Item 67).

**Raw type** — a generic used without a type parameter (e.g. `List`); unsafe, migration-only (Ch5, Item 26).

**readObject** — deserialization "magic constructor"; treat as a public ctor fed hostile bytes (Ch12, Item 88).

**readResolve / writeReplace** — hooks substituting an object on read/write; basis of the proxy pattern (Ch12, Items 89–90).

**Reified vs erased** — arrays know their element type at runtime (reified); generics don't (erased) (Ch5, Item 28).

**Serialization proxy pattern** — private nested class capturing logical state, reconstructed via the public ctor on read (Ch12, Item 90).

**serialVersionUID** — explicit version id controlling serialization compatibility (Ch12, Item 87).

**Skeletal implementation** — `Abstract*` class implementing an interface's boilerplate (Template Method) (Ch4, Item 20).

**Static factory method** — static method returning an instance; not the GoF Factory Method (Ch2, Item 1).

**Strategy enum** — enum delegating shared per-constant behavior to a nested private strategy enum (Ch6, Item 34).

**Tagged class** — class encoding multiple flavors via a type field + switch; replace with subclasses (Ch4, Item 23).

**Telescoping constructor** — constructor overloads adding one param each; unreadable at scale (Ch2, Item 2).

**Thread safety levels** — immutable, unconditionally thread-safe, conditionally thread-safe, not thread-safe, thread-hostile (Ch11, Item 82).

**TOCTOU** — time-of-check/time-of-use gap; close it by copying then validating the copy (Ch8, Item 50).

**Type token** — a `Class<T>` used as a key in a typesafe heterogeneous container (Ch5, Item 33).

**Unchecked exception** — `RuntimeException`/subclass for programming errors; not required to be caught (Ch10, Item 70).

**Unchecked warning** — compiler warning about an unverifiable generic operation; eliminate or suppress narrowly (Ch5, Item 27).

**Utility class** — non-instantiable class of static members; private throwing ctor (Ch2, Item 4).

**volatile** — guarantees visibility/ordering of a field, not atomicity of compound ops (Ch11, Item 78).
