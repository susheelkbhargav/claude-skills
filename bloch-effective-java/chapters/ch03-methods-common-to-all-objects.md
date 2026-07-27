# Chapter 3: Methods Common to All Objects (Items 10–14)

## Core Idea
`Object`'s nonfinal methods (`equals`, `hashCode`, `toString`, `clone`) and `Comparable.compareTo` have general contracts. Override them correctly *as a set*, or subtle bugs (broken collections, mismatched hashing) appear far from the mistake.

## Frameworks Introduced
- **Item 10 — Obey the general contract when overriding `equals`**: equals must be reflexive, symmetric, transitive, consistent, and `x.equals(null)==false`.
  - When to override: value classes needing logical equality (not identity) that aren't instance-controlled.
  - Recipe: `==` self-check → `instanceof` type check → cast → compare significant fields (primitives with `==`, floats with `Float.compare`/`Double.compare`, objects with `equals`, arrays with `Arrays.equals`).
- **Item 11 — Always override `hashCode` when you override `equals`**: equal objects must have equal hash codes, or `HashMap`/`HashSet` break.
  - How: `int result = Type.hashCode(firstField); result = 31*result + Type.hashCode(nextField); ...`. `31` is odd prime; multiplication makes order significant. Or `Objects.hash(...)` (simpler, slower).
- **Item 12 — Always override `toString`**: return concise, useful, informative representation; ideally document format (or explicitly don't) and provide accessors for all data in it.
- **Item 13 — Override `clone` judiciously**: `Cloneable` is a broken mixin; if you must, call `super.clone()` and deep-copy mutable internals. Prefer a **copy constructor / copy factory** instead.
- **Item 14 — Consider implementing `Comparable`**: gives ordering + interop with sorted collections, `Arrays.sort`, etc. `compareTo` contract mirrors `equals`; use `Integer.compare` etc., or the `Comparator` builder methods — never rely on `-` subtraction (overflow).

## Key Concepts
- **General contract**: behavioral spec callers depend on; violating it breaks classes that rely on the method.
- **Logical vs reference equality**: value equality vs identity (`==`).
- **Symmetry/transitivity trap**: adding a value component in a subclass breaks the `equals` contract — there's no way to extend an instantiable class and add a value component while preserving it.
- **Hash collision cascade**: a constant `hashCode` is legal but degrades hash tables to linked lists (O(n)).

## Mental Models
- Treat `equals`/`hashCode` as a *pair* — never override one alone.
- Prefer **composition over inheritance** to sidestep the equals-symmetry problem: put the superclass instance in a private field, add a view method.
- Think of `compareTo` as "`equals` with a direction," and keep `x.compareTo(y)==0` consistent with `x.equals(y)` (or document the exception, e.g. `BigDecimal`).

## Anti-patterns
- **Overriding `equals` but not `hashCode`**: objects vanish from hash-based collections.
- **`equals` on `Object` param typed wrong**: `public boolean equals(MyType o)` *overloads*, not overrides — annotate `@Override`.
- **Subtraction-based comparators** (`a.x - b.x`): integer overflow yields wrong order.
- **Using `clone`** for copying: prefer copy constructor/factory (`new TreeSet<>(s)`), which take interface-typed arguments and don't require `Cloneable`.
- **`float`/`double` field equality with `==`** in equals: use `Float.compare`/`Double.compare` for `NaN`/`-0.0` correctness.

## Code Examples
```java
@Override public boolean equals(Object o) {
    if (o == this) return true;
    if (!(o instanceof PhoneNumber)) return false;
    PhoneNumber pn = (PhoneNumber) o;
    return pn.lineNum == lineNum && pn.prefix == prefix && pn.areaCode == areaCode;
}
@Override public int hashCode() {
    int result = Short.hashCode(areaCode);
    result = 31 * result + Short.hashCode(prefix);
    result = 31 * result + Short.hashCode(lineNum);
    return result;
}
```
- **What it demonstrates**: contract-correct `equals` + `31*`-accumulator `hashCode` treating field order as significant.

```java
// Item 14 — Comparator builder, no subtraction
private static final Comparator<PhoneNumber> COMPARATOR =
    comparingInt((PhoneNumber pn) -> pn.areaCode)
        .thenComparingInt(pn -> pn.prefix)
        .thenComparingInt(pn -> pn.lineNum);
public int compareTo(PhoneNumber pn){ return COMPARATOR.compare(this, pn); }
```

## Reference Tables
| equals contract clause | Meaning |
|---|---|
| Reflexive | `x.equals(x)` |
| Symmetric | `x.equals(y) ⇔ y.equals(x)` |
| Transitive | `x=y, y=z ⇒ x=z` |
| Consistent | repeated calls agree if objects unchanged |
| Non-null | `x.equals(null)==false` |

## Worked Example
Why symmetry breaks with inheritance. `Point` has `equals` on `x,y`. `ColorPoint extends Point` adds `color`. A naive `ColorPoint.equals` that requires color makes `p.equals(cp)` true (Point ignores color) but `cp.equals(p)` false — **asymmetric**. Trying to fix it with "blind" mixed comparison then breaks **transitivity**. Bloch's resolution: don't extend — *compose*:
```java
public class ColorPoint {
    private final Point point;
    private final Color color;
    public Point asPoint(){ return point; }         // view
    @Override public boolean equals(Object o){
        if(!(o instanceof ColorPoint)) return false;
        ColorPoint cp = (ColorPoint) o;
        return cp.point.equals(point) && cp.color.equals(color);
    }
}
```
There is **no** way to extend an instantiable class with a value component and preserve the equals contract — composition is the way out.

## Key Takeaways
1. Override `equals` only for value classes; follow the five-clause contract and the five-step recipe.
2. `hashCode` travels with `equals`, always — use the `31*result + fieldHash` idiom or `Objects.hash`.
3. Make `toString` informative and document whether the format is stable.
4. Avoid `Cloneable`; use copy constructors/factories.
5. Implement `Comparable` for naturally ordered value types; build comparators, never subtract.
6. Inheritance + value components breaks `equals` symmetry/transitivity — compose instead.

## Connects To
- **Ch2 (Item 3)**: instance-controlled classes may skip `equals`.
- **Ch4 (Item 18)**: composition resolves the equals inheritance trap.
- **Ch4 (Item 17)**: immutable classes cache `hashCode` safely.
- **Ch11 (Item 78)**: `hashCode` consistency matters under concurrency.
