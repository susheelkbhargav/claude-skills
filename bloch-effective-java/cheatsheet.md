# Cheatsheet — Effective Java (3rd ed.) Decision Rules

## Object creation
- **Many/optional ctor params (≥4)** → Builder (Item 2). 2–3 required only → static factory or ctor.
- **Need naming / instance control / interface return** → static factory over constructor (Item 1).
- **Need exactly one instance** → single-element `enum` (Item 3, 89). Never a reflection-vulnerable field singleton.
- **A class needs a resource** → inject it via constructor; don't `new` it internally (Item 5).
- **Class has no state, only statics** → private throwing ctor (Item 4).

## Immutability & inheritance
- **Default stance**: make the class immutable; make fields `private final`; if mutable is needed, minimize the mutable surface (Item 17).
- **Want to reuse another class's behavior** → compose + forward, don't extend — *unless* it's a true subtype under your control (Item 18).
- **Designing a public class for extension?** → document self-use (`@implSpec`), no overridable calls in ctor — or make it `final` (Item 19).
- **Type contract** → interface; **shared skeleton** → pair with `Abstract*` (Item 20).

## equals / hashCode / compareTo
- Override `equals` → **must** override `hashCode` (Item 10–11). Both or neither.
- `hashCode`: `result = 31*result + fieldHash` per significant field, or `Objects.hash(...)`.
- Comparators: use `Integer.compare` / `comparingInt(...).thenComparing(...)` — **never** `a - b` (overflow) (Item 14).
- Inheritance + value component → `equals` symmetry breaks → **compose** (Item 10).

## Generics — the wildcard rule
- **PECS**: param *produces* T → `? extends T`; param *consumes* T → `? super T` (Item 31).
- `Comparable`/`Comparator` params → always `? super T`.
- Raw type in new code → **never**; use `List<?>` (safe unknown) or `List<Object>` (Item 26).
- Arrays vs generics → prefer `List<E>` fields over `E[]` (Item 28).
- Unchecked warning → fix it, or `@SuppressWarnings("unchecked")` on **narrowest** scope + why-comment (Item 27).

## Enums over legacy idioms
| See this | Replace with |
|---|---|
| `int`/`String` constants | `enum` (34) |
| value from `ordinal()` | instance field (35) |
| `int` bit flags | `EnumSet` (36) |
| `array[ordinal()]` | `EnumMap` (37) |
| `testFoo` naming pattern | annotation (39) |
| `switch` on enum for behavior | constant-specific method (34) |

## Lambdas & streams
- Lambda ≤ ~3 lines and interface obvious → lambda; else anonymous class or named method (Item 42).
- Clearer as `Type::method`? → method reference (Item 43).
- Custom functional interface? → first check `java.util.function` (Item 44).
- `forEach` **computing** results (mutating a map/counter) → smell → move to a collector (`groupingBy`, `toMap`) (Item 46).
- Public API return type → `Collection`, not `Stream` (Item 47).
- `parallel()` → only on `ArrayList`/array/`IntStream.range`, side-effect-free, and **measured** (Item 48).

## Method design & return values
- Validate params at entry; `Objects.requireNonNull`; document with `@throws` (Item 49).
- Mutable in/out → defensive copy; **copy before validating** (Item 50).
- Param list > 4, or `boolean` flags → break up / use enums / builder (Item 51, 52).
- Two overloads same arity → **rename** instead (Item 52).
- No results: collection → **empty**, never null (Item 54); single maybe-absent → `Optional` (Item 55). Never `Optional<List>`, never `Optional` field/param.

## Exceptions
- Exception for control flow → **never** (Item 69).
- Recoverable → checked (sparingly); programming error / bad precondition → unchecked (Item 70–71).
- Bad arg → `IllegalArgumentException`; bad state → `IllegalStateException`; null → `NullPointerException` (Item 72).
- Low-level exception leaking your abstraction → translate + chain the cause (Item 73).
- Detail message → include failing param/field values (never secrets) (Item 75).
- Empty `catch {}` → **never**; if truly ignoring, name it `ignored` + comment why (Item 77).

## Concurrency
- Shared mutable data → synchronize **both** read and write (exclusion *and* visibility) (Item 78).
- Need visibility only → `volatile`; need atomic compound op → `Atomic*` / `synchronized` (Item 78).
- Inside a lock → do minimal work; **never call an alien method** (snapshot then act outside) (Item 79).
- Threads → prefer `ExecutorService` (Item 80); `wait`/`notify` → prefer `java.util.concurrent` (Item 81); if `wait`, always in a `while` loop.
- Lazy init → static: holder class; instance: double-check (`volatile`). Default = eager (Item 83).
- Correctness depending on priorities/scheduler/`yield` → it's broken (Item 84).

## General programming tells
- Money / exact math with `double` → wrong → `BigDecimal`/`int`/`long` (Item 60).
- `Long`/`Integer` in a loop or with `==` → boxing bug → use primitive (Item 61).
- String `+=` in a loop → O(n²) → `StringBuilder` (Item 63).
- `ArrayList x = ...` as a field/param type → prefer `List x` (interface) (Item 64).
- About to hand-roll date math / RNG / sorting → stop, use the library (Item 59).
- Optimizing before profiling → stop; measure first (Item 67).

## Serialization
- New system needing data interchange → JSON/protobuf, **not** Java serialization (Item 85).
- Deserializing untrusted bytes → never; if forced, `ObjectInputFilter` whitelist (Item 85).
- Implementing `Serializable` → treat as a permanent API commitment (Item 86).
- Physical rep ≠ logical → custom form: `transient` + `writeObject`/`readObject` + explicit `serialVersionUID` (Item 87).
- `readObject` → validate invariants + defensive-copy mutable fields (public-ctor discipline) (Item 88).
- Serializable class with invariants → serialization proxy pattern (Item 90).
