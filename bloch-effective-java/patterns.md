# Patterns & Techniques — Effective Java (3rd ed.)

## Static Factory Method
**When to use**: need a named creation point, instance control (caching/singleton), or a return type that's an interface/subtype (Item 1).
**How**: `public static` method returning an instance; conventional names `from`, `of`, `valueOf`, `instance`, `getInstance`, `create`, `newInstance`.
**Trade-offs**: classes with only static factories can't be subclassed; harder to spot in docs than constructors.

## Builder Pattern
**When to use**: 4+ constructor params, many optional (Item 2).
**How**: static nested `Builder(requiredArgs)` + fluent setters returning `this` + `build()` returning the immutable outer object; validate in `build()`.
**Trade-offs**: more verbose than a constructor; extra `Builder` object per instance.

## Enum Singleton
**When to use**: any singleton, especially one that must survive serialization/reflection (Item 3, 89).
**How**: `public enum Elvis { INSTANCE; }`.
**Trade-offs**: can't lazily initialize; can't extend a class.

## Composition + Forwarding (Decorator)
**When to use**: you want inheritance-like reuse without fragility, especially across package boundaries (Item 18).
**How**: a `Forwarding*` class implements the interface and delegates to a wrapped instance; the wrapper subclasses the forwarder and adds behavior.
**Trade-offs**: one-time boilerplate for the forwarding class; unsuitable for callback frameworks (SELF problem).

## Skeletal Implementation (Template Method)
**When to use**: give implementors a head start on an interface (Item 20).
**How**: interface defines the type; an `Abstract*` class implements the boilerplate atop a few primitive methods.
**Trade-offs**: implementors either extend the skeleton (single inheritance) or forward to a private inner skeleton.

## PECS Wildcards
**When to use**: designing flexible generic APIs (Item 31).
**How**: producers → `? extends T`; consumers → `? super T`; `Comparable`/`Comparator` are consumers (`? super`).
**Trade-offs**: return types should not use wildcards (forces wildcards on callers).

## Typesafe Heterogeneous Container
**When to use**: one container must hold many types safely (Item 33).
**How**: parameterize the key with `Class<T>` (type token); store `Map<Class<?>, Object>`, cast on `get` via `type.cast(...)`.
**Trade-offs**: can't hold non-reifiable types (e.g. `List<String>.class` doesn't exist); relies on client not using raw `Class`.

## Constant-Specific Method Bodies (Enum Behavior)
**When to use**: behavior varies per enum constant (Item 34).
**How**: declare an `abstract` method in the enum; override per constant. For shared behavior use a **strategy enum** (delegate to a nested private enum).
**Trade-offs**: more verbose than a switch, but impossible to forget a constant.

## Extensible Enums via Interface
**When to use**: clients need to add "constants" (e.g. operation sets) (Item 38).
**How**: define an interface; multiple enums `implement` it; program to the interface.
**Trade-offs**: no inheritance of implementation between the enums.

## Defensive Copying
**When to use**: a class stores or returns mutable components across a trust boundary (Item 50, 88).
**How**: copy mutable params **before** the validity check (closes TOCTOU); copy again in accessors; prefer immutable component types (`Instant` over `Date`).
**Trade-offs**: allocation cost; unnecessary if components are immutable.

## Exception Translation + Chaining
**When to use**: a lower-layer exception would leak implementation or confuse callers (Item 73).
**How**: `catch (LowEx e) { throw new HighEx(context, e); }` — higher-level type, cause preserved.
**Trade-offs**: don't over-translate; sometimes propagating unchanged is right.

## Failure Atomicity
**When to use**: a method must not corrupt object state if it fails (Item 76).
**How**: validate before mutating; order failure-prone work first; operate on a copy; or use immutability.
**Trade-offs**: copy-based atomicity costs allocation; not always achievable (e.g. multi-thread `ConcurrentModification`).

## Executor Framework
**When to use**: running tasks concurrently (Item 80).
**How**: `ExecutorService ex = Executors.newFixedThreadPool(n); ex.submit(task); ex.shutdown();`. Submit `Runnable`/`Callable`; use `Future`/`CompletionService` for results.
**Trade-offs**: must size pools and handle shutdown; wrong pool type can deadlock.

## Lazy Initialization
**When to use**: init cost is high and the field is often unused (Item 83) — otherwise initialize eagerly.
**How**: static field → **holder class idiom**; instance field → **double-check idiom** (`volatile`); repeatable → single-check.
**Trade-offs**: complexity and subtle memory-model bugs; rarely worth it.

## Serialization Proxy Pattern
**When to use**: a `Serializable` class with invariants or defensive-copy needs (Item 90).
**How**: private static `SerializationProxy` captures logical state; enclosing class has `writeReplace` returning the proxy and a `readObject` that always throws; proxy's `readResolve` rebuilds via the public constructor.
**Trade-offs**: doesn't work for extendable classes or objects with circular references; small runtime cost.
