# Chapter 6: Enums and Annotations

## Item 34: Use enums instead of int constants

### Opening scenario
"I've got a codebase with `public static final int APPLE_FUJI = 0; public static final int ORANGE_NAVEL = 0;` — both zero. What could possibly go wrong if someone writes `int i = (APPLE_FUJI - ORANGE_TEMPLE) / APPLE_PIPPIN;`? Walk me through why the compiler doesn't stop this."

### Follow-up probes
- Why do you need `APPLE_` and `ORANGE_` prefixes on the constants at all?
- A teammate swaps in String constants instead of ints — is that actually better?
- What happens at the client's call sites if you insert a new constant in the middle of the group, or change one's value, without recompiling clients?
- How would you print one of these constants for debugging? What do you actually see?
- Why is a Java enum "far more powerful" than a C/C++ enum — what's structurally different?
- When would you reach for a switch on an enum's own value versus constant-specific method bodies?

### Naive attempt
The int enum pattern:
```java
public static final int APPLE_FUJI = 0;
public static final int APPLE_PIPPIN = 1;
public static final int APPLE_GRANNY_SMITH = 2;
public static final int ORANGE_NAVEL = 0;
public static final int ORANGE_TEMPLE = 1;
public static final int ORANGE_BLOOD = 2;
```
or its "String enum pattern" cousin using String constants instead.

### What breaks
No type safety: nothing stops `(APPLE_FUJI - ORANGE_TEMPLE) / APPLE_PIPPIN` from compiling as "tasty citrus flavored applesauce." No namespaces — hence the awkward prefixes to avoid clashes like `ELEMENT_MERCURY` vs `PLANET_MERCURY`. Because int enums are compile-time constant variables (JLS 4.12.4), their values are inlined into client bytecode (JLS 13.1); change a value and forget to recompile a client and it silently runs with stale, wrong behavior. No way to iterate the group or get its size, and printing a constant just shows a bare number. The String variant trades that last problem for hard-coded string literals in client code (typos escape compile-time detection) and string-comparison performance costs.

### The recommended approach & why
Use the `enum` type (JLS 8.9). Mechanically: an enum is a full class that exports one instance per constant via a `public static final` field; it's effectively final (no accessible constructor) so instances are instance-controlled — a generalization of the singleton (Item 3). This buys compile-time type safety (wrong-type arguments are compiler errors), per-type namespaces, safe reordering/insertion of constants (fields insulate clients from compiled-in values), a working `toString`, and free `Comparable`/`Serializable`/high-quality `Object` method implementations.

Enums can carry data and behavior — declare `private final` instance fields and a constructor (`Planet(double mass, double radius)`), and add accessor methods. Enums also have a static `values()` method returning constants in declaration order, and a generated `valueOf(String)`.

For per-constant differing *behavior* (not just data), don't switch on `this`:
```java
// Questionable — switches on its own value
public enum Operation {
  PLUS, MINUS, TIMES, DIVIDE;
  public double apply(double x, double y) {
    switch(this) {
      case PLUS:   return x + y;
      case MINUS:  return x - y;
      case TIMES:  return x * y;
      case DIVIDE: return x / y;
    }
    throw new AssertionError("Unknown op: " + this);
  }
}
```
This compiles without the closing `throw` at your peril, and silently fails at runtime if you add a constant and forget the case. Instead use constant-specific method implementations (abstract method in the enum body, overridden per constant) — the compiler forces every constant to supply an implementation:
```java
public enum Operation {
  PLUS  {public double apply(double x, double y){return x + y;}},
  MINUS {public double apply(double x, double y){return x - y;}},
  TIMES {public double apply(double x, double y){return x * y;}},
  DIVIDE{public double apply(double x, double y){return x / y;}};
  public abstract double apply(double x, double y);
}
```
When constants need to *share* code (e.g., payroll overtime pay for weekdays vs. weekends), prefer the strategy enum pattern — delegate to a private nested enum passed into the constructor — over duplicating logic or a shared switch that silently mishandles a new constant. Switching on an enum's value is fine, even good, when you're augmenting a type you don't control with an outside method (e.g., simulating a missing `inverse()` for `Operation`).

Use enums for any fixed, compile-time-known constant set — and remember the set can still evolve later (removing a constant fails loud: compile error if client references it, helpful runtime exception otherwise; that's the best you can hope for, unlike int constants going silently wrong).

### Panel-ready answer checklist
- int/String enum patterns give zero type safety, no namespace, values inlined into clients (stale after recompile skip), no iteration/size, ugly toString.
- Java enums are classes: one instance per constant via `public static final` field, instance-controlled, effectively final.
- Add data via `private final` fields + constructor; keep fields private with accessors.
- For per-constant differing behavior, use constant-specific method bodies (abstract method overridden per constant), not a switch on `this` — compiler enforces coverage on new constants.
- Use the strategy enum pattern when constants need to share partial behavior safely.
- Switching on enum values is legitimate for augmenting a type you don't own, not for behavior that belongs to the type itself.

---

## Item 35: Use instance fields instead of ordinals

### Opening scenario
"Someone wrote `int numberOfMusicians() { return ordinal() + 1; }` on an `Ensemble` enum with constants SOLO, DUET, TRIO... DECTET — what's fragile about this, and when does it silently break?"

### Follow-up probes
- What happens the moment someone reorders or inserts a constant in the middle?
- Can you add a "double quartet" (also 8 musicians) under this scheme? Why not?
- What if you need an 11-musician ensemble but there's no name for it — what are you now forced to do?
- What does the Enum spec itself say `ordinal()` is actually for?
- Is there ever a legitimate use of `ordinal()`?

### Naive attempt
```java
// Abuse of ordinal to derive an associated value - DON'T DO THIS
public enum Ensemble {
  SOLO, DUET, TRIO, QUARTET, QUINTET,
  SEXTET, SEPTET, OCTET, NONET, DECTET;
  public int numberOfMusicians() { return ordinal() + 1; }
}
```

### What breaks
Reordering constants breaks every derived value silently (no compile error — the numbers just change meaning). You can't add a second constant for a value already taken (e.g., "double quartet" = 8, same as OCTET). You can't add a constant for a value without also inserting dummy constants for every intervening unused int (e.g., adding a 12-musician "triple quartet" forces you to invent a placeholder for the unused 11).

### The recommended approach & why
Never derive a semantically meaningful value from `ordinal()`; store it explicitly in an instance field set via the constructor:
```java
public enum Ensemble {
  SOLO(1), DUET(2), TRIO(3), QUARTET(4), QUINTET(5),
  SEXTET(6), SEPTET(7), OCTET(8), DOUBLE_QUARTET(8),
  NONET(9), DECTET(10), TRIPLE_QUARTET(12);
  private final int numberOfMusicians;
  Ensemble(int size) { this.numberOfMusicians = size; }
  public int numberOfMusicians() { return numberOfMusicians; }
}
```
This decouples declaration order from the associated value entirely — duplicates and gaps become trivial. The `Enum` spec itself says `ordinal()` exists for "general-purpose enum-based data structures such as `EnumSet` and `EnumMap`" — i.e., library internals, not application logic. Unless you're writing that kind of data structure, avoid `ordinal()` entirely.

### Panel-ready answer checklist
- `ordinal()` returns declaration position — it changes if constants are reordered/inserted, so deriving business meaning from it is a maintenance trap.
- It also can't represent duplicate values or represent a value without back-filling every unused intermediate int.
- Fix: add a `private final` field, set it via the constructor, expose an accessor.
- The Enum javadoc itself scopes `ordinal()` to EnumSet/EnumMap-style internals — treat that as its only sanctioned use.

---

## Item 36: Use EnumSet instead of bit fields

### Opening scenario
"Here's a `Text` API: `STYLE_BOLD = 1<<0, STYLE_ITALIC = 1<<1, ...` combined with bitwise OR into a bit field parameter. What's wrong with this once it's actually in use, and what happens when you print one of these combined values?"

### Follow-up probes
- Isn't bitwise OR/AND for sets actually fast — why give that up?
- How would you even iterate the elements represented by a bit field?
- What happens when you need more bits than your chosen type (int/long) has?
- Why not just switch to enums but still keep the int/bitmask style for passing sets around?
- What's the internal representation of `EnumSet` — why is it fast?
- Any real limitation of `EnumSet` today?

### Naive attempt
```java
// Bit field enumeration constants - OBSOLETE!
public class Text {
  public static final int STYLE_BOLD          = 1 << 0; // 1
  public static final int STYLE_ITALIC        = 1 << 1; // 2
  public static final int STYLE_UNDERLINE     = 1 << 2; // 4
  public static final int STYLE_STRIKETHROUGH = 1 << 3; // 8
  public void applyStyles(int styles) { ... }
}
// text.applyStyles(STYLE_BOLD | STYLE_ITALIC);
```

### What breaks
All the disadvantages of int constants (Item 34) plus more: a bit field printed as a raw number is even harder to decode than a single int constant; there's no easy way to iterate the set's elements; and you must commit upfront to a bit width (`int` = 32, `long` = 64) — exceeding it means an API-breaking type change.

### The recommended approach & why
Use `java.util.EnumSet`, backed internally by a bit vector (a single `long` when the enum has ≤64 elements), giving bit-field-comparable performance while implementing the full `Set` interface — type safety, iteration, interoperability, and bulk ops (`removeAll`, `retainAll`) implemented under the hood with the same bitwise arithmetic you'd have hand-rolled, without the manual twiddling:
```java
// EnumSet - a modern replacement for bit fields
public class Text {
  public enum Style { BOLD, ITALIC, UNDERLINE, STRIKETHROUGH }
  public void applyStyles(Set<Style> styles) { ... }
}
// text.applyStyles(EnumSet.of(Style.BOLD, Style.ITALIC));
```
Accept the `Set<Style>` interface type in the API, not `EnumSet<Style>` directly (Item 64) — clients should be free to pass any `Set` implementation, even though `EnumSet` is clearly the best choice. One real gap as of Java 9: no built-in immutable `EnumSet`; wrap with `Collections.unmodifiableSet` if needed, at some cost to conciseness and performance.

### Panel-ready answer checklist
- Bit fields inherit every int-constant weakness (Item 34) plus opaque printed values and a hard-coded bit-width ceiling.
- `EnumSet` is backed by a bit vector (single `long` for ≤64 constants) — comparable speed, full `Set` semantics.
- Bulk operations use bitwise arithmetic internally but expose a safe, typed API.
- Accept the `Set` interface in method signatures, not the `EnumSet` implementation type.
- No immutable `EnumSet` as of Java 9 — wrap with `Collections.unmodifiableSet` if immutability is required.

---

## Item 37: Use EnumMap instead of ordinal indexing

### Opening scenario
"I've got `Set<Plant>[] plantsByLifeCycle = (Set<Plant>[]) new Set[LifeCycle.values().length];` indexed by `p.lifeCycle.ordinal()`. It compiles with an unchecked-cast warning and 'works.' What's actually wrong with it?"

### Follow-up probes
- Why does this need an unchecked cast in the first place?
- If you print the array, what do you see, and what extra work do you have to do to label it?
- What's the worst-case failure mode if the index computation is ever wrong?
- Now do it for a two-dimensional relationship — like phase transitions between SOLID/LIQUID/GAS indexed by two ordinals. What's the growth rate of that table?
- Why is `groupingBy(...)` alone not equivalent to hand-building an `EnumMap`?
- Does `EnumMap` actually cost you anything relative to the raw ordinal-indexed array?

### Naive attempt
```java
// Using ordinal() to index into an array - DON'T DO THIS!
Set<Plant>[] plantsByLifeCycle =
  (Set<Plant>[]) new Set[Plant.LifeCycle.values().length];
for (int i = 0; i < plantsByLifeCycle.length; i++)
  plantsByLifeCycle[i] = new HashSet<>();
for (Plant p : garden)
  plantsByLifeCycle[p.lifeCycle.ordinal()].add(p);
```
And the 2-D version: a `Transition[][]` table indexed `[from.ordinal()][to.ordinal()]`.

### What breaks
Arrays aren't compatible with generics (Item 28), forcing an unchecked cast that won't compile cleanly. The array doesn't know what its indices mean, so output must be labeled manually. Worst of all, using the wrong ordinal is entirely the programmer's responsibility — ints don't carry enum type safety — so a mistake either silently does the wrong thing or, if you're lucky, throws `ArrayIndexOutOfBoundsException`. For the 2-D transition table, the array grows quadratically in the number of enum values even when most entries are null, and a mismatch between the table and the enum (forgotten update after adding a constant) fails at runtime as `ArrayIndexOutOfBoundsException`, `NullPointerException`, or silent wrong behavior.

### The recommended approach & why
The array is really acting as a Map from enum to value — so use `java.util.EnumMap`, a fast Map implementation designed for enum keys:
```java
Map<Plant.LifeCycle, Set<Plant>> plantsByLifeCycle =
    new EnumMap<>(Plant.LifeCycle.class);
for (Plant.LifeCycle lc : Plant.LifeCycle.values())
  plantsByLifeCycle.put(lc, new HashSet<>());
for (Plant p : garden)
  plantsByLifeCycle.get(p.lifeCycle).add(p);
```
No unsafe cast, no manual labeling (map keys print themselves), no possibility of index error. `EnumMap` is internally backed by an array too, so it's comparable in speed to the ordinal-indexed version — it just hides that implementation detail behind `Map`'s type safety. The constructor takes the key type's `Class` object as a bounded type token (Item 33).

Stream-based construction is possible but a naive `groupingBy(p -> p.lifeCycle)` won't produce an `EnumMap` — it picks its own map implementation. Use the three-arg `groupingBy(classifier, mapFactory, downstream)` to force `EnumMap`:
```java
Arrays.stream(garden)
  .collect(groupingBy(p -> p.lifeCycle,
      () -> new EnumMap<>(LifeCycle.class), toSet()));
```
Note a subtle semantic difference: the explicit `EnumMap` version always creates an entry per lifecycle; the stream version only creates entries for lifecycles actually present in the data.

For the multidimensional case (phase transitions), use a nested `EnumMap<Phase, EnumMap<Phase, Transition>>`, with the two associated phases captured on the `Transition` enum constants themselves and the outer/inner maps built via cascaded `groupingBy`/`toMap` collectors specifying `EnumMap` factories. Adding a new phase then means adding one enum constant plus its transitions — no resizing a hand-maintained quadratic array, and no way to get the indices subtly wrong.

### Panel-ready answer checklist
- Ordinal-indexed arrays need an unchecked cast (arrays + generics don't mix), require manual output labeling, and push index correctness entirely onto the programmer.
- Two-dimensional ordinal indexing (array of arrays) grows quadratically and fails opaquely (AIOOBE/NPE/silent bug) on any enum/table mismatch.
- `EnumMap` is the correct replacement — internally array-backed (same speed) but exposes full `Map` type safety and self-labeling keys; constructor takes a `Class` bounded type token.
- `groupingBy(classifier)` alone won't yield an `EnumMap` — you must supply the map factory via the three-arg `groupingBy` overload.
- For pairwise/multidimensional relationships, nest `EnumMap<X, EnumMap<Y, V>>` rather than a raw 2-D ordinal array.
- Rule of thumb: rarely if ever should application code call `Enum.ordinal()` (ties back to Item 35).

---

## Item 38: Emulate extensible enums with interfaces

### Opening scenario
"Your calculator's `Operation` enum (PLUS, MINUS, TIMES, DIVIDE) needs to let plugin authors add their own operations — EXP, REMAINDER — without you recompiling your library. Enums can't be extended or subclassed. How do you get extensibility anyway?"

### Follow-up probes
- Why doesn't the language allow one enum to extend another — what would go wrong if it did?
- Where's the one case Bloch actually endorses this — is it common?
- If you make `Operation` an interface and `BasicOperation`/`ExtendedOperation` implement it, what do you give up?
- How do you write a method that iterates "all operations of some caller-supplied extension type" if you can't enumerate over an interface directly?
- Compare passing a `Class<T>` token vs. a `Collection<? extends Operation>` — what's the tradeoff?
- Can BasicOperation and ExtendedOperation share code (like the `toString`/symbol logic)? What's the cost of not being able to?

### Naive attempt
Trying to literally subclass one enum from another — not legal in Java; there's no syntax for it because enum extensibility was deliberately excluded from the language.

### What breaks
Even if it were legal, the language designers rejected it deliberately: it's confusing that instances of an "extension" type would be instances of the base type but not vice versa, there'd be no clean way to enumerate all elements of a base type plus its extensions, and it would complicate the whole enum design/implementation.

### The recommended approach & why
Emulate extensibility with an interface: define an interface for the opcode behavior, and have the base enum implement it.
```java
// Emulated extensible enum using an interface
public interface Operation {
  double apply(double x, double y);
}
public enum BasicOperation implements Operation {
  PLUS("+") { public double apply(double x, double y){return x + y;} },
  MINUS("-") { public double apply(double x, double y){return x - y;} },
  TIMES("*") { public double apply(double x, double y){return x * y;} },
  DIVIDE("/") { public double apply(double x, double y){return x / y;} };
  private final String symbol;
  BasicOperation(String symbol) { this.symbol = symbol; }
  @Override public String toString() { return symbol; }
}
```
`BasicOperation` itself isn't extensible, but `Operation` is — and APIs should be written against `Operation`, not `BasicOperation`. A separate enum can implement the same interface:
```java
public enum ExtendedOperation implements Operation {
  EXP("^") { public double apply(double x, double y){return Math.pow(x, y);} },
  REMAINDER("%") { public double apply(double x, double y){return x % y;} };
  private final String symbol;
  ExtendedOperation(String symbol) { this.symbol = symbol; }
  @Override public String toString() { return symbol; }
}
```
You can pass a single extension instance anywhere an `Operation` is expected, or pass the entire extension enum type. Two ways to do the latter: a bounded type token `Class<T>` where `T extends Enum<T> & Operation`, iterated via `opEnumType.getEnumConstants()`; or a bounded wildcard `Collection<? extends Operation>`, which is less type-ceremony and lets a caller mix operations from multiple implementing enums, at the cost of losing the ability to use `EnumSet`/`EnumMap` on the result.

The real cost: implementations can't be inherited between enum types implementing the same interface — the symbol-storage logic is duplicated in `BasicOperation` and `ExtendedOperation`. Non-state-dependent shared behavior can move into interface default methods (Item 20); anything state-dependent duplicated at scale should go into a shared helper class/method. The Java libraries use this exact pattern — `java.nio.file.LinkOption` implements both `CopyOption` and `OpenOption`.

### Panel-ready answer checklist
- Enums can't extend enums by design — extension-type instances being base-type instances (but not reverse) and losing a clean "enumerate base + extensions" story are the stated reasons.
- Workaround: define an interface for the behavior; base enum and extension enum(s) both implement it; APIs are written against the interface type.
- Iterate an extension type via a bounded type token (`Class<T extends Enum<T> & Operation>` + `getEnumConstants()`) or via `Collection<? extends Operation>` (more flexible, but forfeits `EnumSet`/`EnumMap`).
- Cost: no implementation inheritance across enum types — duplicate small logic, or move non-state-dependent code into interface default methods, or a shared helper for the rest.
- Real-world precedent: `java.nio.file.LinkOption implements CopyOption, OpenOption`.

---

## Item 39: Prefer annotations to naming patterns

### Opening scenario
"Old JUnit 3 required test methods to start with `test`. Someone typos `tsetSafetyOverride` instead of `testSafetyOverride`. What does the framework do, and why is that dangerous?"

### Follow-up probes
- Besides typos, what's a second failure mode of naming-pattern conventions?
- How would a naming pattern let you attach a parameter — say, "this test must throw `IndexOutOfBoundsException`" — to a specific test method? What breaks if you try?
- What are `@Retention` and `@Target` doing, mechanically, on your own annotation type declaration — what happens if you omit `@Retention(RUNTIME)`?
- If a test method is annotated but is an instance method (not static), does it fail to compile? Where does that actually surface?
- What's the difference between an array-valued annotation parameter and `@Repeatable`? What extra care does processing `@Repeatable` annotations require?
- Why can `ExceptionTest`'s parameter be `Class<? extends Throwable>` instead of a String class name, and why does that matter for catching errors early?

### Naive attempt
Encode meaning into names: `test`-prefixed methods, or worse, "encode the expected exception type name into the test method name using some elaborate naming pattern."

### What breaks
Typos cause silent failures — a misnamed method is simply never run, with no compiler or framework complaint (false sense of security). Naming patterns can't be restricted to the right kind of program element — you can name a whole class `TestSafetyMechanisms` hoping it gets special treatment and the framework will just as silently ignore it. And they provide no clean way to associate parameter data (like an expected exception type) with the element — you'd have to hand-roll fragile string encoding, and the compiler can't verify the encoded string actually names a real exception type until you try to run the test.

### The recommended approach & why
Use annotations (JLS 9.7). A marker annotation with no parameters:
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Test { }
```
`@Retention(RUNTIME)` keeps the annotation visible to reflection at runtime (without it the annotation is invisible to any test tool); `@Target(METHOD)` restricts legal use to method declarations — misapplying it, or misspelling `@Test`, is a compile error, unlike a naming-pattern typo. Reflectively, a test runner uses `Method.isAnnotationPresent(Test.class)` and `Method.invoke(null)`, catching `InvocationTargetException` (test failure — unwrap with `getCause()`) versus any other exception (invalid use of the annotation, e.g. on an instance method — not caught at compile time absent an annotation processor, but at least isolated and reported).

For parameterized behavior, give the annotation a typed parameter, e.g. `Class<? extends Throwable> value()` on `ExceptionTest` — a bounded type token (Item 33) — so the compiler guarantees the parameter is an actual `Throwable` subtype; the only residual runtime risk is `TypeNotPresentException` if the referenced class file is missing at runtime despite being valid at compile time.

For "any one of several exceptions," two options: an array-valued parameter (`Class<? extends Exception>[] value()`, using `{...}` syntax for multiple elements) or, since Java 8, `@Repeatable` — annotate the annotation type with `@Repeatable(ContainerAnnotation.class)` where the container type has a single array-valued `value()` and its own correct `@Retention`/`@Target`. Processing repeatables requires care: a repeated annotation is stored under a synthetic containing annotation, so `isAnnotationPresent` on the plain type misses repeats and vice versa — you must check both the annotation type and its container, or use `getAnnotationsByType`, which handles both uniformly.

### Panel-ready answer checklist
- Naming patterns fail silently on typos, can't be restricted to the right element kind, and have no clean way to attach compiler-checked parameters.
- Annotations fix all three: `@Target`/`@Retention` meta-annotations enforce legal placement and runtime visibility; misuse is a compile error, not a silent no-op.
- Reflection API: `isAnnotationPresent`, `Method.invoke`, distinguishing `InvocationTargetException` (real test failure) from other exceptions (invalid annotation use).
- Parameterized annotations should use bounded type tokens (`Class<? extends X>`) so the compiler enforces valid values; `TypeNotPresentException` is the only residual runtime risk.
- Multivalued: array parameter vs. `@Repeatable` — repeatables need checking both the annotation type and its auto-generated container type (or `getAnnotationsByType`) to avoid silently missing annotations.
- Bottom line: toolsmiths should define annotation types instead of naming patterns; ordinary programmers should just consistently use the predefined ones (ties to Items 40, 27).

---

## Item 40: Consistently use the Override annotation

### Opening scenario
"Here's a `Bigram` class with `equals(Bigram b)` and matching `hashCode()`. The `main` method adds 260 supposedly-duplicate bigrams to a `HashSet` and prints `s.size()`. It prints 260, not 26. What's wrong, and why doesn't the compiler catch it?"

### Follow-up probes
- What method did `Bigram` actually inherit from `Object`, and what does it compare?
- If you add `@Override` to the broken `equals(Bigram b)`, what exact compiler error do you get?
- Is there ever a case where skipping `@Override` is fine, per Bloch?
- Does `@Override` apply to methods overriding an interface declaration, not just a superclass? Why does that matter more since default methods exist?
- In an abstract class or interface itself, should you annotate methods that override a supertype's methods, even if they're still abstract?
- What's the IDE's role here beyond what the compiler alone gives you?

### Naive attempt
```java
public boolean equals(Bigram b) {
  return b.first == first && b.second == second;
}
public int hashCode() { return 31 * first + second; }
```
Looks like a correctly overridden `equals`/`hashCode` pair (Items 10, 11).

### What breaks
The parameter type is `Bigram`, not `Object` — so this isn't an override of `Object.equals(Object)` at all, it's an overload (Item 52). `Bigram` therefore still inherits `Object.equals`, which is reference/identity equality — identical to `==`. Each of the ten copies of a given bigram is a distinct object, so `Object.equals` deems them all unequal, and the `HashSet` keeps all 260.

### The recommended approach & why
Annotate every method you believe overrides a supertype declaration with `@Override`:
```java
@Override public boolean equals(Bigram b) { ... }
```
This one line changes nothing about runtime behavior, but the compiler will now refuse to compile it: `method does not override or implement a method from a supertype` — because the parameter type doesn't match `Object.equals(Object)`. That error is what makes you fix it correctly:
```java
@Override public boolean equals(Object o) {
  if (!(o instanceof Bigram)) return false;
  Bigram b = (Bigram) o;
  return b.first == first && b.second == second;
}
```
Rule: use `@Override` on every method you intend to override a superclass declaration, with one narrow exception — a concrete (non-abstract) class overriding an abstract method from its superclass doesn't strictly need it, because the compiler already errors if you fail to implement an abstract method. It's still harmless to annotate those too if you want the visual signal. `@Override` also applies to methods overriding interface declarations, not just class hierarchies — and with default methods now part of the language, annotating concrete implementations of interface methods is good practice to confirm the signature actually matches (though if you know the interface has no default methods, omitting it to reduce clutter is a defensible call). Inside an abstract class or an interface itself, annotate all methods — concrete or still-abstract — believed to override a superclass/superinterface method (e.g., `Set` should annotate every method it re-declares from `Collection`, ensuring it never accidentally introduces a new method instead of refining an existing contract). Most IDEs both auto-insert `@Override` on override and separately warn when a method silently overrides without the annotation — between compiler errors (unintentional non-overrides) and IDE warnings (unintentional overrides), consistent use closes the loop in both directions.

### Panel-ready answer checklist
- `equals(Bigram b)` overloads, not overrides, `Object.equals(Object)` — so `Bigram` silently inherits identity-based equality, and the `HashSet` treats every copy as distinct.
- `@Override` converts this class of bug into a compile-time error ("does not override or implement a method from a supertype").
- Rule: annotate every intended override, with the sole exception of a concrete class overriding an already-abstract superclass method (compiler catches that omission anyway).
- Applies equally to interface method overrides — worth doing on default-method interfaces to verify signature correctness.
- In abstract classes/interfaces, annotate all believed overrides (concrete or still-abstract) — e.g., `Set` re-declaring `Collection` methods.
- IDEs both insert it automatically and warn on missing-but-actual overrides — use both signals.

---

## Item 41: Use marker interfaces to define types

### Opening scenario
"`Serializable` has zero methods. Someone on the team says marker annotations made marker interfaces obsolete, so we should replace `implements Serializable` with `@Serializable` everywhere. Do you agree?"

### Follow-up probes
- What compile-time guarantee does implementing `Serializable` give you that an equivalent annotation wouldn't?
- Why does `ObjectOutputStream.writeObject(Object obj)` still let you pass an unserializable object and blow up at runtime — doesn't `Serializable` exist to prevent exactly that?
- Is `Set` a marker interface? Why does Bloch hedge on that?
- What's the actual decision rule for choosing a marker interface over a marker annotation?
- When are you flatly forced into an annotation instead of an interface?
- What's the one real advantage a marker annotation has over a marker interface?

### Naive attempt
Assume marker annotations strictly supersede marker interfaces because annotations are the newer, more general mechanism — so migrate every marker interface to an annotation for consistency.

### What breaks
That assumption is stated as flatly incorrect. A marker interface defines an actual type implemented by instances of the marked class; a marker annotation does not create a type at all. This has real teeth: if `ObjectOutputStream.writeObject` had been declared to take a `Serializable` argument (instead of `Object`), an attempt to serialize a non-serializable object would be caught at compile time via ordinary type checking. Because the API's argument type is `Object`, that opportunity is missed and such an attempt only fails at runtime — a concrete illustration of what you lose by not having the marker be a real type.

### The recommended approach & why
Two structural advantages of marker interfaces over marker annotations: (1) they define a type, enabling compile-time error detection wherever an API is written to accept that type rather than a supertype; (2) they can be targeted more precisely — an annotation with `@Target(ElementType.TYPE)` can land on any class or interface, but a marker interface can be declared to `extend` the one specific interface it's meant to restrict itself to, guaranteeing every marked type is also a subtype of that interface. (`Set` is arguably a "restricted marker interface" applicable only to `Collection` subtypes — though it's not usually called one because it also refines the contracts of `add`, `equals`, `hashCode`, not just marks.)

Marker annotations have one structural advantage: they're part of the broader annotation facility, so they compose consistently within annotation-based frameworks.

Decision rule: if the marker must apply to a program element other than a class or interface (fields, parameters, etc.), you have no choice — only classes/interfaces can implement/extend, so it must be an annotation. If it's restricted to classes/interfaces, ask: "Might I want to write one or more methods that accept only objects bearing this marking?" If yes, use a marker interface — you get to use it as a parameter type and gain compile-time checking. If you're confident you'll never write such a method, a marker annotation is the better choice, especially if the broader system already leans heavily on annotations.

### Panel-ready answer checklist
- False premise: marker interfaces define an actual type; marker annotations don't — that's the crux, not just historical inertia.
- Concrete cost example: `ObjectOutputStream.writeObject(Object)` takes `Object`, not `Serializable`, so it misses the compile-time check the interface could have enabled — unserializable arguments fail only at runtime.
- Marker interfaces can be scoped precisely by extending exactly the interface they're meant to restrict to; `@Target(TYPE)` annotations can land on any class/interface indiscriminately.
- Marker annotations win only when the marker must target non-type elements, or must integrate into an annotation-heavy framework.
- Decision test: "will I ever want a method parameter typed to only accept marked instances?" — yes means interface, no means annotation.
- Framed as the mirror of Item 22 ("don't use an interface to define constants without a type"): if you do want a type, an interface — even a marker one — is the right tool.
