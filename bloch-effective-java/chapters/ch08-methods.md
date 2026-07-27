# Chapter 8: Methods (Items 49–56)

## Core Idea
Method design is API design. Validate inputs, defend against hostile callers, keep signatures small and unambiguous, and return values that don't surprise (no nulls where a caller least expects them).

## Frameworks Introduced
- **Item 49 — Check parameters for validity**: fail fast at the start of the method with clear exceptions. Document constraints (`@throws`), use `Objects.requireNonNull`, and for public methods throw `IllegalArgumentException`/`NullPointerException`/`IndexOutOfBoundsException`. Private methods can use `assert`.
- **Item 50 — Make defensive copies when needed**: if a class has mutable components supplied/returned across its boundary, copy them (before the validity check, to avoid TOCTOU) so clients can't mutate internals. Prefer immutable component types.
- **Item 51 — Design method signatures carefully**: choose method names per conventions; don't over-provide convenience methods; keep parameter lists short (≤4); prefer interfaces to classes and two-element enums to `boolean` parameters.
- **Item 52 — Use overloading judiciously**: overloading is resolved at *compile time* by static type — surprising. Avoid two overloads with the same arg count; give methods different names when in doubt (e.g. `writeInt`/`writeLong`).
- **Item 53 — Use varargs judiciously**: great for variable arg counts, but every call allocates an array. For methods needing ≥1 arg, take `(Type first, Type... rest)`. Consider fixed overloads for the common 0–3 arg cases when performance matters.
- **Item 54 — Return empty collections or arrays, not nulls**: null returns force callers into null-checks and cause NPEs. Return `Collections.emptyList()` / a zero-length array (`toArray(new T[0])`).
- **Item 55 — Return optionals judiciously**: `Optional<T>` for "no result" where the caller must actively handle absence; never wrap collections/arrays in Optional, never `Optional` a boxed primitive (use `OptionalInt` etc.), and don't use it for fields or method parameters.
- **Item 56 — Write doc comments for all exposed API elements**: document every public/protected class, interface, method, field. State the contract (pre/postconditions via `@param`/`@return`/`@throws`), thread-safety, and side effects.

## Key Concepts
- **Fail-fast**: detect errors as soon as possible, near their source.
- **Defensive copy**: copying mutable inputs/outputs to preserve invariants.
- **TOCTOU (time-of-check/time-of-use)**: window where a checked value is changed before use — copy *then* validate the copy.
- **Overload vs override**: overload = static dispatch (compile-time), override = dynamic dispatch (runtime).
- **`Optional<T>`**: a container for "present or absent," an alternative to null or exceptions.

## Mental Models
- A public method is a contract; its Javadoc *is* the spec. Write the contract, then the code.
- Assume callers are hostile or careless: validate and defensively copy at trust boundaries.
- Prefer *two named methods* to two same-arity overloads — overloading resolves on declared type, not runtime type.
- "No result" ⇒ empty collection (for multi) or `Optional` (for single); never `null` for a collection.

## Anti-patterns
- **Skipping parameter validation**: errors surface deep inside, or worse, corrupt state silently.
- **Storing/returning references to mutable inputs/outputs**: clients mutate your internals (breaks immutability).
- **Long parameter lists / `boolean` flags**: unreadable call sites (`setState(true, false, true)`) → use enums / helper types / builders.
- **Ambiguous overloads** (`remove(int)` vs `remove(Object)` on `List`): silent wrong dispatch → distinct names.
- **Returning `null` for absent collection**: NPE landmine → empty collection.
- **`Optional` fields / params / `Optional<List>`**: misuse; adds cost without benefit.

## Code Examples
```java
// Item 50 — defensive copy: copy THEN validate the copy
public Period(Date start, Date end) {
    this.start = new Date(start.getTime());  // copy first (Date is mutable)
    this.end   = new Date(end.getTime());
    if (this.start.compareTo(this.end) > 0)  // validate the COPY, not the original
        throw new IllegalArgumentException(this.start + " after " + this.end);
}
public Date start(){ return new Date(start.getTime()); } // defensive copy out, too
```
- **What it demonstrates**: closing the TOCTOU window by copying before the check and copying on the way out.

```java
// Item 54 & 55
public List<Cheese> getCheeses(){ return new ArrayList<>(cheesesInStock); } // never null
public Optional<Cheese> max(Collection<Cheese> c){
    return c.isEmpty() ? Optional.empty() : Optional.of(Collections.max(c));
}
```

## Reference Tables
| "No result" case | Return | Item |
|---|---|---|
| Collection/array, none | empty collection / zero-len array | 54 |
| Single value, maybe absent, caller must handle | `Optional<T>` | 55 |
| Single primitive, maybe absent | `OptionalInt`/`OptionalLong`/`OptionalDouble` | 55 |
| Absence is an error | throw exception | 55 |

## Worked Example
The mutable-`Date` attack (Item 50). `Period` looks immutable — `final Date start, end` set in the ctor. But `Date` is mutable, so an attacker keeps a reference: `Date d = new Date(); Period p = new Period(d, d); d.setYear(78);` — now `p`'s internal `start` moved. Even with the ctor fixed, `p.end().setYear(78)` mutates internals via the *returned* reference. Both holes require defensive copies: copy every mutable field **on the way in** (before validation) and **on the way out** (in accessors). The permanent fix is to use immutable component types (`Instant`/`LocalDateTime`) so no copying is needed at all.

## Key Takeaways
1. Validate parameters at method entry; document the constraints as part of the contract.
2. Defensively copy mutable inputs/outputs — copy before validating; prefer immutable component types.
3. Keep signatures small: short param lists, interface types, enums over booleans.
4. Avoid same-arity overloads; name distinctly to keep dispatch predictable.
5. Never return null for a collection/array; return empty. Use `Optional` for single maybe-absent values, not fields/params/collections.
6. Every exposed API element needs a doc comment that states its full contract.

## Connects To
- **Ch4 (Item 17)**: defensive copies preserve immutability.
- **Ch2 (Item 2)**: builders tame long parameter lists.
- **Ch6 (Item 34)**: enum params beat boolean flags.
- **Ch10 (Item 72/74)**: parameter checks throw standard, documented exceptions.
