# Chapter 3: Methods Common to All Objects

## Item 10: Obey the general contract when overriding equals

### Opening scenario
"You put an object as a key in a HashMap, then look it up with an equal-but-different instance — `map.get` returns `null`. Why? Walk me through what the JVM actually does on that lookup, step by step, before you tell me the fix."

### Follow-up probes
- Why not just leave `equals` at reference identity for every class — what's actually gained by overriding it?
- Give me a case where overriding `equals` is flat-out wrong to do. What's the tell?
- You said "compare all fields" — what happens if I add a `color` field to a subclass of `Point` and write `equals` to check it? Walk me through `p.equals(cp)` vs `cp.equals(p)`.
- Suppose I "fix" symmetry by making the subclass do a color-blind comparison when the argument is a plain `Point`. What property do I now break, and can you construct the three objects that prove it?
- Why is `getClass()` instead of `instanceof` also wrong, even though it makes symmetry and transitivity easy? What principle does it violate?
- If I must add a value component to an existing value class, what's the actual correct way to do it?
- Why doesn't `equals(Object o)` guard with `if (o == null) return false;` before the `instanceof` check — what makes that line dead weight?

### Naive attempt
A mediocre candidate says: "Override equals, cast the argument to my type, compare the fields I care about." They write:

```java
public boolean equals(Object o) {
    ColorPoint cp = (ColorPoint) o;
    return cp.color == color;
}
```

No type check, no null handling, no consideration of what happens when compared against the superclass or an unrelated type — and no idea there's a five-part contract at all.

### What breaks
- **No `instanceof` check** → `ClassCastException` the first time someone passes the wrong type into `equals`.
- **`getClass()` test instead of `instanceof`** → breaks the Liskov substitution principle. A `CounterPoint extends Point` (adding no value component, just an instance counter) suddenly fails `unitCircle.contains(counterPointInstance)` even though it's logically a `Point` at that x,y — because `getClass()` demands identical implementation classes, not just logical equivalence.
- **Adding a value component via inheritance** (`ColorPoint extends Point`, overriding `equals` to add `color`) → symmetry breaks: `p.equals(cp)` (color-blind, inherited-style) returns `true` while `cp.equals(p)` returns `false` because `o instanceof ColorPoint` fails for a plain `Point` argument.
- **"Fixing" symmetry with a color-blind mixed comparison** → transitivity breaks:
  ```java
  ColorPoint p1 = new ColorPoint(1, 2, Color.RED);
  Point      p2 = new Point(1, 2);
  ColorPoint p3 = new ColorPoint(1, 2, Color.BLUE);
  ```
  `p1.equals(p2)` and `p2.equals(p3)` both return `true`, but `p1.equals(p3)` returns `false` — RED equals the colorless point, colorless equals BLUE, but RED does not equal BLUE. Two subclasses of `Point` each doing this can even trigger `StackOverflowError` via infinite mutual recursion.
- **Real-world casualty**: `java.sql.Timestamp extends java.util.Date` and adds a nanoseconds field — its `equals` violates symmetry, and Bloch calls this "a mistake and should not be emulated."
- **Unreliable-resource comparisons** → consistency breaks. `java.net.URL.equals` compares resolved IP addresses, which can change over time or depend on network access — "a big mistake" baked in for compatibility reasons.

### The recommended approach & why
The equals contract, verbatim, is an equivalence relation with five properties:

- **Reflexive**: For any non-null reference value `x`, `x.equals(x)` must return `true`.
- **Symmetric**: For any non-null reference values `x` and `y`, `x.equals(y)` must return `true` if and only if `y.equals(x)` returns `true`.
- **Transitive**: For any non-null reference values `x`, `y`, `z`, if `x.equals(y)` returns `true` and `y.equals(z)` returns `true`, then `x.equals(z)` must return `true`.
- **Consistent**: For any non-null reference values `x` and `y`, multiple invocations of `x.equals(y)` must consistently return `true` or consistently return `false`, provided no information used in equals comparisons is modified.
- For any non-null reference value `x`, `x.equals(null)` must return `false`.

When to skip overriding `equals` entirely: each instance is inherently unique (e.g. `Thread`); no need for logical equality (e.g. `Pattern`); a superclass's `equals` is already appropriate (most `Set`/`List`/`Map` implementations inherit from `AbstractSet`/`AbstractList`/`AbstractMap`); the class is private/package-private and `equals` will never be invoked (can even throw `AssertionError` in the body to be sure).

When it *is* appropriate: the class has a notion of **logical equality** distinct from object identity, and no superclass has already overridden `equals` correctly — this is the case for **value classes** (`Integer`, `String`-like types). Exception: instance-controlled classes (Item 1) and enums, where each value has exactly one instance, so identity equality *is* logical equality.

The value-component problem has no clean fix through inheritance: "There is no way to extend an instantiable class and add a value component while preserving the equals contract, unless you're willing to forgo the benefits of object-oriented abstraction." The workaround is **composition over inheritance** (Item 18): give `ColorPoint` a private `Point` field and a public `asPoint()` view method, instead of extending `Point`. (Adding a value component to a subclass of an *abstract* class with no value components of its own — e.g. `Shape → Circle`/`Rectangle` — is fine, since the abstract superclass can never be instantiated directly.)

The recipe for a correct `equals`:
1. Use `==` to check if the argument is a reference to this object — a performance shortcut.
2. Use `instanceof` to check the argument has the correct type (not `getClass()`, unless the class implements an interface that itself refines the equals contract across implementing classes — e.g. `Set`, `List`, `Map`, `Map.Entry`).
3. Cast the argument to the correct type (guaranteed safe after step 2's `instanceof`).
4. For each significant field, check that it matches: `==` for primitives (except `float`/`double`), `equals` recursively for object references, `Float.compare`/`Double.compare` for float/double (because of `NaN` and `-0.0f`), `Objects.equals` for fields that may legitimately be `null`, `Arrays.equals` for arrays.

The `instanceof` operator returns `false` if its first operand is `null`, "regardless of what type appears in the second operand" — so the explicit `if (o == null) return false` guard is redundant once you use `instanceof`.

Full recipe applied:

```java
// Class with a typical equals method
public final class PhoneNumber {
    private final short areaCode, prefix, lineNum;
    ...
    @Override public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof PhoneNumber))
            return false;
        PhoneNumber pn = (PhoneNumber)o;
        return pn.lineNum == lineNum && pn.prefix == prefix
                && pn.areaCode == areaCode;
    }
}
```

Final caveats: **always override `hashCode` when you override `equals`** (Item 11); don't be too clever (don't chase aliasing — `File` deliberately doesn't equate symbolic links to the same target); never substitute another type for `Object` in the signature — `public boolean equals(MyClass o)` *overloads*, not overrides, `Object.equals`, and `@Override` will catch this mistake by failing to compile. AutoValue (or IDE generation) is an acceptable, often preferable, alternative to writing this by hand.

### Panel-ready answer checklist
- States all five contract properties by name (reflexive, symmetric, transitive, consistent, non-null) — not just "override it properly."
- Can construct the `Point`/`ColorPoint` counterexample from memory to prove why inheritance + value component breaks symmetry or transitivity.
- Explains why `getClass()` "fixes" symmetry/transitivity but violates Liskov substitution (cites the `CounterPoint`/`unitCircle` example or an equivalent).
- Knows the composition/view-method workaround (Item 18) as the actual fix, and the abstract-superclass exception.
- Cites at least one real JDK casualty (`Timestamp` symmetry, or `URL` consistency via network-dependent hashing).
- States the four-step recipe (`==` shortcut → `instanceof` → cast → compare significant fields) and the primitive-type-specific comparison rules (`Float.compare`/`Double.compare`, `Objects.equals` for nullable fields, `Arrays.equals`).
- Flags the "overload not override" trap (`equals(MyClass o)`) and names `@Override` as the safeguard.

---

## Item 11: Always override hashCode when you override equals

### Opening scenario
"You put a custom `PhoneNumber` object into a `HashMap` as a key with `put`, then immediately call `get` with a `new PhoneNumber` built from the exact same digits — logically equal by your own `equals` method. It returns `null`. Your `equals` is airtight. What's going on?"

### Follow-up probes
- Which specific clause of the hashCode contract does this violate — say it precisely, not "hashCode is wrong."
- Why does the fact that the two instances happen to hash into the *same bucket* still not save you? What does `HashMap` cache per entry, and why does that make it worse?
- Give me a legal `hashCode` implementation that always returns `42`. Why is it legal, and why is it "atrocious"?
- What's the actual arithmetic recipe for combining multiple fields into one hash, and why the multiplier `31` specifically, and why must it be odd?
- If I cache the hashCode on an immutable object for performance, what could go wrong with the lazy-init version if the class isn't already known to be immutable? What's the danger with concurrent access?
- Should I ever exclude a field from `hashCode` that's used in `equals`? What if I want to exclude a field to speed things up?
- Why does the book warn against specifying the exact algorithm of `hashCode` in your public documentation, even though `String` and `Integer` do exactly that?

### Naive attempt
A mediocre candidate says "just override equals, hashCode isn't that important" or writes:

```java
@Override public int hashCode() { return 42; }
```

reasoning "it's legal, equal objects get the same hash code, done." Or they skip overriding `hashCode` entirely because "equals passes my tests."

### What breaks
Skipping `hashCode` entirely, using `PhoneNumber` from Item 10:

```java
Map<PhoneNumber, String> m = new HashMap<>();
m.put(new PhoneNumber(707, 867, 5309), "Jenny");
```

`m.get(new PhoneNumber(707, 867, 5309))` returns `null` instead of `"Jenny"`. Two `PhoneNumber` instances are logically equal per `equals`, but `Object.hashCode` returns "two seemingly random numbers instead of two equal numbers as required by the contract" — so `get` looks in the wrong hash bucket entirely. Even in the rare case both land in the same bucket, `HashMap` "caches the hash code associated with each entry and doesn't bother checking for object equality if the hash codes don't match" — so it still returns `null`.

The `hashCode() { return 42; }` implementation is legal (equal objects do get equal hash codes — trivially, since *all* objects do) but "ensures that every object hashes to the same bucket, and hash tables degenerate to linked lists. Programs that should run in linear time instead run in quadratic time." For large hash tables this is "the difference between working and not working" — a correctness-adjacent contract technically satisfied while performance collapses.

Excluding significant fields from `hashCode` to save time: "its poor quality may degrade hash tables' performance to the point where they become unusable" — cited historically in pre-Java-2 `String.hashCode`, which sampled only 16 evenly-spaced characters and pathologically collided on large sets of hierarchical names like URLs.

### The recommended approach & why
The hashCode contract, verbatim (adapted from the `Object` specification):

- When `hashCode` is invoked on an object repeatedly during an execution of an application, it must consistently return the same value, provided no information used in `equals` comparisons is modified. This value need not remain consistent from one execution of an application to another.
- If two objects are equal according to `equals(Object)`, then calling `hashCode` on the two objects must produce the same integer result.
- If two objects are unequal according to `equals(Object)`, it is *not required* that `hashCode` produce distinct results — but distinct results for unequal objects may improve hash table performance.

The key provision that silently breaks when you skip `hashCode`: **the second one** — equal objects must have equal hash codes.

The recipe:
1. Declare `int result` and initialize it to the hash code `c` of the first significant field (computed per step 2a).
2. For every remaining significant field `f`:
   - a. Compute `int c`:
     - primitive type → `Type.hashCode(f)` (boxed primitive class);
     - object reference, compared via recursive `equals` → recursively invoke `hashCode` on the field (or on a canonical representation if the comparison is nonstandard); `null` → use `0`;
     - array → treat each significant element as a separate field, combine recursively, or use `Arrays.hashCode` if all elements are significant.
   - b. Combine: `result = 31 * result + c;`
3. Return `result`.

`31` is chosen because it's an odd prime — odd matters because if it were even and the multiplication overflowed, information would be lost (multiplication by 2 is a bit shift); prime is traditional though its benefit is less clear-cut. Multiplication itself matters because it makes the result depend on field *order* — omit it and all anagrams of a string would hash identically.

Applied to `PhoneNumber`:

```java
// Typical hashCode method
@Override public int hashCode() {
    int result = Short.hashCode(areaCode);
    result = 31 * result + Short.hashCode(prefix);
    result = 31 * result + Short.hashCode(lineNum);
    return result;
}
```

Shortcut via `Objects.hash`:

```java
// One-line hashCode method - mediocre performance
@Override public int hashCode() {
    return Objects.hash(lineNum, prefix, areaCode);
}
```

— convenient, "comparable" quality, but slower due to array creation for varargs plus autoboxing.

Caching for immutable classes where hashCode computation is expensive — eager at construction if instances are likely used as hash keys, or lazy with a `0`-initialized field otherwise (thread-safety of lazy init needs care, per Item 83):

```java
// hashCode method with lazily initialized cached hash code
private int hashCode; // Automatically initialized to 0

@Override public int hashCode() {
    int result = hashCode;
    if (result == 0) {
        result = Short.hashCode(areaCode);
        result = 31 * result + Short.hashCode(prefix);
        result = 31 * result + Short.hashCode(lineNum);
        hashCode = result;
    }
    return result;
}
```

(The initial cache value must not coincide with the hash code of a commonly created instance.)

You may exclude *derived* fields (computable from other significant fields) from the hash computation, but you must never exclude a field that *is* used in `equals` — doing so directly risks violating the contract's second provision.

Don't over-specify: unlike `String`/`Integer`, which document their exact hash algorithm ("a mistake we're forced to live with" since it locks out future improvement), leave the algorithm undocumented so you retain the freedom to improve it later.

### Panel-ready answer checklist
- States all three hashCode contract clauses, and specifically names the **second** ("equal objects ⇒ equal hash codes") as the one broken by omission.
- Can explain *why* `get` still returns `null` even in the rare case of an accidental bucket collision — cites the cached-hash-code optimization in `HashMap`.
- Can produce the `31 * result + c` recipe from memory, and explain why 31 is odd/prime and why multiplication (not addition) matters (anagram/order argument).
- Distinguishes "legal" from "good" — the `return 42` example, and states the performance consequence in Big-O terms (linear degrading to quadratic).
- Knows when hashCode caching is appropriate (immutable, expensive to compute) and the thread-safety caveat on lazy initialization.
- States you must never exclude an equals-significant field from hashCode, even for speed, and cites the pre-Java-2 `String.hashCode` collision story as evidence.
- Recommends AutoValue/IDE generation as an acceptable alternative to hand-writing, same as for `equals`.

---

## Item 12: Always override toString

### Opening scenario
"A teammate pastes a bug report: `Assertion failure: expected {abc, 123}, but was {abc, 123}` — the expected and actual values print identically, yet the assertion failed. The test framework isn't lying. What's actually going on, and whose fault is it?"

### Follow-up probes
- Why is this not just a cosmetic nice-to-have — what production-code paths call `toString` for you, without you ever calling it directly?
- Should `toString` for a value class specify its exact output format in the javadoc, or leave it unspecified? Argue both sides.
- If I do specify the format precisely, what other API element should I provide alongside it, and why?
- Is it ever wrong to override `toString`? Name a class type where you deliberately should not.
- If `toString` returns a nicely formatted string, do I still need accessor methods for the individual pieces (e.g. area code)? Why can't callers just parse the string?
- What's the actual contractual wording from `Object`'s spec for what `toString` should return?

### Naive attempt
A mediocre candidate doesn't override `toString` at all, reasoning "it's just for debugging, I'll add it if someone asks." Or they override it but only for their own top-level class, forgetting collections/wrapper objects holding references to it need it too.

### What breaks
Without an override, `Object.toString` returns the class name, `@`, and the unsigned hex hash code — e.g. `PhoneNumber@163b91`. This is technically "concise and easy to read" but "isn't very informative when compared to `707-867-5309`."

Concretely:
- `System.out.println("Failed to connect to " + phoneNumber);` produces a useless diagnostic (`Failed to connect to PhoneNumber@163b91`) instead of `Failed to connect to 707-867-5309`.
- Printing a map of `PhoneNumber` values shows `{Jenny=PhoneNumber@163b91}` instead of `{Jenny=707-867-5309}` — the problem propagates to *any* container holding a reference to your object, not just direct prints.
- Test failure reports become indistinguishable: `Assertion failure: expected {abc, 123}, but was {abc, 123}` — the two objects differ, but their (default) string representations collapse to the same-looking output, because critical state isn't included in the string.
- `toString` fires automatically on `println`, `printf`, string concatenation, `assert`, and debugger display — "even if you never call `toString` on an object, others may," e.g. a component logging an error message that embeds your object.

### The recommended approach & why
The general contract (from `Object`'s specification): the returned string should be "a concise but informative representation that is easy for a person to read," and "It is recommended that all subclasses override this method."

When practical, `toString` should return **all of the interesting information** the object contains. If the object is large or not conducive to a string form, return a sensible summary (e.g. `Manhattan residential phone directory (1487536 listings)` or `Thread[main,5,main]` — though the `Thread` example itself "flunks" the self-explanatory bar Bloch sets).

Key design decision: **whether to document the format**.
- Specifying it precisely gives you a de facto standard, human-readable, parseable representation — usable for both display and persistence (e.g. CSV). If you do this, pair it with a matching static factory or constructor so round-tripping is easy (the pattern `BigInteger`, `BigDecimal`, and boxed primitives all follow).
- Leaving it unspecified keeps you free to improve the format later without breaking client code that parses it — but you must still document your intent explicitly, one way or the other.

Specified-format example:

```java
/**
 * Returns the string representation of this phone number.
 * The string consists of twelve characters whose format is
 * "XXX-YYY-ZZZZ", where XXX is the area code, YYY is the
 * prefix, and ZZZZ is the line number. Each of the capital
 * letters represents a single decimal digit.
 *
 * If any of the three parts of this phone number is too small
 * to fill up its field, the field is padded with leading zeros.
 * For example, if the value of the line number is 123, the last
 * four characters of the string representation will be "0123".
 */
@Override public String toString() {
    return String.format("%03d-%03d-%04d", areaCode, prefix, lineNum);
}
```

Unspecified-format example:

```java
/**
 * Returns a brief description of this potion. The exact details
 * of the representation are unspecified and subject to change,
 * but the following may be regarded as typical:
 *
 * "[Potion #9: type=love, smell=turpentine, look=india ink]"
 */
@Override public String toString() { ... }
```

Regardless of whether you specify the format, **provide programmatic accessors** for the information the string contains (e.g. area code, prefix, line number accessors on `PhoneNumber`). Failing to do so forces callers to parse your string, which is error-prone, hurts performance, and — even with an "unspecified, subject to change" disclaimer — turns the format into a de facto API anyway.

Exceptions: don't write `toString` for a static utility class (no instances to describe); don't write it for most enum types (Java already provides a good one). Do write it on an abstract class whose subclasses share a common string representation (as most collection framework abstract classes do).

AutoValue/IDEs will generate a `toString` for you — fine for a "dump all fields" style representation (e.g. `Potion`), but inappropriate when a class has a specialized, standard external representation (e.g. `PhoneNumber`'s `XXX-YYY-ZZZZ`). Either way, a generated `toString` beats the inherited `Object` one.

### Panel-ready answer checklist
- Quotes the contract phrase: "a concise but informative representation that is easy for a person to read."
- Names concrete call sites that invoke `toString` implicitly: `println`, `printf`, string concatenation, `assert`, debuggers, and logging by *other* components holding a reference to your object.
- Explains the "specify vs. leave unspecified" tradeoff for value classes, and the accompanying factory/constructor recommendation when you do specify it.
- States that accessors must be provided regardless of the format decision, and explains why ("de facto API" trap).
- Names the classes that should *not* get a custom `toString` (static utility classes, most enums) and the ones that *should* despite being abstract (shared-representation abstract classes).
- Can explain the "expected {abc, 123}, but was {abc, 123}" scenario as a symptom of the object's string form omitting distinguishing state.

---

## Item 13: Override clone judiciously

### Opening scenario
"You call `.clone()` on your custom `Stack` class. It doesn't throw, returns an object, `size` looks right — but pushing onto the original corrupts the clone, or vice versa, with no exception, just silently wrong data later. What did `clone()` actually copy, and why is that not enough?"

### Follow-up probes
- What does `Cloneable` actually specify as an interface — list its methods. Given that answer, what does implementing it even do?
- Why is `Object.clone()` protected, and why does that make `Cloneable` "fail to serve its purpose" on its own?
- If my class has only primitive fields or references to immutable objects, is `super.clone()` alone sufficient? When would even that be wrong?
- What does the contract say about `x.clone().getClass() == x.getClass()` — is that a real requirement or just typical behavior? What breaks it?
- Why is `clone()`ing a linked-list-based hash table bucket-by-bucket via a shallow `buckets.clone()` still broken? What's the actual fix, and what's wrong with the naive recursive version of that fix?
- If I'm designing a class for inheritance, should I implement `Cloneable` at all? What are the two legitimate design choices?
- What's the standard alternative to `Cloneable`/`clone` that the book recommends over it, and what specific advantages does it have?

### Naive attempt
A mediocre candidate says "implement `Cloneable`, override `clone()` to call `super.clone()`, done" — treating `clone()` like a copy constructor without checking what fields the object actually holds.

```java
// Naive - works only if there's no mutable state
@Override public Stack clone() {
    try {
        return (Stack) super.clone();
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();
    }
}
```

applied blindly even when the class has a mutable array/field.

### What breaks
`Cloneable` "contains no methods" — it just flips a flag that changes what `Object`'s *protected* `clone()` does internally: implementing it makes `Object.clone()` return a field-by-field copy instead of throwing `CloneNotSupportedException`. This is "a highly atypical use of interfaces... it modifies the behavior of a protected method on a superclass" rather than declaring a capability.

For the `Stack` class (holding `Object[] elements` and `int size`):

```java
public class Stack {
    private Object[] elements;
    private int size = 0;
    ...
}
```

If `clone()` merely returns `super.clone()`, the resulting `Stack` has the correct `size`, but its `elements` field **references the same array as the original**. "Modifying the original will destroy the invariants in the clone and vice versa. You will quickly find that your program produces nonsensical results or throws a `NullPointerException`." The fix is a per-field deep-enough copy:

```java
// Clone method for class with references to mutable state
@Override public Stack clone() {
    try {
        Stack result = (Stack) super.clone();
        result.elements = elements.clone();
        return result;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();
    }
}
```

But even array-level `.clone()` isn't always deep enough. For a hash table with `Entry[] buckets`, each bucket heading a singly linked list:

```java
// Broken clone method - results in shared mutable state!
@Override public HashTable clone() {
    try {
        HashTable result = (HashTable) super.clone();
        result.buckets = buckets.clone();
        return result;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();
    }
}
```

"Though the clone has its own bucket array, this array references the same linked lists as the original" — corrupting both clone and original nondeterministically. The recursive `Entry.deepCopy()` fix works but "consumes one stack frame for each element in the list. If the list is long, this could easily cause a stack overflow" — so the recommended version replaces recursion with iteration.

Structural issues that compound this: `final` fields referring to mutable objects are largely incompatible with `Cloneable` — "clone would be prohibited from assigning a new value to the field" — so you may be forced to strip `final` from fields purely to support cloning. A `clone` method also must never invoke an overridable method on the clone under construction (same rule as constructors, Item 19), or a subclass override can run before the subclass has fixed up its own state in the clone.

### The recommended approach & why
The `clone` contract, verbatim (from `Object`'s specification): "Creates and returns a copy of this object." For any object `x`:

- `x.clone() != x` will be true,
- `x.clone().getClass() == x.getClass()` will be true, "but these are not absolute requirements,"
- `x.clone().equals(x)` will typically be true, "but this is not an absolute requirement."

By convention, the returned object should be obtained by calling `super.clone` — if every class in the hierarchy (except `Object`) obeys this, `x.clone().getClass() == x.getClass()` holds automatically. By convention, the returned object should be **independent** of the original — which is exactly what forces field-level fixing of mutable state.

For a class whose fields are all primitives or immutable object references, `super.clone()` alone suffices:

```java
// Clone method for class with no references to mutable state
@Override public PhoneNumber clone() {
    try {
        return (PhoneNumber) super.clone();
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();  // Can't happen
    }
}
```

(Covariant return type — `PhoneNumber` instead of `Object` — is legal and desirable, eliminating client-side casts. Note: **immutable classes should never provide a `clone` method** at all — it only "encourages wasteful copying.")

For mutable internal state, recursively fix up fields (`elements.clone()` for `Stack`), and for structures with their own internal linked structures, deep-copy them explicitly (iteratively, not recursively, to avoid stack overflow on long chains):

```java
// Iteratively copy the linked list headed by this Entry
Entry deepCopy() {
    Entry result = new Entry(key, value, next);
    for (Entry p = result; p.next != null; p = p.next)
        p.next = new Entry(p.next.key, p.next.value, p.next.next);
    return result;
}
```

An alternative for complex mutable state: call `super.clone()`, reset fields to initial state, then use the object's own higher-level methods (e.g. repeated `put`) to rebuild state — simpler and more robust, but "antithetical to the whole Cloneable architecture" and generally slower.

Public `clone` methods should **omit the `throws CloneNotSupportedException` clause** even though `Object.clone` declares it (checked exceptions are harder to use, Item 71). If designing a class for inheritance, don't implement `Cloneable`: either mimic `Object` with a protected, properly-functioning `clone` that still throws `CloneNotSupportedException` (leaving the choice to subclasses), or block cloning entirely:

```java
// clone method for extendable class not supporting Cloneable
@Override
protected final Object clone() throws CloneNotSupportedException {
    throw new CloneNotSupportedException();
}
```

Thread-safe classes implementing `Cloneable` need a properly synchronized `clone`, same as any other method (Item 78) — `Object.clone` itself is not synchronized.

**The actual recommendation**: prefer a **copy constructor** or **copy factory** over `Cloneable` entirely:

```java
// Copy constructor
public Yum(Yum yum) { ... };

// Copy factory
public static Yum newInstance(Yum yum) { ... };
```

These "don't rely on a risk-prone extralinguistic object creation mechanism; they don't demand unenforceable adherence to thinly documented conventions; they don't conflict with the proper use of `final` fields; they don't throw unnecessary checked exceptions; and they don't require casts." A copy constructor/factory can also take an *interface* type as its argument — a **conversion constructor/factory** — letting a client copy a `HashSet` into a `TreeSet` via `new TreeSet<>(s)`, something `clone()` can never offer. "New interfaces should not extend [`Cloneable`], and new extendable classes should not implement it." The one genuine exception: arrays, "the sole compelling use of the clone facility."

### Panel-ready answer checklist
- States that `Cloneable` has zero methods and explains what it actually does — flips the behavior of `Object`'s protected `clone`, rather than declaring a capability like a normal interface.
- Quotes the three contract expectations (`!=`, same `getClass()`, typically `equals`) and correctly labels which are "not absolute requirements."
- Can reproduce the shallow-copy bug on a mutable-field class (`Stack.elements` or the `HashTable.buckets`/linked-list case) and explain why array-level `.clone()` alone isn't enough for nested mutable structures.
- Knows iterative vs. recursive `deepCopy` and why recursion risks stack overflow on long chains.
- States the `final` mutable-field incompatibility and the overridable-method-during-clone hazard (same rule as constructors).
- Recommends copy constructor/copy factory as the default alternative, and can name their concrete advantages over `Cloneable`, including conversion constructors (`new TreeSet<>(s)`).
- Knows arrays are the legitimate remaining use case for `clone()`.

---

## Item 14: Consider implementing Comparable

### Opening scenario
"You add a `BigDecimal("1.00")` and a `BigDecimal("1.0")` to a `HashSet` — you get two elements. You do the exact same thing with a `TreeSet` — you get one element. Same two objects, same two `add` calls, different outcome. What's different about how each collection decides 'already present'?"

### Follow-up probes
- What exactly does implementing `Comparable` buy you that plain `equals` doesn't — name concrete APIs/collections that depend on it.
- Since `compareTo` isn't declared in `Object`, does it need a null-check and a type-check like `equals` does? Why or why not?
- State the sign-consistency requirement of the contract precisely — what does `sgn(x.compareTo(y)) == -sgn(y.compareTo(x))` actually forbid?
- Is `compareTo` required to be consistent with `equals`? What happens in practice if it isn't — walk through the `BigDecimal` example again with that framing.
- If I extend a `Comparable` class and add a value component, does the same trap from Item 10 apply? What's the fix?
- Why is `(a.hashCode() - b.hashCode())` a broken way to write a comparator, even though it "basically works" in casual testing?
- Walk me through building a multi-field `compareTo` two ways: primitive comparisons by hand, and the Java 8 `Comparator` construction methods. What's the actual tradeoff between them?

### Naive attempt
A mediocre candidate writes a difference-based comparator because it "looks clean":

```java
// BROKEN difference-based comparator - violates transitivity!
static Comparator<Object> hashCodeOrder = new Comparator<>() {
    public int compare(Object o1, Object o2) {
        return o1.hashCode() - o2.hashCode();
    }
};
```

or writes multi-field comparisons using `<` and `>` relational operators directly on primitive fields, treating `compareTo` like a simple ternary combination without thinking about overflow or transitivity.

### What breaks
The difference-based comparator is "fraught with danger of integer overflow and IEEE 754 floating point arithmetic artifacts" — two large hash codes of opposite sign can overflow `int` subtraction and produce a result with the wrong sign, silently breaking the ordering's transitivity without ever throwing.

The `BigDecimal("1.0")` vs `BigDecimal("1.00")` case: these two instances are **unequal under `equals`** (different scale) but **equal under `compareTo`** (same numeric value) — `compareTo` is **inconsistent with equals**. A `HashSet` uses `equals` for containment, so it keeps both (2 elements); a `TreeSet` uses the ordering imposed by `compareTo` in place of `equals`, so it treats them as duplicates (1 element). "It is not a catastrophe if this happens, but it's something to be aware of" — sorted collections built on such a class "may not obey the general contract appropriate to the collection interfaces," since those interfaces are defined in terms of `equals`.

The Item 10 value-component trap recurs identically for `Comparable`: "there [is] no way to extend an instantiable class and add a value component while preserving the `compareTo` contract, unless you are willing to forgo the benefits of object-oriented abstraction" — same composition-based workaround applies (don't extend; contain-and-view).

### The recommended approach & why
The `compareTo` contract (general shape, using `sgn` for the mathematical signum function):

- The implementor must ensure `sgn(x.compareTo(y)) == -sgn(y.compareTo(x))` for all `x`, `y` (which implies `x.compareTo(y)` must throw an exception if and only if `y.compareTo(x)` throws one).
- The implementor must ensure the relation is transitive: `(x.compareTo(y) > 0 && y.compareTo(z) > 0)` implies `x.compareTo(z) > 0`.
- `x.compareTo(y) == 0` implies `sgn(x.compareTo(z)) == sgn(y.compareTo(z))`, for all `z`.
- It is **strongly recommended, but not required**, that `(x.compareTo(y) == 0) == (x.equals(y))`. A class violating this should say so explicitly: "Note: this class has a natural ordering that is inconsistent with equals."

Key structural differences from `equals`: `Comparable` is generic and statically typed (`Comparable<T>`), so `compareTo` needs **no runtime type check or cast** — a wrong-type argument fails to compile, not at runtime. It also does not need to work across unrelated types the way `equals` must handle `null`/other-type gracefully — `compareTo` is *permitted* to throw `ClassCastException` on a type mismatch and typically does. A `null` argument should throw `NullPointerException`, "and it will, as soon as the method attempts to access its members" — no explicit guard required.

Single-field example, delegating to a `Comparator`:

```java
// Single-field Comparable with object reference field
public final class CaseInsensitiveString
        implements Comparable<CaseInsensitiveString> {
    public int compareTo(CaseInsensitiveString cis) {
        return String.CASE_INSENSITIVE_ORDER.compare(s, cis.s);
    }
    ...
}
```

Multi-field, comparing most-significant field first, using the boxed-primitive static `compare` methods (not `<`/`>`, which the book calls "verbose and error-prone" and "no longer recommended" since Java 7 added `compare` to all boxed primitives):

```java
// Multiple-field Comparable with primitive fields
public int compareTo(PhoneNumber pn) {
    int result = Short.compare(areaCode, pn.areaCode);
    if (result == 0) {
        result = Short.compare(prefix, pn.prefix);
        if (result == 0)
            result = Short.compare(lineNum, pn.lineNum);
    }
    return result;
}
```

Java 8 comparator-construction-method style — more concise, at "a modest performance cost" (Bloch measured ~10% slower sorting):

```java
// Comparable with comparator construction methods
private static final Comparator<PhoneNumber> COMPARATOR =
        comparingInt((PhoneNumber pn) -> pn.areaCode)
                .thenComparingInt(pn -> pn.prefix)
                .thenComparingInt(pn -> pn.lineNum);

public int compareTo(PhoneNumber pn) {
    return COMPARATOR.compare(this, pn);
}
```

`comparingInt` builds a `Comparator` from an `int`-typed key extractor; `thenComparingInt` chains a tie-breaking key, stackable arbitrarily deep for lexicographic ordering. Analogues exist for `long` and `double` (and their narrower/wider relatives), plus `comparing`/`thenComparing` overloads for object reference keys (natural order, or an explicit `Comparator` on the extracted key).

The correct fix for the broken hash-code comparator — use the static `compare` method or a construction method instead of subtraction:

```java
// Comparator based on static compare method
static Comparator<Object> hashCodeOrder = new Comparator<>() {
    public int compare(Object o1, Object o2) {
        return Integer.compare(o1.hashCode(), o2.hashCode());
    }
};

// Comparator based on Comparator construction method
static Comparator<Object> hashCodeOrder =
        Comparator.comparingInt(o -> o.hashCode());
```

### Panel-ready answer checklist
- States all three formal contract clauses (anti-symmetry via `sgn`, transitivity, and the `compareTo(y) == 0` sign-consistency clause) plus the "strongly recommended, not required" consistency-with-equals clause, verbatim in spirit.
- Explains why `compareTo` doesn't need `equals`'s null-check/instanceof dance — generics give compile-time type safety, and `null` naturally throws `NullPointerException` on first field access.
- Can walk through the `BigDecimal("1.0")`/`BigDecimal("1.00")` HashSet-vs-TreeSet divergence as the concrete consequence of "inconsistent with equals," and states which collection interfaces' contracts are technically at risk (`Set`, `Map` — defined in terms of `equals`, not `compareTo`).
- Identifies the value-component-via-inheritance trap as identical to Item 10's, and the same composition-based fix.
- Explains precisely why hash-code subtraction is broken (`int` overflow / float artifacts), not just "it's bad practice."
- Can write both the hand-rolled multi-field `compareTo` (boxed-primitive `compare`, most-significant-field-first) and the Java 8 `Comparator.comparingInt().thenComparingInt()` chain, and states the real tradeoff (conciseness vs. ~10% performance cost).
