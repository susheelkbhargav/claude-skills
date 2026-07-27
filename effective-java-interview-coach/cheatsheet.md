# Effective Java — Interview Cheatsheet

## Creation (1-9)
- Unclear ctor name / need caching or subtype return → static factory (1). 4+ params, some optional → **Builder** (2).
- Exactly one instance → private ctor + **enum** (3). Depends on a resource → ctor-inject, never static-utility/singleton (5).
- Reusing an expensive immutable object → cache `static final` (6). Own array (e.g. Stack) → null refs on pop, else leak (7).
- Frees a resource → `AutoCloseable`+try-with-resources; never finalizer/cleaner — no run guarantee, ~50x slower (8-9).

## equals/hashCode/toString/clone/compareTo (10-14)
- Override `equals` only if logical≠identity; contract: reflexive/symmetric/transitive/consistent. Adding a field via subclass breaks symmetry → compose, don't extend (10).
- **Always override `hashCode` with `equals`** — unequal hashcodes break hash collections (11). Override `toString` unless uninteresting (12).
- `Cloneable` is broken → prefer copy ctor/factory (13). Value classes → `Comparable`; never subtract ints, use `Integer.compare` (14).

## Classes & interfaces (15-25)
- Minimize accessibility of every member; accessors only (15-16).
- New value type → favor **immutability** — thread-safe free, cost is a new object per state change (17). "is-a" unclear/cross-package → **composition over inheritance** (18).
- Designing for subclassing → document self-use + design for extension, or **prohibit** (final) (19).
- Abstract class vs interface → prefer **interface**; abstract class only to share hierarchy-wide code (20).
- Tag field + switch → smell "tagged class" → hierarchy (23). One-outer-class helper → **static** member class (24).

## Generics — PECS (26-33)
- **PECS**: producer-`? extends T`, consumer-`? super T`; `Comparable`/`Comparator` are consumers → `Comparable<? super T>` (31).

## Enums (34-41)
- Enum > int constants. **Smell: `ordinal()` for business logic** → use instance field; `EnumSet`/`EnumMap` (34-37).

## Lambdas & streams (42-48)
- Stateless one-off function → lambda; needs name/state → real class (42). Calls one method → method reference (43).
- Streams suit uniform transform/filter/combine; skip if local mutable state/early break needed (45). Keep stream functions side-effect-free (46). Return `Collection`, not `Stream`, unless callers only stream it (47).
- **Parallel streams** pay off only on cheaply-splittable sources (`ArrayList`, array, `HashMap/HashSet`, `ConcurrentHashMap`, int/long ranges) + reduction/short-circuit ops. Never with `Stream.iterate`/`limit()`. Always measure (48).

## Methods (49-56)
- Validate params, fail fast (49). Defensive-copy untrusted mutable params/returns (50). Never return `null` for empty collection/array (54).
- `Optional<T>` for an expected "no value" case — never for collections/arrays/streams, never boxed-primitive (use `OptionalInt/Long/Double`), never as map key/value (55).

## General programming (57-68)
- Locals at narrowest scope; `for-each` over indexed loops unless indexing/lockstep needed (57-58).
- Prefer primitives over boxed — autoboxing in a hot loop: ~10x slower (61). Money → `int`/`long` cents or `BigDecimal`, never `float`/`double` (60). Smell: `+=` concat in a loop is O(n²) (63).

## Exceptions (69-77)
- Caller can recover? → **checked**. Programming error / unsure → **unchecked**; never define `Error` subclasses (70).
- Checked → boilerplate; consider `Optional`/boolean-check (71). Off-the-shelf exception fits → reuse (72). Crossing a layer → translate, chain cause (73). Smell: empty `catch` (77).

## Concurrency (78-84)
| Mechanism | Atomicity | Visibility | Use when |
|---|---|---|---|
| `synchronized` | Yes | Yes | Multi-step invariant |
| `volatile` | No (`i++` not atomic) | Yes | Single flag only |
| `j.u.c.atomic` | Yes (lock-free CAS) | Yes | Single-var counters/IDs |
| none | No | No | Never for shared mutable state (78) |

- Never call an "alien"/callback method inside a synchronized block (deadlock risk) — move it outside ("open call") (79).
- Prefer executors > raw `Thread`; `j.u.concurrent` utilities > `wait`/`notify` (80-81).
- Lazy init: don't unless costly+rarely used. Static field → holder-class idiom; instance field → double-check, field `volatile` (83).

## Serialization (85-90)
- Prefer non-Java serialization (JSON/protobuf) — deserializing untrusted streams is a code-exec risk; `Serializable` = near-permanent API commitment (85-86).
- Write `readObject` defensively, like a ctor facing hostile input (88). Singleton → enum, not bare `readResolve` (89); proxy pattern for nontrivial cases (90).
