# Chapter 5: Generics

## Item 26: Don't use raw types

### Opening scenario
"Here's a field: `private final Collection stamps = ...;` documented with a comment `// Contains only Stamp instances`. Someone does `stamps.add(new Coin(...))`. It compiles, it runs, no error. Where does this blow up, and why didn't the compiler catch it?"

### Follow-up probes
- What's the actual difference between the raw type `List` and the parameterized type `List<Object>`? Can you pass a `List<String>` to each?
- Is `Set<?>` just a fancier spelling of the raw type `Set`? Why does `c.add("verboten")` fail to compile on a `Set<?>` but succeed (with a warning) on raw `Set`?
- Why does the language allow raw types at all, if they're unsafe?
- Is there any legitimate use of a raw type in modern code?

### Naive attempt
Declare `private final Collection stamps = ...;` (raw) reasoning "I don't want to commit to a type parameter here."

### What breaks
The erroneous insertion `stamps.add(new Coin(...))` compiles with only a vague `unchecked call` warning. Failure surfaces later and elsewhere: `for (Iterator i = stamps.iterator(); ...) Stamp stamp = (Stamp) i.next();` throws `ClassCastException` at the cast — far from the code that actually inserted the `Coin`. With the parameterized field `Collection<Stamp> stamps`, the same insertion is a compile-time error naming the exact incompatible types.

### The recommended approach & why
Never use raw types in new code. If you want a collection that legitimately holds arbitrary objects, use the parameterized type `List<Object>`, not the raw type `List` — `List<String>` is a subtype of raw `List` but *not* of `List<Object>`, so passing a `List<String>` into something declared for the raw type silently defeats the type system, while `List<Object>` params still reject it correctly. If the element type genuinely doesn't matter and you don't need to insert into the collection, use an unbounded wildcard `Set<?>` — it lets the compiler forbid inserting anything but `null` into it, closing the hole raw types leave open. The one legitimate use of a raw type is with `instanceof`: `if (o instanceof Set)` then cast to the wildcard type `Set<?>`, never to raw `Set`. Raw types exist purely for migration compatibility with pre-generics code and are implemented via erasure for that reason (Item 28).

### Panel-ready answer checklist
- States plainly: raw types compile-time-unsafe, `List<Object>` and `Set<?>` are both safe.
- Explains why `List<String>` is a subtype of raw `List` but not of `List<Object>` (subtyping rules for generics, Item 28).
- Distinguishes "opted out of generics" (raw type) from "explicitly holds any type" (`List<Object>`).
- Names the one legal raw-type use: `instanceof` checks, followed by a wildcard-typed cast.
- Names migration compatibility as the historical reason raw types are legal at all.

---

## Item 27: Eliminate unchecked warnings

### Opening scenario
"Your CI log has 40 `[unchecked]` warnings scattered across a generics-heavy module. Someone proposes slapping `@SuppressWarnings("unchecked")` on the class to make CI green. What's your objection?"

### Follow-up probes
- What's the difference between a warning you can eliminate and one you merely suppress?
- Where exactly should `@SuppressWarnings` go — method, statement, variable?
- Can you put it on a `return` statement? Why not?
- What must be true before you're allowed to suppress a warning at all?
- Why does the diamond operator (`<>`) matter here?

### Naive attempt
```java
@SuppressWarnings("unchecked")
public class Venery { ... }
```
or scattering `@SuppressWarnings` across whole multi-line methods to silence the compiler quickly.

### What breaks
Class- or method-wide suppression doesn't just hide the warnings you understood — it hides every future unchecked warning introduced anywhere in that scope, including genuine bugs. A real example from `ArrayList.toArray`: the cast `(T[]) Arrays.copyOf(elements, size, a.getClass())` produces `warning: [unchecked] unchecked cast`. Putting `@SuppressWarnings` on the whole method (or attempting it on the `return` statement, which is illegal — it isn't a declaration) either fails to compile or masks the scope far beyond the one line that's actually justified.

### The recommended approach & why
Eliminate every unchecked warning you can — e.g. `Set<Lark> exaltation = new HashSet();` just needs the diamond: `new HashSet<>()`. When a warning is truly unavoidable but you can *prove* the code is typesafe, suppress it with `@SuppressWarnings("unchecked")` in the narrowest possible scope — typically a single local variable declaration, not a method or class. The `ArrayList.toArray` fix: introduce a local `result` variable just to hold the annotation:
```java
@SuppressWarnings("unchecked") T[] result =
    (T[]) Arrays.copyOf(elements, size, a.getClass());
return result;
```
Always attach a comment explaining *why* the cast is safe — if you can't write that comment convincingly, you've probably found a real bug, not a false positive. Suppressing without proof just gives you false confidence: the code will compile clean but can still throw `ClassCastException` at runtime, and a genuinely new/dangerous warning will get lost among the ones you blanket-silenced.

### Panel-ready answer checklist
- Says "eliminate first, suppress only when proven safe" — not "suppress to satisfy CI."
- States the narrowest-scope rule and why (masking future genuine warnings).
- Knows `@SuppressWarnings` can't go on a `return` statement (not a declaration) and gives the local-variable workaround.
- Requires a justifying comment alongside every suppression.
- Connects "leftover unchecked warning" directly to "potential `ClassCastException` at runtime."

---

## Item 28: Prefer lists to arrays

### Opening scenario
"You write `new List<E>[]`, or `new List<String>[]`, or even `new E[]` inside a generic class, and the compiler flatly refuses to compile it — 'generic array creation.' An array of `Object` compiles fine. What's fundamentally different about arrays vs. generics that causes this?"

### Follow-up probes
- What does covariant mean for arrays, and what does invariant mean for generics — give an example of each failing.
- What does "reified" mean, and how does it relate to erasure?
- Why is `Object[] objectArray = new Long[1]; objectArray[0] = "oops";` legal to compile but blows up at runtime?
- If generic array creation is illegal, how does `ArrayList<E>` work internally — doesn't it need an array?
- What's the practical fix when you hit this error in your own generic type?

### Naive attempt
Force it through with an unchecked cast: `choiceArray = (T[]) choices.toArray();` inside a generic `Chooser<T>` class, or declare `private final T[] choiceArray;` directly.

### What breaks
`new List<E>[]`, `new List<String>[]`, `new E[]` are flatly illegal — generic array creation errors — because if allowed, compiler-generated casts elsewhere in an otherwise-correct program could fail with `ClassCastException`, violating the generic type system's fundamental guarantee. Casting around it, `(T[]) choices.toArray()`, compiles but only with an unchecked-cast warning: the compiler can't verify it because `T`'s runtime identity is erased. Contrast with arrays: `Object[] objectArray = new Long[1]; objectArray[0] = "I don't fit in";` compiles cleanly (arrays are covariant — `Long[]` is a subtype of `Object[]`) but throws `ArrayStoreException` at runtime, because arrays are reified and enforce their element type at runtime. Generics are the opposite: invariant (`List<Object>` is neither sub- nor supertype of `List<String>`) and erased (no runtime element-type enforcement), so `ol.add("...")` on a mistyped `List<Object> ol = new ArrayList<Long>()` doesn't even compile — you find the error earlier, at compile time.

### The recommended approach & why
When array-vs-generics friction shows up (compile errors or unchecked warnings from mixing them), the fix is almost always to replace the array with a `List`. Rewriting `Chooser<T>` to back itself with `List<T> choiceList = new ArrayList<>(choices);` instead of `T[] choiceArray` compiles with zero errors and zero warnings — slightly more verbose, possibly marginally slower, but it buys a guarantee against `ClassCastException` at runtime. The underlying mechanism: arrays are covariant + reified (runtime safety, no compile-time safety); generics are invariant + erased (compile-time safety, no runtime enforcement). They encode type safety at opposite points in the program's lifecycle, which is exactly why mixing them produces friction.

### Panel-ready answer checklist
- Defines covariant (arrays) vs invariant (generics) with a concrete failing example for each.
- Defines reified (arrays) vs erased (generics) and ties erasure back to migration compatibility (Item 26).
- Explains precisely why `new E[]` / `new List<E>[]` are illegal (would let compiler-generated casts fail silently).
- Gives the standard remedy: swap the internal array for a `List`.
- Notes this is a general rule of thumb, not absolute — e.g. `ArrayList`/`HashMap` are still array-backed internally for performance, which Item 29 revisits.

---

## Item 29: Favor generic types

### Opening scenario
"Here's a toy `Stack` backed by `Object[]`, with `push(Object e)` / `Object pop()`. Clients have to cast every `pop()` result and that cast can fail at runtime. Generify this class for me — walk me through it."

### Follow-up probes
- What's the first compile error you hit the moment you swap `Object[]` for `E[]`?
- Give me the two ways to work around it, and the tradeoffs between them.
- Which one causes heap pollution, and why is it "harmless" in this specific case?
- Can I create a `Stack<int>`? Why not?
- How would you restrict a generic type's parameter to only certain types, e.g. `DelayQueue`?

### Naive attempt
```java
public class Stack<E> {
  private E[] elements;
  ...
  public Stack() { elements = new E[DEFAULT_INITIAL_CAPACITY]; }
  ...
}
```

### What breaks
`elements = new E[DEFAULT_INITIAL_CAPACITY];` is a generic array creation error — flatly won't compile, per Item 28's rule that you can't create an array of a non-reifiable type.

### The recommended approach & why
Two fixes, both requiring a proof-then-suppress step (Item 27):
1. **Cast at creation** (preferred, more common): `elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];` — legal but produces an unchecked-cast warning. Proof of safety: `elements` is private, never returned or passed elsewhere, and only ever populated via `push(E e)`, so every element really is an `E`. Suppress with `@SuppressWarnings("unchecked")` on the constructor (the narrowest scope containing the cast). This is more readable — the field stays declared as `E[]` — but it does cause heap pollution: the array's runtime type is `Object[]`, not `E[]`, even though its compile-time type says otherwise. Harmless here because the array is never exposed outside the class.
2. **Cast at each read**: keep the field as `Object[] elements`, and cast on `pop()`: `@SuppressWarnings("unchecked") E result = (E) elements[--size];` — no heap pollution, but requires a separate cast (and justification comment) at every read site instead of once.
Bounded type parameters extend this: `class DelayQueue<E extends Delayed> implements BlockingQueue<E>` restricts `E` to subtypes of `Delayed`, letting the implementation call `Delayed` methods on elements without casting. Note primitives can't be type arguments (`Stack<int>` is a compile error) — use boxed types instead (Item 61).

### Panel-ready answer checklist
- Identifies the exact first compiler error: generic array creation on `new E[...]`.
- States both fixes (array-cast-at-creation vs. Object-array-with-cast-at-read) and their tradeoffs (readability/conciseness vs. avoiding heap pollution).
- Can justify *why* the unchecked cast in `Stack` is actually safe (private field, never escapes, only populated through `push(E)`).
- Defines heap pollution and confirms it's harmless specifically because the array never leaves the class.
- Mentions bounded type parameters (`<E extends Delayed>`) as the mechanism for restricting permissible type arguments.

---

## Item 30: Favor generic methods

### Opening scenario
"Here's a static utility: `public static Set union(Set s1, Set s2) { Set result = new HashSet(s1); result.addAll(s2); return result; }`. It compiles with two unchecked warnings. Fix the signature, not the body."

### Follow-up probes
- Where exactly does the type parameter list go in the declaration?
- What's a "generic singleton factory," and how does erasure make it possible to use one object for every `T`?
- What does `<E extends Comparable<E>>` mean and why is it called a recursive type bound?
- Why would you eventually want bounded wildcards here instead (forward-reference to Item 31)?

### Naive attempt
Leave the raw-typed signature as-is and either ignore the two `unchecked` warnings or blanket-suppress them at the method level.

### What breaks
The raw-typed version compiles but with two `unchecked` warnings: `unchecked call to HashSet(Collection<? extends E>) as a member of raw type HashSet` and similarly for `addAll`. It's not memory-unsafe by itself, but it invites a caller to pass in mismatched-but-erased collections and get a `ClassCastException` down the line.

### The recommended approach & why
Add a type parameter list between the modifiers and the return type: `public static <E> Set<E> union(Set<E> s1, Set<E> s2)`, using `E` throughout. This compiles with zero warnings and gives full type safety with no casts at call sites. For stateless, type-independent objects (e.g. an identity function), use the **generic singleton factory** pattern — one physical `UnaryOperator<Object> IDENTITY_FN` instance, dispensed through a generic static factory that performs a single justified unchecked cast:
```java
private static UnaryOperator<Object> IDENTITY_FN = (t) -> t;
@SuppressWarnings("unchecked")
public static <T> UnaryOperator<T> identityFunction() {
  return (UnaryOperator<T>) IDENTITY_FN;
}
```
This works *because* generics are erased — at runtime there's only one class, so one instance safely serves every `T`. Finally, a **recursive type bound** like `<E extends Comparable<E>>` expresses "any type that can be compared to itself" — used to require *mutual comparability* across a whole collection: `public static <E extends Comparable<E>> E max(Collection<E> c)`.

### Panel-ready answer checklist
- Produces the corrected generic signature with the type parameter list correctly placed.
- Explains the generic singleton factory and why erasure specifically is what makes a single object safe for every `T`.
- Correctly reads `<E extends Comparable<E>>` aloud as "any type comparable to itself" / mutual comparability.
- Flags that `union`'s parameters could be made more flexible with wildcards — sets up Item 31 without conflating the two.

---

## Item 31: Use bounded wildcards to increase API flexibility

### Opening scenario
"You wrote a `pushAll(Iterable<E> src)` method on a generic `Stack<E>`. Calling it with an `Iterable<Integer>` on a `Stack<Number>` fails to compile: `incompatible types: Iterable<Integer> cannot be converted to Iterable<Number>`. `Integer` is a `Number` — why doesn't this work, and what's wrong with the method signature?"

### Follow-up probes
- Now write `popAll(Collection<E> dst)` to go with it — does the same problem hit it, and does the fix look the same?
- What if a parameter is both a producer and a consumer at once — what wildcard do you use then?
- Why should return types basically never use wildcards?
- Apply PECS to `Comparable<T>` — what should `max`'s bound actually look like, and why?
- Why do `List<?>` and `List<E>` sometimes both work for the same method signature — when do you pick one over the other?
- Why won't `list.set(i, list.set(j, list.get(i)))` compile on a `List<?>` parameter, and how do you get around it without an unsafe cast or a raw type?

### Naive attempt
```java
// pushAll method without wildcard type - deficient!
public void pushAll(Iterable<E> src) {
    for (E e : src)
        push(e);
}
```
And symmetrically for the consumer side:
```java
// popAll method without wildcard type - deficient!
public void popAll(Collection<E> dst) {
    while (!isEmpty())
        dst.add(pop());
}
```

### What breaks
Both compile cleanly in isolation, but reject perfectly sound client code because `List<Type1>` is neither a subtype nor a supertype of `List<Type2>` — generics are invariant, unlike arrays (Item 28). `Stack<Number> numberStack = ...; Iterable<Integer> integers = ...; numberStack.pushAll(integers);` fails with `incompatible types: Iterable<Integer> cannot be converted to Iterable<Number>`. Symmetrically, `Stack<Number> numberStack = ...; Collection<Object> objects = ...; numberStack.popAll(objects);` fails because `Collection<Object>` is not a subtype of `Collection<Number>`. Neither failure is a real type-safety violation — pushing `Integer`s onto a `Stack<Number>` and draining a `Stack<Number>` into a `Collection<Object>` are both actually sound — the signatures are just too rigid to say so.

### The recommended approach & why
**PECS: producer–extends, consumer–super.** If a parameterized type only *produces* `T` values for the method to consume, bound it with `? extends T`; if it only *consumes* `T` values fed to it by the method, bound it with `? super T`.
- `pushAll`'s `src` produces `E`s for the stack to consume → `public void pushAll(Iterable<? extends E> src)`.
- `popAll`'s `dst` consumes `E`s the stack produces → `public void popAll(Collection<? super E> dst)`.
With both changes, the earlier client calls compile cleanly and safely. If a parameter is *both* producer and consumer, wildcards buy nothing — you need an exact type match.
The same heuristic applies beyond `Stack`: `Chooser`'s constructor only reads from `choices` to produce `T`s, so `Collection<T> choices` becomes `Collection<? extends T> choices`; `union`'s two `Set<E>` parameters are pure producers, so both become `Set<? extends E>` — but the **return type stays `Set<E>`, never a wildcard**, because wildcards in return types just push the complexity onto every caller instead of absorbing it in the API.
Applying PECS *twice* to `max`: the input list produces `T`s → `List<? extends T>`; but `Comparable<T>` itself *consumes* `T` (via `compareTo`) so it should be `Comparable<? super T>` — giving `public static <T extends Comparable<? super T>> T max(List<? extends T> list)`. This is what makes `List<ScheduledFuture<?>>` acceptable even though `ScheduledFuture` doesn't implement `Comparable<ScheduledFuture>` directly (only `Comparable<Delayed>` via `Delayed`). Rule of thumb: **comparables and comparators are always consumers** — prefer `Comparable<? super T>` / `Comparator<? super T>` over `Comparable<T>` / `Comparator<T>`.
There's also a duality between type parameters and wildcards: if a type parameter appears only once in a signature, replace it with a wildcard — `swap(List<E> list, int i, int j)` becomes the simpler public API `swap(List<?> list, int i, int j)`. But the naive body `list.set(i, list.set(j, list.get(i)))` won't compile against `List<?>` — you can't put anything but `null` into a `List<?>`. The fix is **wildcard capture**: delegate to a private generic helper that captures the unknown type as a real type variable:
```java
public static void swap(List<?> list, int i, int j) {
    swapHelper(list, i, j);
}
private static <E> void swapHelper(List<E> list, int i, int j) {
    list.set(i, list.set(j, list.get(i)));
}
```
This keeps the simple wildcard-based public signature while the private helper does the type-safe work internally.

### Panel-ready answer checklist
- States the PECS mnemonic verbatim (producer-extends, consumer-super) and applies it correctly to both `pushAll` (extends) and `popAll` (super) with the reasoning, not just the rule.
- Explicitly ties the failure to generics being invariant (contrast with array covariance, Item 28).
- States the "never use wildcards in return types" rule and why (pushes complexity onto callers).
- Handles the double-PECS case on `Comparable<? super T>` / `Comparator<? super T>` and states "comparables/comparators are always consumers."
- Knows the type-parameter-appears-once → replace-with-wildcard rule, and can produce the wildcard-capture helper-method pattern when the naive wildcard body won't compile.

---

## Item 32: Combine generics and varargs judiciously

### Opening scenario
"This method has no visible casts: `static <T> T[] toArray(T... args) { return args; }`. Used inside a `pickTwo(T a, T b, T c)` helper that returns two of the three arguments via `toArray`, then assigned to a `String[]` from `pickTwo("Good", "Fast", "Cheap")`. It throws `ClassCastException` at runtime. Where's the cast, and why does it fail?"

### Follow-up probes
- Why is it *legal* to declare a generic varargs parameter at all, when creating a generic array explicitly (`new List<E>[]`, Item 28) is flatly illegal?
- What exactly is heap pollution?
- What are the two conditions that make a generic varargs method actually safe?
- What does `@SafeVarargs` do, and why can it only go on `static`, `final`, or `private` methods — never a plain overridable instance method?
- What's the alternative if you can't satisfy those safety conditions?

### Naive attempt
```java
// UNSAFE - Exposes reference to its generic parameter array!
static <T> T[] toArray(T... args) {
    return args;
}
```
combined with something like:
```java
static <T> T[] pickTwo(T a, T b, T c) {
    switch(ThreadLocalRandom.current().nextInt(3)) {
        case 0: return toArray(a, b);
        case 1: return toArray(a, c);
        case 2: return toArray(b, c);
    }
    throw new AssertionError();
}
```

### What breaks
Compiling `pickTwo` generates a varargs array for `toArray` typed `Object[]` — the most specific type the compiler can guarantee for the erased `T`, regardless of what's actually passed at the call site. `toArray` returns that array untouched, so `pickTwo` always physically returns an `Object[]`, even though its declared return type is `T[]`. `String[] attributes = pickTwo("Good", "Fast", "Cheap");` compiles with **no warnings at all** at the call site — but the compiler-generated hidden cast to `String[]` fails at runtime with `ClassCastException`, two stack frames removed from the actual heap-pollution source (`toArray`), and the varargs array was never even modified after the parameters were stored.

### The recommended approach & why
A generic (or parameterized-type) varargs method is safe only if it **(1)** doesn't store anything into the varargs array (which would overwrite the caller's arguments) and **(2)** doesn't let a reference to the array (or a clone) escape to untrusted code. Library methods like `Arrays.asList`, `Collections.addAll`, and `EnumSet.of` satisfy both and are marked `@SafeVarargs`, which suppresses the confusing unchecked warnings for callers without them needing individual `@SuppressWarnings("unchecked")` at every call site. `@SafeVarargs` is only legal on methods that can't be overridden (`static`, `final`, or — since Java 9 — `private` instance methods), because there's no way to guarantee every future override stays safe. A safe example:
```java
@SafeVarargs
static <T> List<T> flatten(List<? extends T>... lists) {
    List<T> result = new ArrayList<>();
    for (List<? extends T> list : lists)
        result.addAll(list);
    return result;
}
```
When you can't satisfy the two safety conditions (as with `toArray`), the alternative is to drop varargs entirely and take a `List<T>` parameter instead, relying on `List.of` (itself `@SafeVarargs`-annotated) at call sites:
```java
static <T> List<T> flatten(List<List<? extends T>> lists) { ... }
static <T> List<T> pickTwo(T a, T b, T c) {
    switch(rnd.nextInt(3)) {
        case 0: return List.of(a, b);
        case 1: return List.of(a, c);
        case 2: return List.of(b, c);
    }
    throw new AssertionError();
}
```
This is fully typesafe because it uses only generics, never arrays — at the cost of slight verbosity and possibly slower call sites.

### Panel-ready answer checklist
- Correctly locates the hidden cast: it's the compiler-inserted cast on the *assignment* (`String[] attributes = pickTwo(...)`), not inside `toArray` or `pickTwo` themselves.
- Defines heap pollution and connects it to a variable of a parameterized type referring to an object not of that type.
- States both safety conditions for `@SafeVarargs` verbatim (don't store into the array; don't let it escape).
- Explains the `static`/`final`/`private`-only restriction on `@SafeVarargs` in terms of overriding.
- Offers the `List<T>` parameter as the alternative when the varargs method can't be made safe.

---

## Item 33: Consider typesafe heterogeneous containers

### Opening scenario
"You need a container that can hold one favorite instance of arbitrarily many *different* types — a `String`, an `Integer`, a `Class` — all in the same object, and get each one back typesafely without casting. A normal `Map<K,V>` only gives you one `V` type across all entries. How do you design this?"

### Follow-up probes
- Why does `Map<Class<?>, Object>` work as the backing store when `Class<?>` looks like it should reject puts?
- How does `getFavorite` return a `T` from a `Map<Class<?>, Object>` without an unchecked cast?
- Name the two limitations of this pattern — what breaks it, and what can't it hold?
- How would you close the "malicious raw `Class`" hole and get runtime, not just compile-time, safety?
- What's a *bounded* type token, and where does the JDK actually use one?

### Naive attempt
Back it with `Map<String, Object> favorites` keyed by a name string, and cast blindly at retrieval: `return (T) favorites.get(name);`.

### What breaks
The string-keyed version compiles with an unchecked-cast warning at the return, and gives zero compile-time guarantee that what's stored under a given key actually matches the type the caller expects at `get` time — a caller mistake surfaces only as a `ClassCastException`, and there's nothing tying the key to a type at all.

### The recommended approach & why
Parameterize the **key**, not the container. Use `Class<T>` objects as keys — a *type token* — since `Class` is itself generic (`String.class` is `Class<String>`, not just `Class`):
```java
public class Favorites {
    private Map<Class<?>, Object> favorites = new HashMap<>();
    public <T> void putFavorite(Class<T> type, T instance) {
        favorites.put(Objects.requireNonNull(type), instance);
    }
    public <T> T getFavorite(Class<T> type) {
        return type.cast(favorites.get(type));
    }
}
```
The map's *type* isn't a wildcard — its *key type* is: every entry can have a differently-parameterized `Class<?>` key (`Class<String>` here, `Class<Integer>` there), which is exactly where the heterogeneity comes from. `putFavorite` discards the compile-time type link between key and value (both get stored as raw `Object`), but `getFavorite` re-establishes it via `Class`'s own generic `cast` method (`T cast(Object obj)`) — a dynamic analogue of Java's cast operator that avoids ever writing an unchecked `(T)` cast in your own code.
Two real limitations: **(1)** a malicious client can bypass safety by passing a raw `Class` — this generates an unchecked warning at the call site but is otherwise unstoppable at compile time; you can close it at runtime by making `putFavorite` itself defensively cast: `favorites.put(type, type.cast(instance));`. **(2)** non-reifiable types can't be used — you can't get a distinct `Class` object for `List<String>` vs. `List<Integer>`; both share the single raw `List.class` literal, so `List<String>.class` is a compile error and there's no workaround.
When you need to restrict which types are acceptable, use a **bounded type token** — a type token constrained via a bounded type parameter or bounded wildcard, e.g. `AnnotatedElement.getAnnotation`: `public <T extends Annotation> T getAnnotation(Class<T> annotationType)`. To safely cast an unknown `Class<?>` into a bounded token at runtime, use `Class.asSubclass`, which performs the cast dynamically and throws `ClassCastException` if it's wrong — avoiding an unchecked cast entirely:
```java
return element.getAnnotation(annotationType.asSubclass(Annotation.class));
```

### Panel-ready answer checklist
- States the core trick: parameterize the key (`Class<T>`), not the container.
- Explains why `Map<Class<?>, Object>` permits heterogeneous puts — the wildcard is on the key type, not the map.
- Shows how `getFavorite` avoids an unchecked cast via `Class.cast()`, contrasted against a naive blind `(T)` cast.
- Names both limitations: raw-`Class` bypass (fixable via defensive `type.cast(instance)` at put time) and non-reifiable types (unfixable — `List<String>.class` doesn't exist).
- Defines bounded type token and cites `getAnnotation`/`asSubclass` as the JDK's real usage.
