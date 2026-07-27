# Chapter 4: Classes and Interfaces

## Item 15: Minimize the accessibility of classes and members

### Opening scenario
"Here's a public class from your team's library. Every field is package-private, half the helper methods are public 'just in case someone needs them later.' A downstream team just filed a bug because they built on top of one of those 'just in case' public methods, and now you can't refactor it without breaking their build. Walk me through what's wrong with how this class was designed."

### Follow-up probes
- "Why not just make everything public and let convention/documentation tell people not to touch it?"
- "What's the actual difference in commitment between a package-private member and a protected one?"
- "A field is `final` and points to an immutable object, but it's `public`. Is that safe? Is it a good idea?"
- "You have a public `static final` array of constants. What's wrong with that, specifically?"
- "Does the Java 9 module system solve this problem for you automatically?"
- "Your subclass overrides a method and wants to tighten its visibility from `public` to `protected` for information hiding. Can it?"

### Naive attempt
"I'd just mark things private if I don't think anyone needs them, and public if I think someone might." No systematic rule — accessibility decided by guesswork about future need rather than by minimizing exposure now.

### What breaks
Every public class, interface, or member becomes a permanent commitment to the exported API — you must support it forever for compatibility. A public mutable array constant (`public static final Thing[] VALUES = {...}`) lets any client mutate shared internal state — "a frequent source of security holes." A gratuitously public top-level class can't later be deleted or replaced without breaking clients, even though it was never meant to be part of the API.

### The recommended approach & why
Rule of thumb: **make each class or member as inaccessible as possible.** For top-level classes/interfaces there are only two levels — package-private and public; default to package-private unless the type must be part of the exported API. If a package-private top-level class is used by only one class, consider making it a `private static` nested class of that one user (Item 24).

For members, four levels in increasing order of accessibility: `private` → package-private (default) → `protected` → `public`. After designing the public API, the reflex should be "make everything else private"; only loosen to package-private if another class in the same package genuinely needs it. `protected` is a big jump — it's part of the exported API and a public commitment to an implementation detail, so it should be rare.

A hard constraint: overriding methods cannot be *more* restrictive than the superclass method (Liskov substitution) — an interface method implementation must be `public`.

Public classes should have no public fields except `public static final` constants that are primitives or references to *immutable* objects. A nonzero-length array is always mutable, so never expose it directly as a constant — fix it with either an unmodifiable list wrapper (`Collections.unmodifiableList(Arrays.asList(PRIVATE_VALUES))`) or a defensive-copy accessor (`return PRIVATE_VALUES.clone();`).

Java 9 modules add two more (advisory) access levels but are "largely advisory" outside the JDK itself and not worth adopting without a compelling need.

### Panel-ready answer checklist
- States the rule of thumb: lowest access level consistent with correct function.
- Knows the four member access levels and their scope, in order.
- Cites the Liskov/overriding restriction on tightening access.
- Explains why public mutable fields break invariants, thread-safety, and future flexibility.
- Knows the two fixes for public array constants (unmodifiable wrapper vs. defensive-copy accessor).
- Correctly scopes the module system as advisory, not a general solution.

---

## Item 16: In public classes, use accessor methods, not public fields

### Opening scenario
"Here's a `Point` class with public `x` and `y` fields, used throughout a public library. Product now wants to switch the internal representation to polar coordinates for a performance win. What happens to every client that touches `point.x`?"

### Follow-up probes
- "Does your answer change if the class is package-private instead of public?"
- "What if the fields are `final`? Does that make direct field exposure fine?"
- "Name a real class in the JDK that got this wrong, and what it cost."
- "Is there a case where exposing fields directly on a public type is defensible?"

### Naive attempt
"Fields vs. getters is just style — accessors are boilerplate, and if the field is `final` there's no real risk." Treats accessor methods as a stylistic nicety rather than an encapsulation boundary.

### What breaks
Client code compiled against `point.x` is permanently coupled to that in-memory layout; you can never change representation without breaking binary/source compatibility. You lose the ability to enforce invariants (nothing stops `point.x = NaN`) and the ability to take action on access (logging, lazy computation, validation). The book cites `java.awt.Point` and `java.awt.Dimension` as "cautionary tales" — exposing `Dimension`'s internals directly "resulted in a serious performance problem that is still with us today" (Item 67 reference).

### The recommended approach & why
For a class accessible outside its package: **always use private fields + public accessor methods** (getters, and setters for mutable classes) to preserve the freedom to change internal representation later.

For a package-private class or a private nested class, direct field exposure is fine if the fields adequately describe the abstraction — the coupling is confined to code you also control, so a later representation change is a local, contained edit.

Immutable public fields (`public final int hour;`) are a middle ground: still "questionable" because you lose representation flexibility and the ability to act on reads, but you can still enforce invariants at construction time, and there's no thread-safety hazard from mutation.

### Panel-ready answer checklist
- Distinguishes public classes (always use accessors) from package-private/private nested classes (fields OK if they fit the abstraction).
- Explains what accessors buy you: representation flexibility, invariant enforcement, hook for side effects.
- Notes immutable public fields are "less harmful, though still questionable" — not a free pass.
- Can cite `java.awt.Point`/`Dimension` as the cautionary real-world example.

---

## Item 17: Minimize mutability

### Opening scenario
"You have a `BankAccount`-style class with a `Date lastModified` field and public getters/setters. QA reports that two threads reading and writing the same instance produce corrupted, inconsistent state — sometimes a balance changes underneath a computation that assumed it was stable. Also, someone stored an instance in a `HashSet`, mutated it after insertion, and now `set.contains(that same instance)` returns false. Diagnose what class of bug this is before we talk about fixes."

### Follow-up probes
- "Why not just synchronize every getter/setter instead of making the class immutable? What does that actually buy you, and what does it cost?"
- "What if the class needs a field like a `Date` or an array — those are inherently mutable. How do you keep the class immutable anyway?"
- "Doesn't creating a new object for every operation kill performance? Give me a concrete example where this hurts."
- "If you can't make the class immutable outright, what's the fallback design principle?"
- "You said 'make the class final to prevent subclassing' — is that the only way? What's the alternative, and why might it be *better*?"
- "Does immutability get you anything for free around partial-failure/exception scenarios?"

### Naive attempt
A mediocre candidate says: "Just make the setters `synchronized` so only one thread can mutate at a time," or "mark the fields `private` and add getters, that's encapsulation, that's enough." This treats thread-safety and encapsulation as solved by access control and locking, without addressing that the *state itself* can still change out from under any observer, aliasing, or concurrent reader — synchronization only serializes the corruption, it doesn't eliminate the shared mutable state that causes it.

### What breaks
With synchronized setters, you still have a mutable object with an arbitrarily complex state-transition space; a thread can read a half-updated view between two calls (compound actions aren't atomic just because individual setters are synchronized), you still can't safely put an instance in a `HashSet`/use it as a map key once mutated, you can't share instances across threads without discipline, and no method gets "failure atomicity for free" (Item 76) — a modification that throws partway through can leave the object in a corrupted state. Locking also costs throughput and adds boilerplate/deadlock risk that buys you nothing over just not having mutable state.

### The recommended approach & why
**The five rules to make a class immutable** (exact from the book):
1. Don't provide methods that modify the object's state (mutators).
2. Ensure the class can't be extended — normally via `final`, but there's a more flexible alternative (below).
3. Make all fields `final` — expresses intent, is enforced by the compiler, and is required for correct behavior when an instance reference is handed to another thread without synchronization (JLS memory model).
4. Make all fields `private` — prevents clients from getting direct access to mutable objects referenced by fields.
5. **Ensure exclusive access to any mutable components** — never initialize such a field from a client-supplied reference, never return the field itself from an accessor; make defensive copies (Item 50) in constructors, accessors, and `readObject`.

Rule 5 is the direct answer to "what about a `Date` or array field": you don't avoid the mutable component, you make sure no one but you ever holds a live reference to it — defensively copy on the way in and on the way out.

Worked example — `Complex` (verbatim structure from the book):
```java
// Immutable complex number class
public final class Complex {
    private final double re;
    private final double im;

    public Complex(double re, double im) {
        this.re = re;
        this.im = im;
    }

    public double realPart()      { return re; }
    public double imaginaryPart() { return im; }

    public Complex plus(Complex c) {
        return new Complex(re + c.re, im + c.im);
    }
    public Complex minus(Complex c) {
        return new Complex(re - c.re, im - c.im);
    }
    public Complex times(Complex c) {
        return new Complex(re * c.re - im * c.im,
                            re * c.im + im * c.re);
    }
    public Complex dividedBy(Complex c) {
        double tmp = c.re * c.re + c.im * c.im;
        return new Complex((re * c.re + im * c.im) / tmp,
                            (im * c.re - re * c.im) / tmp);
    }

    @Override public boolean equals(Object o) {
        if (o == this) return true;
        if (!(o instanceof Complex)) return false;
        Complex c = (Complex) o;
        return Double.compare(c.re, re) == 0
            && Double.compare(c.im, im) == 0;
    }
    @Override public int hashCode() {
        return 31 * Double.hashCode(re) + Double.hashCode(im);
    }
    @Override public String toString() {
        return "(" + re + " + " + im + "i)";
    }
}
```
Note the *functional* style: `plus`/`minus`/`times`/`dividedBy` return new instances rather than mutating `this`; method names are prepositions, not verbs (`plus`, not `add`) — signaling "this doesn't change the operand." (`BigInteger`/`BigDecimal` broke this naming convention and it caused real usage bugs.)

Why immutability pays off:
- **Simplicity** — one state for the object's whole lifetime; invariants established once at construction hold forever.
- **Inherent thread safety** — "immutable objects require no synchronization... they can be shared freely." No corruption possible under concurrent access, ever — this directly answers "why not just synchronize the setters": synchronization is a workaround for shared mutable state; immutability removes the shared mutable state.
- **Free sharing** — provide `public static final` constants for common values (`Complex.ZERO`, `ONE`, `I`), or static factories that cache instances (as `Integer`/`BigInteger` do), reducing allocation and GC pressure.
- **Never need defensive copies of them** — copies of an immutable object are "forever equivalent to the originals," so no `clone` method or copy constructor is needed (the `String` copy constructor is a historical mistake — "should rarely, if ever, be used," Item 6).
- **Can share internals** — `BigInteger.negate()` reuses the original's internal magnitude array; safe only because nothing can mutate it.
- **Great building blocks** — safe as map keys/set elements (their hash/equality can't shift underneath the collection) and as components of larger objects whose invariants you don't have to re-verify.
- **Failure atomicity for free** (Item 76) — state never changes, so there's no window for a half-finished mutation to be observed.

The disadvantage: **a distinct object per distinct value** — potentially costly for large objects undergoing many-step transformations. Book's example: flipping one bit of a million-bit `BigInteger` via `flipBit` allocates an entirely new million-bit object (time/space proportional to size), whereas mutable `BitSet.flip(0)` does it in constant time. Two mitigations: (1) guess the common multistep operations and provide them as primitives so intermediate objects are never materialized (`BigInteger`'s package-private mutable "companion class" for modular exponentiation); (2) if you can't predict the operations, provide a public mutable companion class (`String`'s companion is `StringBuilder`).

**Enforcing non-extensibility without `final`:** make all constructors `private`/package-private and expose `public static` factories instead (Item 1) — e.g., `Complex.valueOf(re, im)`. This is often the *better* alternative: it's effectively final to outside-package clients (can't subclass a type with no accessible constructor), but internally allows multiple package-private implementation classes and lets you improve caching/performance later without touching the public API. (Cautionary tale: `BigInteger`/`BigDecimal` were not built this way and are not effectively final — an untrusted subclass can violate immutability, forcing defensive code like `val.getClass() == BigInteger.class ? val : new BigInteger(val.toByteArray())` when security depends on real immutability.)

Relaxation allowed: fields may be non-final only to cache derived, expensive-to-recompute values (lazy initialization) — legal *because* the object is immutable, so recomputation would always yield the same answer (e.g., `PhoneNumber.hashCode()` caching its result).

Fallback when immutability is genuinely impractical: **minimize mutability** — make every field `private final` unless there's a compelling reason not to; reduce the number of representable states; never expose a public "reinitialize" method. `CountDownLatch` is the book's exemplar of "mutable, but its state space is kept intentionally small — use once, then done."

### Panel-ready answer checklist
- Recites all five rules of immutability, not just "make fields final."
- Directly rebuts "just synchronize the setters" — explains synchronization serializes access to mutable state, immutability eliminates the mutable state (and the corresponding thread-safety hazards) entirely.
- Handles the "what about a Date/array field" edge case with rule 5 — defensive copies in and out, never hand out or accept the live reference.
- Names the real cost: object-per-value allocation, with the `BigInteger.flipBit` vs `BitSet.flip` contrast, and the two mitigations (primitive multistep operations; public mutable companion class).
- Knows the "private constructors + static factories" alternative to `final` and why it's often preferred (effectively final externally, flexible internally).
- Can state the fallback principle when immutability isn't fully achievable: minimize state space, `private final` by default.

---

## Item 18: Favor composition over inheritance

### Opening scenario
"Your team extended `HashSet` to add an instrumented `InstrumentedHashSet` that counts every element ever inserted, by overriding `add` and `addAll`. In production, after calling `addAll` with 3 elements, `getAddCount()` reports 6, not 3. Nothing in your override is obviously wrong. What's going on, and whose fault is it?"

### Follow-up probes
- "Could you fix it by just removing your `addAll` override and letting the inherited one run?"
- "What if you instead re-implement `addAll` yourself, iterating and calling `add` for each element — does that fully solve it?"
- "Suppose you don't override anything you don't need to — you only *add* new methods, never override. Is that safe from all of this?"
- "The superclass ships a new method next release that also inserts elements, and your subclass doesn't know about it. What happens to any invariant your subclass was trying to enforce?"
- "What's the cost of switching to composition — do you have to hand-write every forwarding method?"
- "Give me the actual test for whether inheritance is appropriate here at all."

### Naive attempt
Mediocre candidate ships:
```java
// Broken - Inappropriate use of inheritance!
public class InstrumentedHashSet<E> extends HashSet<E> {
    private int addCount = 0;

    public InstrumentedHashSet() { }
    public InstrumentedHashSet(int initCap, float loadFactor) {
        super(initCap, loadFactor);
    }

    @Override public boolean add(E e) {
        addCount++;
        return super.add(e);
    }
    @Override public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }
    public int getAddCount() { return addCount; }
}
```
This "looks reasonable" and compiles cleanly — that's exactly why it's a good trap question.

### What breaks
`HashSet.addAll` is internally implemented on top of `add` (an undocumented implementation detail). Calling `s.addAll(List.of("Snap","Crackle","Pop"))` runs the overridden `addAll` (adds 3 to `addCount`, calls `super.addAll`), which internally invokes the overridden `add` once per element (adds 1 to `addCount` three more times) — total 6, double-counting every element. **Inheritance violates encapsulation**: the subclass's correctness now depends on an implementation detail of the superclass that isn't part of its contract and can change release to release. Removing the `addAll` override "fixes" this instance but the class is still fragile, now silently depending on the fact that `addAll` *is* implemented atop `add` — reimplementing `addAll` yourself (iterate + call `add` per element) avoids that specific coupling but is tedious, error-prone, may cost performance, and is sometimes outright impossible when a method needs access to private superclass state. Separately: even a subclass that *only adds new methods* and never overrides anything can still break — if the superclass later adds a method with the same signature but a different return type, the subclass fails to compile; same signature and return type silently turns your addition into an override, reintroducing the same fragility, except now against a contract that didn't exist when you wrote your method. Real-world precedent: `Hashtable` and `Vector` had actual security holes when retrofitted into the Collections Framework, because code relied on subclassing to enforce a predicate on inserted elements, and a new superclass insertion method bypassed the override entirely.

### The recommended approach & why
**Favor composition and forwarding.** Give the new class a private field referencing an instance of the existing class ("composition" — the existing class becomes a component); each new-class method invokes the corresponding method on the wrapped instance and returns the result ("forwarding methods"). Split into (a) a thin class holding your new behavior and (b) a reusable forwarding class implementing the interface purely by delegation:
```java
// Wrapper class - uses composition in place of inheritance
public class InstrumentedSet<E> extends ForwardingSet<E> {
    private int addCount = 0;

    public InstrumentedSet(Set<E> s) { super(s); }

    @Override public boolean add(E e) {
        addCount++;
        return super.add(e);
    }
    @Override public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }
    public int getAddCount() { return addCount; }
}

// Reusable forwarding class
public class ForwardingSet<E> implements Set<E> {
    private final Set<E> s;
    public ForwardingSet(Set<E> s) { this.s = s; }

    public void clear()               { s.clear();            }
    public boolean contains(Object o) { return s.contains(o); }
    public boolean isEmpty()          { return s.isEmpty();   }
    public int size()                 { return s.size();      }
    public Iterator<E> iterator()     { return s.iterator();  }
    public boolean add(E e)           { return s.add(e);      }
    public boolean remove(Object o)   { return s.remove(o);   }
    public boolean containsAll(Collection<?> c) { return s.containsAll(c); }
    public boolean addAll(Collection<? extends E> c) { return s.addAll(c); }
    public boolean removeAll(Collection<?> c) { return s.removeAll(c);   }
    public boolean retainAll(Collection<?> c) { return s.retainAll(c);   }
    public Object[] toArray()          { return s.toArray();  }
    public <T> T[] toArray(T[] a)      { return s.toArray(a); }
    @Override public boolean equals(Object o) { return s.equals(o);  }
    @Override public int hashCode()    { return s.hashCode(); }
    @Override public String toString() { return s.toString(); }
}
```
This is the **Decorator pattern** / **wrapper class** (also loosely called delegation, though technically delegation requires the wrapped object to be passed a self-reference back). Because `InstrumentedSet` depends only on the `Set` contract, it works with *any* `Set` implementation and any pre-existing constructor: `new InstrumentedSet<>(new TreeSet<>(cmp))`, `new InstrumentedSet<>(new HashSet<>(INIT_CAPACITY))`, or even wrapping an already-in-use instance temporarily. Adding new methods to the wrapped class later has zero impact — there's no self-use dependency to break.

**Costs of the wrapper approach**, addressed head-on: it's not suited to *callback frameworks* where wrapped objects hand out `this` to third parties — callbacks then bypass the wrapper (the "SELF problem"). Performance/memory overhead of forwarding calls is in practice negligible. Writing forwarding methods is tedious but a one-time cost per interface — reusable, and libraries like Guava already provide forwarding classes for all collection interfaces.

**The actual test for when inheritance is appropriate:** a genuine "is-a" relationship — ask "is every B really an A?" If you can't answer yes, B should contain a private A and expose a different API (A is an implementation detail, not an essential part of B). The book's canonical counter-examples: `Stack` is not a `Vector`, so `Stack` should not extend `Vector`; a property list is not a hash table, so `Properties` should not extend `Hashtable` — and the `Properties`/`Hashtable` mistake let clients bypass the string-only key/value invariant via direct `Hashtable` access, permanently, because clients came to depend on the violation. Using inheritance where composition fits also needlessly exposes implementation details, permanently caps your API's ability to change, and propagates any flaws in the superclass's API into yours — composition instead lets you design around those flaws. It is safe to use inheritance within a package under one team's control, or when extending a class specifically designed and documented for extension (Item 19) — the risk is specifically inheriting across package boundaries from an ordinary concrete class not designed for it.

### Panel-ready answer checklist
- Diagnoses the `InstrumentedHashSet` double-count bug as "inheritance violates encapsulation" via undocumented self-use (`addAll` implemented atop `add`), not a coding mistake.
- Shows why re-implementing `addAll` from scratch is only a partial fix — tedious, may be impossible without private access, still fragile to future superclass changes.
- Covers the "only add new methods, never override" trap — still breakable by a same-signature superclass addition in a later release.
- Produces the composition + forwarding class solution and names it (wrapper class / Decorator pattern), explaining why it has zero self-use coupling.
- States the SELF problem as the genuine limitation of wrappers (callback frameworks).
- Gives the is-a test with the `Stack`/`Vector` and `Properties`/`Hashtable` real JDK counter-examples.

---

## Item 19: Design and document for inheritance or else prohibit it

### Opening scenario
"A colleague subclasses your `Super` class. Their subclass has a `final` field set in its constructor. They call `new Sub()` and immediately print that field — and get `null` on the first call but the real value on the second call to the same method. Nobody touched multithreading. Explain how a final field gets observed in two different states from single-threaded code."

### Follow-up probes
- "Where exactly does the premature call happen, mechanically?"
- "What's the fix — should `Sub` just avoid calling `overrideMe` in its own constructor?"
- "Does this restriction apply to any other lifecycle-like methods besides constructors?"
- "You want to let subclasses hook into `clear()`'s internals for a performance win on sublists. What tool does the book give you for exactly that?"
- "If you don't know in advance what protected members subclasses will need, how do you find out, concretely?"
- "Given all of this cost, when should you just prohibit subclassing outright, and how, mechanically?"

### Naive attempt
"I'd just document what the class does (its public contract) and trust subclasses to override sensibly." Treats normal API documentation ("what it does") as sufficient, missing that safe inheritance requires documenting *how* — the self-use of overridable methods — which is a different, uncomfortable kind of disclosure.

### What breaks
The demonstrated failure:
```java
public class Super {
    // Broken - constructor invokes an overridable method
    public Super() { overrideMe(); }
    public void overrideMe() { }
}

public final class Sub extends Super {
    private final Instant instant;
    Sub() { instant = Instant.now(); }
    @Override public void overrideMe() { System.out.println(instant); }
    public static void main(String[] args) {
        Sub sub = new Sub();
        sub.overrideMe();
    }
}
```
`Super`'s constructor runs before `Sub`'s constructor body, but it invokes `overrideMe`, which is *overridden* in `Sub` — so the override runs before `instant` is assigned, printing `null`. The second, direct call to `overrideMe()` after full construction prints the real value — the same final field observed in two states. Had `overrideMe` dereferenced `instant`, it would have thrown `NullPointerException`. The same hazard applies to `clone` and `readObject` — both behave like constructors; an overriding method invoked from either can run before the subclass has finished establishing its own state, and for `clone` specifically, damage can leak into the *original* object, not just the copy.

### The recommended approach & why
To genuinely design a class for inheritance:
- **Document self-use precisely.** For every public/protected method, document which overridable methods it calls, in what order, and how each call's result affects what follows — via the `@implSpec` Javadoc tag (added Java 8), producing an "Implementation Requirements" section. Example cited verbatim from `AbstractCollection.remove`: the spec states plainly that the implementation iterates via the collection's iterator and calls the iterator's `remove`, and that it throws `UnsupportedOperationException` if that iterator doesn't support `remove`. This is a deliberate exception to "document what, not how" — inheritance itself forces implementation detail into the contract.
- **Expose judiciously chosen `protected` hooks** where needed for efficient subclassing — e.g., `AbstractList.removeRange(fromIndex, toIndex)` exists purely so subclasses can give `clear()` on sublists linear instead of quadratic behavior; it's "of no interest to end users," solely an extension point.
- **Test by writing subclasses — the only real test.** "Three subclasses are usually sufficient," and at least one should be written by someone other than the superclass author, because a missing protected member becomes painfully obvious only when you actually try to extend the class, and an unused one is a signal to make it private.
- **Never invoke an overridable method from a constructor, `clone`, or `readObject`,** directly or indirectly — the concrete failure mode above.
- **`Cloneable`/`Serializable` on an inheritance-designed class are extra hazards** — generally avoid implementing them on such classes; if you must, remember `clone`/`readObject` carry the same "no overridable method calls" restriction as constructors, and any `readResolve`/`writeReplace` must be `protected`, not `private`, or subclasses silently lose them.
- **Recognize this is a heavy, permanent commitment** — designing for inheritance locks in self-use patterns and protected-member implementation choices "forever." It's clearly right for abstract classes/skeletal implementations (Item 20); clearly wrong for immutable classes (Item 17).
- **For ordinary concrete classes, the safer default is to prohibit subclassing** — either declare the class `final`, or make all constructors private/package-private with public static factories instead (Item 17's alternative, which still allows internal subclassing). If a class implements a meaningful interface (`Set`, `List`, `Map`), prohibiting subclassing costs nothing — use the Item 18 wrapper pattern to add functionality instead. If it doesn't implement such an interface and inheritance must be allowed, mechanically eliminate all self-use of overridable methods first: move each overridable method's body into a private helper, have the overridable method just call the helper, and have all other self-use call the helper directly — so overriding a method can never again change another method's behavior.

### Panel-ready answer checklist
- Traces the `null`-then-real-value bug to "superclass constructor runs first and calls an overridden method before the subclass constructor has initialized its state."
- Extends the same restriction to `clone`/`readObject`, noting `clone` can corrupt the *original* object.
- Names `@implSpec` and describes what it discloses (self-use of overridable methods) and why that appears to violate normal API-doc practice.
- Cites `AbstractList.removeRange` as the canonical "protected hook that exists only to help subclasses" example.
- States the empirical test for design completeness: write ~3 subclasses, at least one by an outside author.
- Gives the two concrete mechanisms to prohibit subclassing: `final`, or private/package-private constructors + static factories.

---

## Item 20: Prefer interfaces to abstract classes

### Opening scenario
"You need a type that both `Singer` and `Songwriter` classes can implement, and some classes are genuinely both. Someone proposes an abstract base class `Performer` with default behavior for both roles. What goes wrong the moment a class needs to be both a `Singer` and a `Songwriter`?"

### Follow-up probes
- "Since Java 8 lets abstract classes and interfaces both have method bodies, what's actually left as the distinguishing constraint?"
- "How do you retrofit an existing, already-shipped class to support a new interface, versus a new abstract superclass?"
- "If two interface methods overlap in required boilerplate, how do you avoid making every implementor duplicate that logic?"
- "What can't you put in an interface at all, even with default methods?"
- "What's this pattern called where you provide an abstract class alongside an interface to make it easier to implement, and can you sketch how it applies to `Map.Entry`?"

### Naive attempt
"Abstract classes and interfaces are basically interchangeable now that interfaces can have default methods — pick whichever." Ignores the single-inheritance constraint, which is the actual reason interfaces remain structurally superior for defining a type.

### What breaks
Because Java permits only single inheritance, requiring implementors to extend an abstract class caps them at one supertype forever — `Performer extends nothing else` blocks any class that's also, say, a `Comparable`-implementing subtype of some other hierarchy, and forces you to place the abstract class impossibly high in the hierarchy to cover every combination, causing "collateral damage" by dragging in unrelated descendants. With `n` independent optional behaviors, an abstract-class-per-combination design needs up to 2ⁿ classes — a *combinatorial explosion* — versus interfaces, which any class can `implements` in any combination (`SingerSongwriter extends Singer, Songwriter`). Existing classes also can't be retrofitted to extend a new abstract class without moving it to a common ancestor; they *can* trivially be retrofitted to `implements` a new interface (this is literally how `Comparable`, `Iterable`, and `AutoCloseable` were added to pre-existing JDK classes).

### The recommended approach & why
Prefer interfaces for defining types that permit multiple implementations. Interfaces enable:
- **Mixins** — a type that can be "mixed in" to declare optional behavior alongside a class's primary type (`Comparable` is the canonical mixin) — structurally impossible for an abstract class, since a class can have only one parent.
- **Nonhierarchical type frameworks** — `SingerSongwriter extends Singer, Songwriter` composes cleanly; an abstract-class equivalent would force a rigid, exploding hierarchy.
- **Safe functionality enhancement via the wrapper idiom (Item 18)** — abstract-class-typed designs leave programmers only inheritance to extend behavior, which is more fragile than a wrapper class.
- **Default methods** for obvious implementations built on other interface methods (documented via `@implSpec`, Item 19) — but you cannot provide default implementations for `Object` methods (`equals`, `hashCode`, `toString`), interfaces can't hold instance fields or non-public static members (except private static methods), and you can't add default methods to an interface you don't control.

**Best of both worlds — the skeletal implementation (Template Method pattern):** the interface defines the type (with default methods where possible); a companion abstract class (conventionally named `Abstract<Interface>`, e.g. `AbstractCollection`, `AbstractSet`, `AbstractList`, `AbstractMap`) implements the remaining non-primitive methods atop the primitive ones — including the `Object`-method overrides that default methods can't provide. Worked example: `AbstractMapEntry` implements `equals`/`hashCode`/`toString` for `Map.Entry` in terms of the primitives `getKey`/`getValue`:
```java
public abstract class AbstractMapEntry<K,V> implements Map.Entry<K,V> {
    @Override public V setValue(V value) {
        throw new UnsupportedOperationException();
    }
    @Override public boolean equals(Object o) {
        if (o == this) return true;
        if (!(o instanceof Map.Entry)) return false;
        Map.Entry<?,?> e = (Map.Entry) o;
        return Objects.equals(e.getKey(), getKey())
            && Objects.equals(e.getValue(), getValue());
    }
    @Override public int hashCode() {
        return Objects.hashCode(getKey()) ^ Objects.hashCode(getValue());
    }
    @Override public String toString() {
        return getKey() + "=" + getValue();
    }
}
```
A class that can't extend the skeletal class (because it already extends something else) can still implement the interface directly, or use **simulated multiple inheritance**: forward interface calls to a private inner class that extends the skeletal implementation. Extending the skeletal class is "the obvious choice... but it is strictly optional." Skeletal implementations, since they're designed for inheritance, must follow all of Item 19's documentation/design guidelines. A related, non-abstract variant is a *simple implementation* (e.g. `AbstractMap.SimpleEntry`) — the simplest possible working implementation, usable as-is or subclassed.

### Panel-ready answer checklist
- Identifies single inheritance as the concrete reason abstract classes can't serve as flexible type definitions (mixins, combinatorial explosion, retrofitting).
- Notes what default methods still can't do: `Object`-method overrides, instance fields, non-public statics (except private static methods), or editing interfaces you don't own.
- Names and sketches the skeletal implementation / Template Method pattern with the `Abstract*` naming convention.
- Can produce the `Map.Entry`/`AbstractMapEntry` example, explaining specifically why `equals`/`hashCode` had to live in the abstract class and not as interface default methods.
- Mentions simulated multiple inheritance as the escape hatch when a class can't extend the skeletal class.

---

## Item 21: Design interfaces for posterity

### Opening scenario
"Your team ships `Collection.removeIf` as a Java 8 default method. A third-party library's `SynchronizedCollection` — which wraps every method with a lock on a client-supplied object — doesn't override it. It compiles fine against the new interface. In production, under concurrent modification, it throws `ConcurrentModificationException` or behaves unpredictably. Nobody touched that library's code. Explain the failure."

### Follow-up probes
- "Whose bug is this — the JDK's, for adding the method, or the library's, for not updating?"
- "Could the default `removeIf` implementation have been written to just work for every possible implementation?"
- "Does the same risk apply if you add an *abstract* method to an interface instead of a default one?"
- "If default methods are risky to retrofit, why do they exist at all — what were they actually for?"
- "How would you validate a brand-new interface before shipping it, to avoid ever being in this position?"

### Naive attempt
"Default methods let you add methods to interfaces safely without breaking anyone — that's the whole point of them." Treats default methods as a compatibility guarantee rather than a best-effort general implementation that can silently violate an implementor's invariants.

### What breaks
The default `removeIf` is literally:
```java
default boolean removeIf(Predicate<? super E> filter) {
    Objects.requireNonNull(filter);
    boolean result = false;
    for (Iterator<E> it = iterator(); it.hasNext(); ) {
        if (filter.test(it.next())) {
            it.remove();
            result = true;
        }
    }
    return result;
}
```
It "knows nothing about synchronization and has no access to the field that contains the locking object" — so it can't uphold `SynchronizedCollection`'s fundamental promise to synchronize around every call. Default methods are "injected" into pre-existing implementations without their author's knowledge or consent — those implementations were written under the pre-Java-8 assumption that their interface would never gain new methods. The JDK's own `Collections.synchronizedCollection` had to be manually patched to override `removeIf` and similar methods to add the missing synchronization before delegating to the default logic; third-party implementations that didn't get an equivalent patch stay silently broken — "existing implementations of an interface may compile without error or warning but fail at runtime."

### The recommended approach & why
Adding default methods to *existing* interfaces should be avoided unless the need is critical, and even then only after seriously considering whether it could break existing implementations — default methods can't be written to preserve every implementation's invariants because they have no visibility into implementation-specific state (like a lock object). Default methods remain genuinely valuable for *new* interfaces, to supply standard implementations that ease the burden of implementing them (Item 20) — the risk is specifically retroactive injection into interfaces with an established implementor population. Also: default methods were never designed to support removing methods or changing existing signatures — neither is possible without breaking clients regardless of default methods.

Since interface flaws are costly or impossible to fix post-release, **test rigorously before shipping**: have multiple programmers implement each new interface in genuinely different ways (aim for at least three diverse implementations), and write multiple client programs exercising each implementation for varied tasks — this surfaces missing/wrong methods and self-use assumptions while they're still cheap to fix.

### Panel-ready answer checklist
- Explains default methods as "best-effort generic implementation," not a compatibility guarantee against invariant violation.
- Walks through why `removeIf`'s default breaks `SynchronizedCollection` specifically (no access to the lock object).
- Notes the JDK had to hand-patch its own `synchronizedCollection` — a concrete case where the "safe" mechanism still required manual intervention.
- States that default methods can't support removing methods or changing signatures at all.
- Gives the pre-release testing prescription: ~3 diverse implementations, multiple client programs.

---

## Item 22: Use interfaces only to define types

### Opening scenario
"You find an interface with zero methods — just a pile of `static final` double constants for physical constants — and half the classes in the codebase `implements` it purely so they can write `AVOGADROS_NUMBER` instead of `PhysicalConstants.AVOGADROS_NUMBER`. What's your objection to this, beyond 'it's a bit odd'?"

### Follow-up probes
- "What actually leaks into the public API because of this pattern?"
- "What's the concrete consequence if, in a later release, the class no longer needs those constants internally?"
- "What if the class implementing the constant interface is non-final — does that make it worse?"
- "Give me the three legitimate places to put constants instead, and when you'd pick each."
- "How do you avoid the `ClassName.CONSTANT` qualification without resorting to a constant interface?"

### Naive attempt
"Interfaces are just a namespace for shared constants, so implementing one to inherit its constants is a normal use of interfaces." Confuses "type a client can rely on" with "convenient constant bag" — an interface a class implements is supposed to say something about what clients can *do* with instances of that class, and a constant interface says nothing.

### What breaks
That a class uses certain constants internally is purely an implementation detail; implementing the constant interface leaks that detail into the exported API and can confuse API users about the class's actual type contract. Worse, it's a permanent commitment: if a future release stops needing the constants, the class must still `implements` the interface forever to preserve binary compatibility. If the class is non-final, every subclass's namespace is now polluted with those constants too — an unremovable, inherited side effect having nothing to do with the subclass's own purpose.

### The recommended approach & why
Interfaces should be used only to define types. For exporting constants, choose based on ownership:
- **Strongly tied to an existing class/interface** — add them directly to that type (e.g., `Integer.MIN_VALUE`/`MAX_VALUE`).
- **Naturally an enumerated type** — export via an `enum` (Item 34).
- **Otherwise** — a noninstantiable utility class (Item 4):
```java
// Constant utility class
package com.effectivejava.science;
public class PhysicalConstants {
    private PhysicalConstants() { }  // Prevents instantiation
    public static final double AVOGADROS_NUMBER = 6.022_140_857e23;
    public static final double BOLTZMANN_CONST  = 1.380_648_52e-23;
    public static final double ELECTRON_MASS    = 9.109_383_56e-31;
}
```
To avoid the `PhysicalConstants.AVOGADROS_NUMBER` qualification when a class makes heavy use of it, use **static import**:
```java
import static com.effectivejava.science.PhysicalConstants.*;
public class Test {
    double atoms(double mols) { return AVOGADROS_NUMBER * mols; }
}
```
This gets you the ergonomic win the constant-interface antipattern was chasing, without polluting any class's type contract or subclass namespace.

### Panel-ready answer checklist
- States the core principle: an implemented interface should say something about what clients can do with the type; constants say nothing.
- Names the concrete cost: permanent binary-compatibility commitment even after the constants are no longer needed, and namespace pollution down a non-final class's subclass tree.
- Gives all three legitimate homes for constants (existing type, enum, noninstantiable utility class) matched to when each applies.
- Offers static import as the ergonomic alternative to constant interfaces.

---

## Item 23: Prefer class hierarchies to tagged classes

### Opening scenario
"Here's a `Figure` class with an enum tag field `Shape { RECTANGLE, CIRCLE }`, fields for both a rectangle's `length`/`width` and a circle's `radius` sitting side by side, and an `area()` method that switches on the tag. A new `TRIANGLE` case needs to be added. What has to change, and what's your worry about doing it safely?"

### Follow-up probes
- "Can any of those fields be made `final`?"
- "What happens at runtime if someone adds the new enum value but forgets to add the switch case?"
- "Does the type system give you any help distinguishing a circle-flavored `Figure` from a rectangle-flavored one at compile time?"
- "How would you retrofit this into a class hierarchy, concretely — what goes in the root, what goes in each subclass?"
- "Suppose you now also need `Square`. How does the hierarchy model that a square is a special rectangle?"

### Naive attempt
"Just add `TRIANGLE` to the enum, add three new fields for triangle-specific data, and add a `case TRIANGLE:` branch to every switch — the pattern already works, just extend it." Treats the tagged-class shape as a stable, extensible design rather than recognizing every one of these edits as a fresh opportunity for a silent, uncaught runtime failure.

### What breaks
Tagged classes are "verbose, error-prone, and inefficient": boilerplate (enum, tag field, switches) obscures readability by jumbling multiple flavors' implementations into one class body; every instance carries the irrelevant fields of every *other* flavor (wasted memory); fields generally can't be `final` unless constructors initialize the irrelevant ones too (more boilerplate); constructors must set the tag and the right fields "with no help from the compiler" — get it wrong and it fails at runtime, not compile time; adding a flavor requires source access plus remembering to update every switch statement, and a missed case is a runtime `AssertionError` or silent bug, never a compile error; and the type of an instance ("`Figure`") gives no clue at all which flavor it actually is.

### The recommended approach & why
Convert the tag into **subtyping**. Define an abstract root class with an abstract method per tag-dependent behavior (here, just `area()`); put any flavor-independent fields/methods in the root. Then one concrete subclass per flavor, holding only that flavor's fields:
```java
abstract class Figure {
    abstract double area();
}
class Circle extends Figure {
    final double radius;
    Circle(double radius) { this.radius = radius; }
    @Override double area() { return Math.PI * (radius * radius); }
}
class Rectangle extends Figure {
    final double length;
    final double width;
    Rectangle(double length, double width) {
        this.length = length;
        this.width  = width;
    }
    @Override double area() { return length * width; }
}
```
This fixes every listed shortcoming: no boilerplate tag machinery, each flavor's own class carries only its own fields (all `final`), the compiler enforces that every concrete subclass implements every abstract method (no missing-switch-case runtime failure is even possible), independent teams can extend the hierarchy without touching the root's source, and each flavor is now a distinct, checkable type. Hierarchies also model **real subtype relationships** the tagged version couldn't express at all — adding `Square` as a special `Rectangle`:
```java
class Square extends Rectangle {
    Square(double side) { super(side, side); }
}
```
(Fields accessed directly here for brevity — poor practice if the hierarchy were public; see Item 16.)

### Panel-ready answer checklist
- Lists the concrete tagged-class costs: boilerplate, wasted memory per instance, non-final fields, compiler-unchecked constructors, missing-switch-case runtime failures, no type-level flavor distinction.
- Produces the abstract-root + one-concrete-subclass-per-flavor refactor.
- Explains why the compiler now catches what used to be a runtime `AssertionError`.
- Extends the hierarchy to model a genuine is-a relationship (`Square extends Rectangle`) that a tag field never could.

---

## Item 24: Favor static member classes over nonstatic

### Opening scenario
"A `Map` implementation's internal `Entry` class was accidentally declared as a nonstatic inner class instead of `static`. The map still works correctly in every test. Months later, someone notices instances of the enclosing map are never getting garbage collected even after the map itself is unreachable everywhere in application code except leftover `Entry` objects. Explain how a working, functionally-correct class became a memory leak."

### Follow-up probes
- "What specifically does a nonstatic member class instance hold onto that a static one doesn't?"
- "When would you actually *want* the nonstatic version — give a real JDK example."
- "You have four kinds of nested classes on the table — static member, nonstatic member, anonymous, local. Give me the decision rule for each."
- "If this were a `public` member class of an exported class, does the static/nonstatic choice still matter as much later?"
- "What can't an anonymous class do that a local or member class can?"

### Naive attempt
"It's a style choice — `static` or not doesn't change behavior since the class still works and passes tests." Treats the modifier as cosmetic, missing that it changes what every instance secretly carries and how long the enclosing object can be reachable.

### What breaks
Each nonstatic member class instance implicitly holds a reference to an enclosing instance, established permanently at creation time (normally by instantiating the member class from within an instance method of the enclosing class, or manually via `enclosingInstance.new MemberClass(args)`). This reference "takes up space... and adds time to construction," and — the serious part — "it can result in the enclosing instance being retained when it would otherwise be eligible for garbage collection... The resulting memory leak can be catastrophic. It is often difficult to detect because the reference is invisible." An `Entry` object doesn't need the enclosing map at all for `getKey`/`getValue`/`setValue` — so an accidentally-nonstatic `Entry` correctly implements every visible behavior while silently pinning its map in memory for as long as any entry survives.

### The recommended approach & why
**If a nested class doesn't need a reference to an enclosing instance, always declare it `static`.** Reserve nonstatic member classes specifically for cases where each instance genuinely needs to invoke methods on, or obtain a reference to, its enclosing instance (via qualified `this`) — a nonstatic member class instance cannot even exist without an enclosing instance. Real use case: implementations of `Map`'s `keySet`/`entrySet`/`values` views, and collection iterators (`Set`/`List` implementations), are typically nonstatic member classes precisely because they need to operate against their enclosing collection:
```java
public class MySet<E> extends AbstractSet<E> {
    @Override public Iterator<E> iterator() { return new MyIterator(); }
    private class MyIterator implements Iterator<E> { ... }
}
```
Contrast: a `Map.Entry`-style component that never touches the enclosing map should be a `private static` member class. This matters even more for exported API — if a member class is `public`/`protected` on an exported class, static vs. nonstatic is now a binary-compatibility commitment you can't reverse later.

**The other two kinds, for completeness:**
- **Anonymous classes** — no name, declared and instantiated simultaneously at point of use; have an enclosing instance only in a nonstatic context; can't have static members besides constant variables; can't be `instanceof`-tested by name, can't implement two interfaces or extend-and-implement simultaneously, callers see only members inherited from the declared supertype; must stay short (~10 lines) for readability. Largely superseded by lambdas (Item 42) for function objects, but still used for one-off static factory bodies (e.g. `intArrayAsList` in Item 20).
- **Local classes** — least-used kind; declared anywhere a local variable could be, same scoping rules; like member classes they have names and can be reused; like anonymous classes, they get an enclosing instance only in nonstatic contexts and can't hold static members; should also stay short.

Decision rule: if the nested class needs visibility beyond one method, or is too long for a method body, make it a member class — nonstatic only if instances need the enclosing-instance link, otherwise static. Within a method, if you need instances from only one call site and a preexisting type already characterizes it, use anonymous; otherwise use local.

### Panel-ready answer checklist
- Explains the hidden enclosing-instance reference as the concrete mechanism of the memory leak, and why it's hard to detect ("the reference is invisible").
- Gives the rule: static unless instances genuinely need the enclosing instance.
- Cites a real justified nonstatic use (`Map` view classes / collection iterators referencing their enclosing collection).
- Notes the binary-compatibility stakes when the choice is made on a public/protected member of an exported class.
- Can distinguish all four nested-class kinds and give the selection rule between anonymous and local.

---

## Item 25: Limit source files to a single top-level class

### Opening scenario
"A file `Utensil.java` defines both `class Utensil` and `class Dessert`. Later, someone accidentally creates `Dessert.java` that *also* defines both `Utensil` and `Dessert`, with different constant values. Running `javac Main.java Utensil.java` prints `pancake`; running `javac Dessert.java Main.java` prints `potpie`. Nobody changed any source file between these two runs — explain why the program's output depends on the compiler command line."

### Follow-up probes
- "Why does `javac Main.java Dessert.java` fail to compile at all instead of picking one silently?"
- "Is this a compiler bug, or expected behavior given the language rules?"
- "What's your fix if `Utensil` and `Dessert` really do belong conceptually close together?"
- "Does splitting into separate files have any downside worth mentioning?"
- "Would a build tool like Maven/Gradle mask this problem for you?"

### Naive attempt
"As long as the code compiles and passes tests today, having a couple of small helper classes share a file is harmless — it's just an organizational preference." Misses that the danger isn't stylistic, it's a genuine correctness hazard triggered purely by compiler invocation order.

### What breaks
Defining multiple top-level classes in one file makes it possible for the same class name to have multiple definitions across files. `Main.java` references `Utensil` (encountered first) and `Dessert`; the compiler, on seeing the `Utensil` reference, opens `Utensil.java` and pulls in *both* class definitions found there. If `Dessert.java` is later also passed on the command line, the compiler now has two definitions of `Utensil` and two of `Dessert` — `javac Main.java Dessert.java` fails with a "multiply defined" error because compiling `Main.java` first pulls in `Utensil.java`'s copies via the `Utensil` reference, then `Dessert.java` on the command line collides with them. But `javac Main.java` or `javac Main.java Utensil.java` compiles and prints `pancake`, while `javac Dessert.java Main.java` compiles cleanly and prints `potpie` — same source tree, different, silently different output, "clearly unacceptable," purely as a function of argument order.

### The recommended approach & why
**Never put multiple top-level classes or interfaces in a single source file.** This alone guarantees a class can't have multiple definitions at compile time, which in turn guarantees the compiled behavior is independent of compilation order. If the classes are conceptually subservient to one class (as `Utensil`/`Dessert` are to the program using them), prefer making them **`private static` member classes (Item 24)** of the class that uses them, rather than splitting into more top-level files — this improves readability (keeps related code together) and lets you shrink their accessibility to `private`, which separate top-level files can't do:
```java
// Static member classes instead of multiple top-level classes
public class Test {
    public static void main(String[] args) {
        System.out.println(Utensil.NAME + Dessert.NAME);
    }
    private static class Utensil { static final String NAME = "pan"; }
    private static class Dessert { static final String NAME = "cake"; }
}
```

### Panel-ready answer checklist
- Traces exactly how compilation order determines which `Utensil`/`Dessert` definitions win, and why one particular ordering fails to compile at all.
- States the rule: one top-level class/interface per source file, no exceptions.
- Prefers `private static` member classes (Item 24) over separate files when classes are genuinely subservient to one owner — names both the readability and accessibility benefits.
- Notes this guarantees compiled behavior is independent of `javac` argument order.
