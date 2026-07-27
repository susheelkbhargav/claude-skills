# Chapter 6: Enums and Annotations (Items 34–41)

## Core Idea
Enums are full-featured classes, not glorified `int`s — use them for fixed sets of constants and their behavior. Annotations replace fragile naming patterns for metadata. Prefer both to the string/int idioms they supplant.

## Frameworks Introduced
- **Item 34 — Use enums instead of `int` constants**: enums are typesafe, namespaced, printable, and can carry data + behavior. Attach fields via a constructor; add per-constant behavior with **constant-specific method bodies** (abstract method overridden per constant).
- **Item 35 — Use instance fields instead of ordinals**: never derive a value from `ordinal()` — it breaks on reordering/insertion; store the value in a field.
- **Item 36 — Use `EnumSet` instead of bit fields**: `EnumSet` is as compact/fast as bit flags but typesafe and readable.
- **Item 37 — Use `EnumMap` instead of ordinal indexing**: `EnumMap` (array-backed, fast) replaces `array[enum.ordinal()]` indexing safely.
- **Item 38 — Emulate extensible enums with interfaces**: enums can't extend, but can implement an interface — clients can add new "constants" via other enums implementing the same interface (e.g. `Operation` interface).
- **Item 39 — Prefer annotations to naming patterns**: e.g. `@Test` instead of methods that must start with `test`. Annotations are typesafe, can carry parameters, and are processed by tools/reflection.
- **Item 40 — Consistently use the `@Override` annotation**: lets the compiler catch accidental overloads that should be overrides. Use on every method you believe overrides a supertype method.
- **Item 41 — Use marker interfaces to define types**: a marker interface (e.g. `Serializable`) defines a type usable at compile time and can target more precisely than a marker annotation.

## Key Concepts
- **Enum**: a class with a fixed set of instances (constants), instance-controlled.
- **Constant-specific method body**: per-constant override of an abstract method declared in the enum.
- **`ordinal()`**: position of a constant — an implementation detail for `EnumSet`/`EnumMap`, not for app logic.
- **`EnumSet` / `EnumMap`**: high-performance Set/Map specialized for enum keys.
- **Marker interface vs marker annotation**: empty interface signaling a property vs annotation with no elements.

## Mental Models
- If a variable's value comes from a small fixed set, it's an enum — even if it currently has no behavior.
- Put "what varies per constant" into the enum via a constant-specific method, turning a `switch` into polymorphism.
- Use the **strategy enum** pattern when multiple constants share behavior: delegate to a nested private strategy enum instead of a switch.
- Marker *interface* when the marked thing is a type you'll use in APIs; marker *annotation* for pure metadata / non-class targets.

## Anti-patterns
- **`int`/`String` constant enums** (`public static final int SEASON_SPRING = 0`): no type safety, no namespace, brittle.
- **Deriving data from `ordinal()`**: reordering constants silently corrupts values.
- **Bit fields** (`int flags = BOLD | ITALIC`): unreadable, untypesafe → `EnumSet`.
- **`array[ordinal()]` indexing**: fragile, uncheckable → `EnumMap`.
- **`switch` on enum for per-constant behavior**: prefer constant-specific methods; a switch forgets new constants silently.
- **Naming patterns** (`testX`, `_field`): no compiler/tool support → annotations.

## Code Examples
```java
// Item 34 — enum with data + constant-specific behavior
public enum Operation {
    PLUS("+"){ public double apply(double x,double y){ return x+y; } },
    MINUS("-"){ public double apply(double x,double y){ return x-y; } },
    TIMES("*"){ public double apply(double x,double y){ return x*y; } },
    DIVIDE("/"){ public double apply(double x,double y){ return x/y; } };
    private final String symbol;
    Operation(String symbol){ this.symbol = symbol; }
    public abstract double apply(double x, double y);
    @Override public String toString(){ return symbol; }
}
```
- **What it demonstrates**: each constant carries data (`symbol`) and its own behavior — no `switch`, impossible to forget a case.

```java
// Item 38 — extensible via interface
public interface Op { double apply(double x, double y); }
public enum BasicOp implements Op { PLUS { public double apply(double a,double b){return a+b;} } }
public enum ExtendedOp implements Op { EXP { public double apply(double a,double b){return Math.pow(a,b);} } }
```

## Reference Tables
| Legacy idiom | Replace with | Item |
|---|---|---|
| `int` constants | `enum` | 34 |
| value from `ordinal()` | instance field | 35 |
| `int` bit flags | `EnumSet` | 36 |
| `array[ordinal()]` | `EnumMap` | 37 |
| naming pattern | annotation | 39 |
| `Serializable`-style marker | marker interface | 41 |

## Worked Example
Grouping with `EnumMap` (Item 37). Given `Plant.LifeCycle` (ANNUAL, PERENNIAL, BIENNIAL), group a garden by life cycle. The ordinal-indexed version `Set<Plant>[] table = new Set[3]` indexed by `lc.ordinal()` compiles with unchecked warnings, needs manual bounds, and prints ordinals. The `EnumMap` version is typesafe and readable:
```java
Map<Plant.LifeCycle, Set<Plant>> byCycle = new EnumMap<>(Plant.LifeCycle.class);
for (Plant.LifeCycle lc : Plant.LifeCycle.values()) byCycle.put(lc, new HashSet<>());
for (Plant p : garden) byCycle.get(p.lifeCycle).add(p);
// Even cleaner with streams: groupingBy(p -> p.lifeCycle, () -> new EnumMap<>(...), toSet())
```
`EnumMap` matches array performance (it's array-backed internally) with none of the ordinal fragility.

## Key Takeaways
1. Enums replace `int`/`String` constants — typesafe, self-describing, extensible with data and behavior.
2. Never derive meaning from `ordinal()`; store it in a field.
3. `EnumSet`/`EnumMap` replace bit fields and ordinal-indexed arrays with equal speed and full safety.
4. Emulate extensibility via a shared interface, since enums can't be subclassed.
5. Constant-specific methods beat `switch` for per-constant behavior.
6. Annotations replace naming patterns; always use `@Override`; marker interfaces define types annotations can't.

## Connects To
- **Ch2 (Item 3)**: single-element enum = best singleton.
- **Ch3 (Item 14)**: enums are naturally `Comparable` (by declaration order).
- **Ch5 (Item 33)**: `Class` type tokens relate to `@Override`/reflection processing.
- **Ch12 (Item 89)**: enums give free serialization-safe instance control.
