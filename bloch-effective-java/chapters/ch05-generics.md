# Chapter 5: Generics (Items 26–33)

## Core Idea
Use generics to catch type errors at compile time and eliminate casts. Understand erasure's consequences (raw types, unchecked warnings, arrays-vs-lists) and master bounded wildcards for flexible, safe APIs.

## Frameworks Introduced
- **Item 26 — Don't use raw types**: `List` (raw) opts out of generics and loses type safety; use `List<Object>` (holds anything, still checked) or `List<?>` (unbounded wildcard, holds unknown but safe). Raw types exist only for migration compatibility.
- **Item 27 — Eliminate unchecked warnings**: fix each or, if you can *prove* safety, suppress with `@SuppressWarnings("unchecked")` on the narrowest scope + a comment explaining why.
- **Item 28 — Prefer lists to arrays**: arrays are *covariant* and *reified*; generics are *invariant* and *erased*. Mixing (`new List<String>[]`) is illegal; prefer `List<E>` fields over `E[]` for compile-time safety.
- **Item 29 — Favor generic types**: parameterize your own classes (e.g. a `Stack<E>`) instead of using `Object` + casts.
- **Item 30 — Favor generic methods**: e.g. `static <E> Set<E> union(Set<E> s1, Set<E> s2)`; use the generic singleton factory and recursive type bounds (`<T extends Comparable<T>>`) where needed.
- **Item 31 — Use bounded wildcards to increase API flexibility**: **PECS** — Producer `extends`, Consumer `super`. A parameter that *produces* T uses `? extends T`; one that *consumes* T uses `? super T`.
- **Item 32 — Combine generics and varargs judiciously**: generic varargs create an unsafe heap-pollution hole; if your method is genuinely typesafe, annotate `@SafeVarargs`; never store into or expose the varargs array.
- **Item 33 — Consider typesafe heterogeneous containers**: parameterize the *key* (`Class<T>` as a type token) instead of the container, so one container safely holds many types: `favorites.put(Class<T>, T)` / `get(Class<T>) -> T`.

## Key Concepts
- **Erasure**: generic type info removed at runtime; `List<String>` and `List<Integer>` share one `List` class.
- **Raw type**: generic name used without type parameter — unsafe.
- **Reified vs erased**: arrays know their element type at runtime; generics don't.
- **Covariant vs invariant**: `String[]` is a `Object[]`; `List<String>` is *not* a `List<Object>`.
- **Bounded wildcard**: `? extends T` / `? super T`.
- **Heap pollution**: a variable of a parameterized type refers to an object of a different type.
- **Type token**: a `Class<T>` used as a key.

## Mental Models
- **PECS mnemonic**: if it's an *in* (produces values you read) use `extends`; if it's an *out* (you write values into it) use `super`. Comparables/Comparators are always consumers → `? super`.
- Prefer `List<?>` over raw `List` when you don't care about the element type but want safety.
- Think of arrays as "runtime-typed" and generics as "compile-time-typed" — don't mix.

## Anti-patterns
- **Raw types in new code**: loses all generic safety.
- **Ignoring/blanket-suppressing unchecked warnings**: hides real bugs; suppress narrowly with justification.
- **Generic array creation** (`new E[]` / `new List<E>[]`): illegal/unsafe; use `List<E>` or cast a raw array with a documented suppression.
- **`? super T` where you should read produced values**: you can only read `Object` out of a `super`-bounded wildcard.
- **Storing into or leaking a generic varargs array**: heap pollution.

## Code Examples
```java
// Item 31 — PECS in action
public void pushAll(Iterable<? extends E> src){ for (E e : src) push(e); } // src PRODUCES E
public void popAll(Collection<? super E> dst){ while(!isEmpty()) dst.add(pop()); } // dst CONSUMES E

// max with recursive bound + consumer comparator
public static <E extends Comparable<? super E>> E max(Collection<? extends E> c){ ... }
```
- **What it demonstrates**: `extends` for producers, `super` for consumers → callers can pass `Stack<Number>` sources of `Integer`, or drain into `Collection<Object>`.

```java
// Item 33 — Typesafe heterogeneous container
public class Favorites {
    private Map<Class<?>, Object> favorites = new HashMap<>();
    public <T> void put(Class<T> type, T instance){ favorites.put(type, type.cast(instance)); }
    public <T> T get(Class<T> type){ return type.cast(favorites.get(type)); }
}
// f.put(String.class, "Java"); String s = f.get(String.class); — one map, many types, all typesafe
```

## Reference Tables
| Situation | Wildcard | Reason |
|---|---|---|
| Param produces T (you read) | `? extends T` | Producer-extends |
| Param consumes T (you write) | `? super T` | Consumer-super |
| Don't care, need safety | `?` (unbounded) | Read as Object only |
| `Comparable`/`Comparator` | `? super T` | Always a consumer |

## Worked Example
Migrating a raw `Stack` to a generic type (Item 29). Starting from `Object[] elements` with casts everywhere, the generic version stores `E[]`. But `elements = new E[capacity]` is illegal (erasure). Two fixes: (a) create `Object[]` and cast once — `elements = (E[]) new Object[cap];` — with a `@SuppressWarnings("unchecked")` justified because the array is private and only ever holds `E`; or (b) type the field as `Object[]` and cast at each `pop`. Bloch prefers (a): one suppression at construction beats a cast per read, and the invariant ("this array only ever contains E") is easy to verify by inspection.

## Key Takeaways
1. Never use raw types in new code; use `List<Object>` or `List<?>`.
2. Eliminate every unchecked warning you can; suppress the rest narrowly with a comment.
3. Prefer lists to arrays — invariance + erasure keeps generics safe where arrays don't.
4. Parameterize your own types and methods to remove client casts.
5. Learn PECS cold — it's the single most useful generics rule for API design.
6. Type tokens enable one container to safely hold heterogeneous types.

## Connects To
- **Ch3 (Item 14)**: `Comparable<? super T>` uses PECS.
- **Ch4 (Item 20)**: generic interfaces define flexible types.
- **Ch7 (Item 44)**: standard functional interfaces are generic.
- **Ch8 (Item 53)**: varargs pitfalls (Item 32) overlap.
