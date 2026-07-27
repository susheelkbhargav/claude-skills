# Chapter 4: Classes and Interfaces (Items 15–25)

## Core Idea
Design types that hide their internals, minimize mutability, and prefer flexible composition and interfaces over rigid inheritance. Encapsulation and immutability are the load-bearing walls of maintainable Java.

## Frameworks Introduced
- **Item 15 — Minimize the accessibility of classes and members**: make everything as private as possible; the single most important factor in decoupling. Levels: `private` → package-private → `protected` → `public`. Public classes should rarely have public mutable fields; `public static final` array fields are a security hole (return a copy or an unmodifiable list).
- **Item 16 — In public classes, use accessor methods, not public fields**: preserves flexibility. Package-private/nested classes *may* expose fields if it reduces clutter.
- **Item 17 — Minimize mutability**: immutable classes are simpler, thread-safe, freely shared. Five rules: no mutators; class can't be extended (`final` or private ctor + factories); all fields `final`; all fields `private`; ensure exclusive access to any mutable components (defensive copies).
- **Item 18 — Favor composition over inheritance**: inheritance across package boundaries is fragile (violates encapsulation). Use a **forwarding wrapper** (decorator) holding the wrapped instance.
- **Item 19 — Design and document for inheritance or else prohibit it**: document self-use of overridable methods (`@implSpec`); constructors must not invoke overridable methods; otherwise make the class `final` or ctor private.
- **Item 20 — Prefer interfaces to abstract classes**: interfaces allow mixins, nonhierarchical types, multiple inheritance of type; combine with a skeletal `Abstract*` implementation (Template Method).
- **Item 21 — Design interfaces for posterity**: default methods let you add to interfaces, but can break existing implementors — design carefully; don't rely on default methods to fix a bad interface post-release.
- **Item 22 — Use interfaces only to define types**: not for constants — that's the **constant interface antipattern**; use a utility class or enum.
- **Item 23 — Prefer class hierarchies to tagged classes**: a class with a `type` field + switch is a verbose, error-prone hierarchy in disguise.
- **Item 24 — Favor static member classes over nonstatic**: a nonstatic inner class holds a hidden reference to its enclosing instance (memory leak + cost); make it `static` unless you need that link.
- **Item 25 — Limit source files to a single top-level class**: avoids compile-order-dependent behavior.

## Key Concepts
- **Encapsulation / information hiding**: decoupling components so they develop/test/change independently.
- **Immutability**: state fixed at construction.
- **Composition + forwarding**: wrapper holds an instance and forwards calls (decorator pattern).
- **Skeletal implementation**: `Abstract*` class implementing an interface's boilerplate.
- **Tagged class**: one class encoding multiple "flavors" via a tag field.
- **Fragile base class problem**: subclass breaks when superclass internals change.

## Mental Models
- Ask "who needs to see this?" and default the answer to *no one* — start `private`, widen only under pressure.
- Prefer immutable; if not, minimize the mutable surface ("classes should be immutable unless there's a very good reason to make them mutable").
- "Is-a" ⇒ maybe inheritance; anything less ⇒ composition. Inheritance is for *true subtypes within the same package/control*.
- Interface = capability; abstract class = shared skeleton. Ship the interface, offer the skeleton.

## Anti-patterns
- **Public mutable fields** / **public static final array**: broken encapsulation & security hole.
- **Inheriting to reuse code** across packages: fragile; use composition.
- **Constant interface** (`interface Constants { double PI = ...; }`): pollutes the type; leaks into public API.
- **Tagged classes**: verbose, memory-wasteful, uninformative — replace with subclasses.
- **Nonstatic member class by default**: needless enclosing-instance reference.
- **Calling overridable methods from a constructor**: subclass override runs before subclass fields init.

## Code Examples
```java
// Item 18 — Composition + forwarding (reusable wrapper)
public class ForwardingSet<E> implements Set<E> {
    private final Set<E> s;
    public ForwardingSet(Set<E> s){ this.s = s; }
    public boolean add(E e){ return s.add(e); }
    public int size(){ return s.size(); }
    // ... forward every method
}
public class InstrumentedSet<E> extends ForwardingSet<E> {
    private int addCount = 0;
    public InstrumentedSet(Set<E> s){ super(s); }
    @Override public boolean add(E e){ addCount++; return super.add(e); }
}
```
- **What it demonstrates**: robust "inheritance-like" reuse without fragility; works with any `Set` implementation.

```java
// Item 17 — Immutable value class
public final class Complex {
    private final double re, im;
    public Complex(double re, double im){ this.re = re; this.im = im; }
    public Complex plus(Complex c){ return new Complex(re+c.re, im+c.im); } // returns new instance
}
```

## Reference Tables
| Choice | Prefer | Over | Why |
|---|---|---|---|
| Reuse | Composition (18) | Inheritance | Encapsulation preserved |
| Type contract | Interface (20) | Abstract class | Mixins, multiple inheritance |
| Mutability | Immutable (17) | Mutable | Thread-safe, shareable |
| Access | private (15) | public | Decoupling |
| Member class | static (24) | nonstatic | No hidden ref |

## Worked Example
Why cross-package inheritance is fragile (Item 18). `InstrumentedHashSet extends HashSet` overrides `add` and `addAll` to count. But `HashSet.addAll` internally calls `add` — so adding 3 elements via `addAll` counts them **six** times (once in overridden `addAll`, once per self-called `add`). The count depends on undocumented superclass self-use, which can change between releases. The forwarding-wrapper version above has no such coupling: `InstrumentedSet` counts only its own `add`, and works over *any* `Set`.

## Key Takeaways
1. Minimize accessibility — private by default; public API is a lifelong commitment.
2. Make classes immutable unless you have a strong reason not to.
3. Favor composition + forwarding over inheritance; inherit only within your control and true subtype relationships.
4. Design interfaces to define types; pair with skeletal implementations; keep constants out.
5. Prefer class hierarchies to tagged classes; static member classes to nonstatic.
6. Design for inheritance explicitly (document self-use) or forbid it (`final`).

## Connects To
- **Ch2 (Item 2)**: builders build immutable objects.
- **Ch3 (Item 10)**: composition fixes the equals inheritance trap.
- **Ch5 (Item 28)**: interfaces + generics for flexible APIs.
- **Ch11 (Item 78–82)**: immutability = thread-safety for free.
