# Chapter 2: Creating and Destroying Objects (Items 1–9)

## Core Idea
Control *how* objects come into existence and *when* they leave: prefer flexible creation idioms (factories, builders) over raw constructors, avoid needless allocation, and make resource cleanup deterministic instead of trusting the garbage collector.

## Frameworks Introduced
- **Item 1 — Static factory methods instead of constructors**: a `public static` method returning an instance.
  - When to use: you want a *named* creation point, control over instance count (caching/singletons), or a return type that's a subtype/interface not visible to the caller.
  - How: `public static Boolean valueOf(boolean b){ return b ? Boolean.TRUE : Boolean.FALSE; }`. Naming conventions: `from`, `of`, `valueOf`, `instance`/`getInstance`, `create`/`newInstance`, `getType`, `newType`.
  - Cost: classes with only static factories (no public/protected ctor) can't be subclassed; factories are harder to find in docs than constructors.
- **Item 2 — Builder when faced with many constructor parameters**: fluent step-by-step construction.
  - When to use: 4+ parameters, especially many optional ones; telescoping constructors and JavaBeans (setters) both fail — telescoping is unreadable, JavaBeans leaves objects mutable and half-built.
  - How: static nested `Builder` with a fluent setter per field returning `this`; `build()` returns the immutable outer object. Simulates named optional params.
- **Item 3 — Enforce the singleton property with a private constructor or an enum type**: single-element enum is the best singleton — concise, serialization-safe, reflection-proof.
- **Item 4 — Enforce noninstantiability with a private constructor**: utility classes (only static members) get a `private` ctor that throws `AssertionError`.
- **Item 5 — Prefer dependency injection to hardwiring resources**: pass resources into the constructor rather than instantiating them internally; a class depending on a resource should not create it. Enables testing and flexibility.
- **Item 6 — Avoid creating unnecessary objects**: reuse a single object over creating equivalent ones repeatedly.
- **Item 7 — Eliminate obsolete object references**: null out references you're done with when *you* manage memory (e.g. own array in a stack), else you get a memory leak.
- **Item 8 — Avoid finalizers and cleaners**: unpredictable, dangerous, slow. Use `AutoCloseable` + explicit `close()` instead. Cleaners only as a safety net.
- **Item 9 — Prefer try-with-resources to try-finally**: for anything `AutoCloseable`.

## Key Concepts
- **Static factory**: static method returning an instance; not the GoF Factory Method pattern.
- **Telescoping constructor**: constructor overloads adding one param each — unreadable at scale.
- **Fluent API**: chained calls returning `this`.
- **Utility class**: non-instantiable class of static members (e.g. `Math`, `Collections`).
- **Autoboxing trap**: mixing primitive and boxed types silently creates objects.
- **Obsolete reference**: a reference the program will never dereference again but still holds, preventing GC.

## Mental Models
- Think of a constructor as one anonymous option; a static factory as a *named menu* of construction strategies you can evolve without breaking callers.
- Use a Builder when the parameter list is a "form to fill in," not a fixed tuple.
- Treat memory leaks in a managed language as *unintentional object retention* — hunt the reference that won't die.
- Prefer "reuse" only when the object is immutable or provably stateless; don't cache mutable objects to "save allocation."

## Anti-patterns
- **Telescoping constructors** for many optional params: unreadable, error-prone (silent param swaps).
- **JavaBeans setters** for construction: object is mutable and can be observed in an inconsistent state; precludes immutability.
- **`new String("x")`**: creates a needless object; the literal already is a `String`.
- **`new Boolean(b)`**: prefer `Boolean.valueOf(b)`; avoid autoboxing in loops (`Long sum` vs `long sum` = huge slowdown).
- **Relying on finalizers/`System.gc()`** for cleanup: no guarantee they run, ever.
- **Over-nulling references**: nulling out is the exception (self-managed memory), not the norm; let variables fall out of scope normally.

## Code Examples
```java
// Item 2 — Builder pattern
public class NutritionFacts {
    private final int servings, calories, fat;
    public static class Builder {
        private final int servings;         // required
        private int calories = 0, fat = 0;  // optional, defaulted
        public Builder(int servings){ this.servings = servings; }
        public Builder calories(int v){ calories = v; return this; }
        public Builder fat(int v){ fat = v; return this; }
        public NutritionFacts build(){ return new NutritionFacts(this); }
    }
    private NutritionFacts(Builder b){ servings=b.servings; calories=b.calories; fat=b.fat; }
}
// NutritionFacts n = new NutritionFacts.Builder(1).calories(100).fat(35).build();
```
- **What it demonstrates**: named optional parameters + immutability + validation in `build()`.

```java
// Item 3 — Enum singleton (preferred)
public enum Elvis { INSTANCE; public void sing(){ /* ... */ } }

// Item 9 — try-with-resources
static String firstLine(String path) throws IOException {
    try (BufferedReader br = new BufferedReader(new FileReader(path))) {
        return br.readLine();
    }
}
```

## Reference Tables
| Need | Use | Not |
|------|-----|-----|
| Named/controlled creation | Static factory (Item 1) | Bare constructor |
| Many optional params | Builder (Item 2) | Telescoping ctor / setters |
| Exactly one instance | `enum` singleton (Item 3) | `public static final` field + reflection-vulnerable ctor |
| No instances (utility) | private ctor throwing (Item 4) | abstract class |
| Resource cleanup | try-with-resources (Item 9) | finalizer / try-finally |

## Worked Example
Refactoring a leak (Item 7). A hand-rolled stack keeps its own backing array. `pop()` returns `elements[--size]` but *leaves the slot pointing at the popped object* — the array still references it, so it never gets collected even though the program will never touch it again:
```java
public Object pop(){
    if (size == 0) throw new EmptyStackException();
    Object result = elements[--size];
    elements[size] = null; // <-- eliminate obsolete reference
    return result;
}
```
Without the null-out line, a stack that grows then shrinks silently retains every popped object. The fix is one line, but you only null out *because this class manages its own memory* — for ordinary local variables, going out of scope is enough.

## Key Takeaways
1. Reach for static factories and builders before public constructors when you need naming, instance control, or many params.
2. Single-element `enum` is the simplest correct singleton.
3. Immutable, half-built-proof construction beats setter-based construction.
4. Inject dependencies; don't hardwire them — it's the difference between testable and untestable.
5. Deterministic cleanup = `AutoCloseable` + try-with-resources; finalizers/cleaners are last-resort safety nets, never primary cleanup.
6. Memory leaks in Java come from references you forgot to release (caches, listeners, self-managed arrays).

## Connects To
- **Ch3 (Item 17)**: builders enable immutability.
- **Ch4 (Item 18)**: DI (Item 5) favors composition over inheritance for wiring.
- **Ch6 (Item 34)**: enum singleton ties to enum design.
- **Ch9 (Item 57–59)**: scope minimization complements eliminating obsolete references.
