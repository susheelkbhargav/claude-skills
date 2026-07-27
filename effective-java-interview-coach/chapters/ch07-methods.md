# Chapter 8: Methods

## Item 49: Check parameters for validity

### Opening scenario
"Here's a BigInteger-style `mod(BigInteger m)` method. A caller passes -5 as the modulus. Where should that fail, and why does it matter where?"

### Follow-up probes
- Why validate at the top instead of letting the computation fail naturally?
- Public method vs. private helper — how does validation style differ?
- A factory stores an array reference for later use and gets null — why worse than a method that uses null immediately?
- When is it fine to skip an explicit check and rely on an implicit one (e.g., `Collections.sort`)?

### Naive attempt
Skip validation, or validate only some paths, trusting callers to "pass good data."

### What breaks
Three tiers, worst to least visible: the method returns normally but leaves an object corrupted (surfaces as an unrelated bug much later — a failure-atomicity violation, Item 76); returns normally with a silently wrong result; or fails with a confusing exception mid-computation instead of cleanly at the boundary. Worst case: a factory stores an unchecked null, and the `NullPointerException` fires later, far from the origin.

### The recommended approach & why
Validate at the top; document restrictions via `@throws` (`IllegalArgumentException`, `IndexOutOfBoundsException`, `NullPointerException`). Use `Objects.requireNonNull(x, msg)` — returns its input, so it doubles as check-and-assign. Java 9's `Objects.checkFromIndexSize/checkFromToIndex/checkIndex` cover index ranges but can't customize messages or handle closed ranges. For non-public methods, where you control every call site, use `assert` instead — throws `AssertionError`, costs nothing unless `-ea` is passed. Skip explicit checks when validation is implicit in the computation itself (sorting implicitly checks comparability) — but if the implicit check throws the *wrong* exception, apply exception translation (Item 73). Constructors especially must validate: never let an invalid constructor produce an object that violates its own invariants.

### Panel-ready answer checklist
- Three failure tiers, worst-first: corrupted state found far away > silent wrong answer > confusing mid-method exception.
- `Objects.requireNonNull` + the Java 9 range-check trio, and their limits.
- Public/protected (documented, defensive) vs. private/package (assertions, dev-time only).
- BigInteger's class-level Javadoc covering the implicit NPE, avoiding per-method clutter.

---

## Item 50: Make defensive copies when needed

### Opening scenario
"You built a `Period` class taking `Date start`/`Date end`, meant to be immutable. Later someone calls `end.setYear(78)` on the `Date` they originally passed in, and the 'immutable' `Period` silently changes. Why?"

### Follow-up probes
- Why not just document that callers shouldn't mutate the Date?
- What if an attacker mutates the object on another thread between your validity check and your copy?
- Is `Date.clone()` safe to use for the copy?
- The accessors still return the raw fields — what's the second attack?
- Is defensive copying only for immutable classes?

### Naive attempt
```java
public Period(Date start, Date end) {
    if (start.compareTo(end) > 0)
        throw new IllegalArgumentException(start + " after " + end);
    this.start = start; this.end = end;
}
public Date start() { return start; }
```

### What breaks
`Date` is mutable, so storing the caller's reference gives them a live handle to internal state: `end.setYear(78)` after construction mutates `p`. Even with a fixed constructor, unguarded accessors leak the same references — `p.end().setYear(78)` is the second attack.

### The recommended approach & why
Copy every mutable parameter in, copy mutable fields out:
```java
public Period(Date start, Date end) {
    this.start = new Date(start.getTime());
    this.end   = new Date(end.getTime());
    if (this.start.compareTo(this.end) > 0)
        throw new IllegalArgumentException(this.start + " after " + this.end);
}
public Date start() { return new Date(start.getTime()); }
```
Copy **before** validating and validate the copies — otherwise a TOCTOU (time-of-check/time-of-use) window lets another thread mutate the parameter between check and copy. Never use `clone()` on a parameter whose class is nonfinal/subclassable by untrusted parties — a malicious subclass's `clone()` could return a rigged instance. (Accessors may use `clone()` safely since the field's exact class is known; a constructor/factory copy is still generally preferred, Item 13.) Best fix: use immutable components (`Instant`, not `Date`) so there's nothing to defend. Applies beyond immutability too — any stored client-supplied mutable reference (a `Set` element, `Map` key) or returned internal component may need copying if post-hoc mutation would corrupt an invariant. Skip only with mutual trust or an explicit, documented ownership handoff.

### Panel-ready answer checklist
- Two attacks: mutate the constructor arg after construction; mutate the object returned by an accessor.
- TOCTOU, and why copy-then-validate-the-copy is the safe order.
- `clone()` unsafe for untrusted nonfinal types, safe when the exact class is known.
- Real fix: immutable components (Item 17) eliminate the problem entirely.
- Generalizes to any stored/returned mutable reference, including arrays (always mutable).

---

## Item 51: Design method signatures carefully

### Opening scenario
"Review `createUser(String, String, String, String, boolean, boolean, int, String)`. What's wrong, and what would you change?"

### Follow-up probes
- Why "four parameters or fewer," and what specifically breaks with long identically-typed sequences?
- Give three techniques to shorten a parameter list.
- Why favor `Map` over `HashMap` as a parameter type?
- Why prefer a two-element enum over a boolean parameter?

### Naive attempt
Add a boolean flag per mode switch, let one method's parameter list keep growing, add a convenience overload for every observed call pattern.

### What breaks
Long identically-typed lists compile fine even transposed — `createUser(true, false)` silently does the wrong thing, no compiler help. Too many convenience methods make a class hard to learn/test/maintain (doubly true for interfaces). `Thermometer.newInstance(true)` is meaningless next to `newInstance(CELSIUS)`, and a boolean can't gain a third option (`KELVIN`) later without a new factory.

### The recommended approach & why
Name methods per standard conventions and library consensus. Don't over-provide convenience methods — each must "pull its weight." Shorten long lists via: (1) split into orthogonal methods (`List.subList` + `indexOf`, composable); (2) a helper class for a parameter group that recurs as a distinct entity (rank+suit → `Card`), typically a static member class; (3) adapt Builder from construction to invocation — setter calls, then one `execute()`. Favor interface parameter types (`Map`, not `HashMap`) so callers aren't forced into one implementation or an expensive copy. Favor a two-element enum over `boolean` unless the boolean's meaning is obvious from the method name — enums read better and extend cleanly.

### Panel-ready answer checklist
- ≤4-parameter guideline; identically-typed sequences are the worst offender (silent transposition bugs).
- All three shortening techniques with an example each.
- Interface-typed parameters avoid forcing callers into one implementation or a copy.
- `TemperatureScale` enum-vs-boolean example plus the extensibility argument.

---

## Item 52: Use overloading judiciously

### Opening scenario
"Three overloads of `classify` — one each for `Set<?>`, `List<?>`, `Collection<?>` — run against a `HashSet`, an `ArrayList`, and `Map.values()`. It prints 'Unknown Collection' three times. Why?"

### Follow-up probes
- Why does overriding (the `Wine`/`SparklingWine`/`Champagne` example) behave differently?
- What's the "safe, conservative policy" for exporting overloads?
- Constructors can't be renamed — how do you safely export several with the same arity?
- What broke with `List.remove(int)` vs `remove(Object)` once generics/autoboxing arrived?

### Naive attempt
Overload `classify(Set<?>)`, `classify(List<?>)`, `classify(Collection<?>)`, expecting the most-specific version to run based on runtime type, as with overriding.

### What breaks
Overload resolution happens at **compile time**, based on the argument's declared type — the loop variable is statically `Collection<?>`, so only the third overload is ever applicable regardless of actual runtime type. Overriding is the opposite: dynamic dispatch on runtime type. Concretely: `List<Integer>` has both `remove(E)` and `remove(int index)`; `list.remove(i)` on an `int` picks positional removal, while `set.remove(i)` autoboxes to value removal — same-looking calls, different semantics, no compiler warning.

### The recommended approach & why
Fix the classifier with one method and explicit `instanceof` checks — overloading can't provide runtime dispatch. Rule: **never export two overloads with the same parameter count**; never overload a varargs method (Item 53 exception). Where same-arity overloads are unavoidable (constructors always are), ensure every pair has at least one "radically different" parameter position — impossible to cast a non-null expression to both (`ArrayList(int)` vs `ArrayList(Collection)`). Autoboxing broke "primitives are radically different from references," which is exactly what bit `List.remove`. Lambdas made it worse: don't overload with different functional interfaces in the same position (`Runnable` vs `Callable<T>`) — inexact method references like `System.out::println` are excluded from applicability testing until a target type is chosen. If retrofitting forces a same-arity overload (`String.contentEquals(CharSequence)` alongside `contentEquals(StringBuffer)`), make the specific one forward to the general one so behavior is identical.

### Panel-ready answer checklist
- Overload resolution is static/compile-time; override resolution is dynamic/runtime.
- Conservative rule: no same-arity overloads, no overloaded varargs methods.
- "Radically different" definition plus the `ArrayList` example.
- `Set.remove(i)` vs `List.remove(i)` autoboxing trap and its fix (cast to `Integer`).
- Functional-interface overload trap (`Runnable`/`Callable`) and the forwarding pattern for unavoidable overloads.

---

## Item 53: Use varargs judiciously

### Opening scenario
"A junior engineer writes `min(int... args)` with a runtime `if (args.length == 0) throw ...`. What's wrong with this design, independent of correctness?"

### Follow-up probes
- What happens under the hood on every varargs call, and why does it matter on a hot path?
- Redesign `min` so "at least one argument" is a compile-time guarantee.
- What does `EnumSet` do to avoid the array-allocation cost?
- Can you overload a varargs method safely?

### Naive attempt
```java
static int min(int... args) {
    if (args.length == 0) throw new IllegalArgumentException("Too few arguments");
    int min = args[0];
    for (int i = 1; i < args.length; i++) if (args[i] < min) min = args[i];
    return min;
}
```

### What breaks
`min()` with zero arguments compiles cleanly and fails only at **runtime** — "one or more" is enforced by an `if`, not the type system. Also uglier: an explicit check, no clean for-each without an `Integer.MAX_VALUE` seed hack.

### The recommended approach & why
Split into a required first parameter plus a varargs tail:
```java
static int min(int firstArg, int... remainingArgs) {
    int min = firstArg;
    for (int arg : remainingArgs) if (arg < min) min = arg;
    return min;
}
```
Zero arguments is now a compile error. More generally: every varargs call allocates and initializes an array, so on a *measured* hot path, use the fixed-arity overload ladder — declare separate overloads for 0..3 params plus one varargs overload for the rare >3 case, paying the allocation only on the tail. `EnumSet`'s static factories do this to compete with bit fields (Item 36) on performance. Precede varargs with any required parameters, and per Item 52, don't overload a varargs method except via this ladder.

### Panel-ready answer checklist
- Varargs mechanism: array allocated + populated + passed, every call.
- `(T first, T... rest)` fixes "one or more" at compile time, not via a runtime check.
- Fixed-arity overload ladder, with `EnumSet` as the real justification.
- Varargs' sweet spot: `printf`-style and reflection-style APIs with genuinely variable arg counts.

---

## Item 54: Return empty collections or arrays, not nulls

### Opening scenario
"`getCheeses()` returns `null` when the shop has no stock, a populated list otherwise. Every caller needs a null check. What's wrong, and does returning null actually save anything?"

### Follow-up probes
- Isn't null cheaper than allocating an empty collection every time?
- If you've actually measured allocation as a bottleneck, what's the fix that avoids null?
- Does the same argument apply to arrays?
- Is preallocating the array passed to `toArray` to the exact size a good idea?

### Naive attempt
```java
public List<Cheese> getCheeses() {
    return cheesesInStock.isEmpty() ? null : new ArrayList<>(cheesesInStock);
}
```

### What breaks
Every caller needs defensive null-handling (`if (cheeses != null && cheeses.contains(...))`). This is needed at nearly every call site; forgetting it is easy since most calls return non-null, so the bug hides for years until the empty case occurs — surfacing as an NPE far from its root cause. The "saves an allocation" justification fails twice: don't optimize before measuring (Item 67), and it's avoidable without ever returning null.

### The recommended approach & why
Just return the possibly-empty container:
```java
public List<Cheese> getCheeses() { return new ArrayList<>(cheesesInStock); }
```
With actual evidence of a real cost, return a shared immutable empty collection (immutables are freely shareable, Item 17): `cheesesInStock.isEmpty() ? Collections.emptyList() : new ArrayList<>(cheesesInStock)`. Same for arrays — never return null for zero-length; return the correct-length array: `cheesesInStock.toArray(new Cheese[0])`, or reuse one preallocated zero-length array across calls (all zero-length arrays are immutable). Do **not** presize the array passed to `toArray` — studies show it actually hurts performance rather than helping.

### Panel-ready answer checklist
- Rule: never return null in place of an empty collection or array.
- Rebut "saves an allocation": premature optimization, and avoidable anyway.
- `Collections.emptyList/emptySet/emptyMap` and the shared zero-length-array pattern.
- Presizing the `toArray` argument is a documented anti-pattern, not an optimization.

---

## Item 55: Return optionals judiciously

### Opening scenario
"`max(Collection<E>)` throws `IllegalArgumentException` on empty input. Someone proposes returning `Optional<E>` instead. What actually improves, and is `Optional` always the right call for 'might not have a result'?"

### Follow-up probes
- What's expensive about the exception-throwing approach?
- Why not just return null instead of wrapping in `Optional`?
- Should you ever wrap a `List`, `Map`, or another `Optional` inside an `Optional`?
- What's wrong with `Optional<Integer>` vs `OptionalInt`?
- When should you actually use `isPresent()` vs. `map`/`orElse`?

### Naive attempt
```java
public static <E extends Comparable<E>> E max(Collection<E> c) {
    if (c.isEmpty()) throw new IllegalArgumentException("Empty collection");
    ...
}
```
Or the null-returning equivalent, pushing a manual check onto every caller.

### What breaks
Exceptions should be reserved for exceptional conditions (Item 69) and are expensive — the full stack trace is captured at creation. Null avoids that cost but reintroduces Item 54's problem: callers must remember a special-case check, and a missed one surfaces as an NPE somewhere unrelated, later.

### The recommended approach & why
Return `Optional<E>`:
```java
public static <E extends Comparable<E>> Optional<E> max(Collection<E> c) {
    if (c.isEmpty()) return Optional.empty();
    E result = null;
    for (E e : c) if (result == null || e.compareTo(result) > 0) result = Objects.requireNonNull(e);
    return Optional.of(result);
}
```
`Optional<T>` is an immutable container holding one non-null `T` or nothing; similar in spirit to checked exceptions — it forces the caller to confront the "no value" case, unlike an unchecked exception or null. Never let an `Optional`-returning method itself return null. The caller decides: `.orElse(default)`, `.orElseThrow(ExceptionFactory::new)` (a factory, not an instance, avoids paying creation cost unless thrown), or `.get()` if nonemptiness is provable. Use `orElseGet(Supplier)` when the default is expensive to compute. Prefer `map`/`filter`/`flatMap`/`ifPresent`/`or`/`ifPresentOrElse` over manual `isPresent()` checks, which are a "safety valve," rarely the clearest form. `Optional::stream` (Java 9) bridges `Stream<Optional<T>>` to `Stream<T>` via `flatMap`.
Boundaries: never wrap container types (collections, maps, streams, arrays, `Optional` itself) in `Optional` — return the empty container directly (Item 54). Never return `Optional` of a boxed primitive — use `OptionalInt`/`OptionalLong`/`OptionalDouble` instead (minor primitives like `Boolean`/`Byte`/`Character`/`Short`/`Float` are tolerable exceptions). Never use `Optional` as a map value (two ways to express absence) or, almost never, as a key/collection element. Storing one in a field is usually a "bad smell" but can be legitimate for classes with many genuinely-optional primitive-typed fields (the `NutritionFacts` case). Cost tradeoff: `Optional` requires its own allocation plus an extra read indirection — measure before using it on a performance-critical path.

### Panel-ready answer checklist
- `Optional` as a third option beside exceptions (expensive, exceptional-only) and null (silently ignorable).
- Absolute rule: never return null from an `Optional`-returning method.
- `orElse` vs `orElseGet` vs `orElseThrow(factory)` vs `.get()`, and when each applies.
- Don't-wrap list: containers, boxed primitives (use `OptionalInt` etc.), map values/keys/elements.
- Real cost (allocation + indirection) — measure, don't assume, on hot paths.

---

## Item 56: Write doc comments for all exposed API elements

### Opening scenario
"You inherit a public library class with zero Javadoc. What does Javadoc actually fall back to, and what should precede every exported class/interface/constructor/method/field?"

### Follow-up probes
- What goes in a method's doc comment beyond a one-liner — preconditions, postconditions, side effects?
- What's `@implSpec` for, and how does its intent differ from a normal doc comment?
- Why can't a public class rely on a default constructor if you care about documentation?
- Give the doc-comment period/abbreviation gotcha and its fix.
- Should overloaded methods share a summary description?

### Naive attempt
Skip doc comments on "obvious" methods, omit `@param`/`@throws`/`@return`, or reuse the same summary sentence across overloads.

### What breaks
With no doc comment, Javadoc falls back to reproducing the bare declaration — frustrating and error-prone. Two members sharing a summary description (easy with overloads) causes real confusion. A doc-comment period followed by a space/tab/line-break truncates the *summary description* right there — `"A college degree, B.S., M.S. or Ph.D."` gets cut to `"A college degree, B.S., M.S."` because Javadoc treats the first sentence-ending period as the boundary.

### The recommended approach & why
Precede every exported class, interface, constructor, method, and field with a doc comment (most unexported ones too, less thoroughly). Public classes shouldn't rely on default constructors — there's no way to attach a comment to one. A method's comment states the **contract** (what, not how) except for inheritance-designed classes (Item 19), which use `@implSpec` (Java 8+) to describe the contract with **subclasses** specifically:
```java
/**
 * Returns true if this collection is empty.
 * @implSpec This implementation returns {@code this.size() == 0}.
 * @return true if this collection is empty
 */
```
Document preconditions (largely implicit via `@throws` for unchecked exceptions), postconditions, and side effects (e.g., "starts a background thread"). Use `@param` for every parameter, `@return` unless void, `@throws` for every exception, checked or unchecked. Use `{@code}` for code font plus suppressed markup processing; for multi-line examples wrap `<pre>{@code ... }</pre>`. Use `{@literal}` to suppress markup without switching to code font — e.g., around a stray abbreviation period (`{@literal M.S.}`) to prevent summary truncation. Never give two members in a class the same summary description, even similar-behaving overloads. Summary style: verb phrase for methods/constructors ("Returns the number of elements..."), noun phrase for classes/interfaces/fields ("An instantaneous point on the time-line."). Document every type parameter on generics (`@param <K> ...`), every constant on enums, every member on annotations. Package/module docs live in `package-info.java`/`module-info.java`. Document thread-safety (Item 82) and serialized form (Item 87) where applicable. `{@inheritDoc}` reuses an interface's comment instead of duplicating it.

### Panel-ready answer checklist
- No doc comment means Javadoc just reproduces the bare declaration.
- `@implSpec` (method-to-subclass contract) vs. ordinary comments (method-to-client contract).
- Four documentable contract pieces: preconditions, postconditions, side effects, thrown exceptions.
- `{@code}` (code font + suppress markup) vs. `{@literal}` (suppress markup only), plus the period-truncation gotcha.
- Summary convention: verb phrase for methods/constructors, noun phrase for types/fields; no duplicate summaries across overloads.
