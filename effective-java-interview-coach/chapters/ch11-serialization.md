# Chapter 12: Serialization

## Item 85: Prefer alternatives to Java serialization

### Opening scenario
"Your service accepts a byte stream over RMI/JMS from another internal team and calls `ObjectInputStream.readObject()` on it. A pentest report comes back saying this endpoint allows remote code execution, even though your own classes are all written carefully and validate everything in their constructors. How is that possible, and what do you do about it?"

### Follow-up probes
- Why is the attack surface "all classes on the classpath that implement `Serializable`," not just your own classes?
- What's a gadget? What's a gadget chain? Can you name a real-world incident?
- Can an attacker cause damage without any gadget at all — what's a deserialization bomb, and why does it evade normal safeguards (small stream size, bounded stack depth)?
- If you can't eliminate serialization from a legacy system, what's the next-best mitigation, and what does it *not* protect against?
- Whitelisting vs. blacklisting for deserialization filters — which does Bloch recommend and why?

### Naive attempt
"We validate all our own `readObject` methods carefully, and none of our classes do anything dangerous in `readObject`, so we're safe."

### What breaks
The attack surface isn't limited to your own code. `readObject` is "essentially a magic constructor that can be made to instantiate objects of almost any type on the class path, so long as the type implements `Serializable`" (Item 85, line 5273). Attackers chain together `readObject`/deserialization side effects across platform libraries and third-party libraries (e.g., Apache Commons Collections) they never touch your code at all — this is exactly the mechanism behind the SFMTA Muni ransomware attack that shut down San Francisco's fare system for two days in 2016. Separately, even with zero gadgets, a 5,744-byte stream of 201 nested `HashSet`s can force ~2^100 `hashCode` invocations (a "deserialization bomb") — a DoS with no memory blowup and no deep stack to flag it as abnormal.

### The recommended approach & why
"The best way to avoid serialization exploits is never to deserialize anything" — for new systems, use a cross-platform structured-data representation (JSON or protobuf) instead of Java serialization; these support only simple attribute-value data, not arbitrary object-graph reconstruction, so there's no "magic constructor" attack surface. If you're stuck with legacy serialization: never deserialize untrusted data (never accept RMI traffic from untrusted sources), and if you must risk it, use `java.io.ObjectInputFilter` (Java 9+, backported) — prefer whitelisting (reject by default, accept a known-safe list) over blacklisting (accept by default, reject known-bad), because blacklisting only protects against already-discovered threats. Note this filter guards against excessive memory/graph depth but does *not* stop the hash-collision-style bomb above.

### Panel-ready answer checklist
- State the core claim precisely: `readObject` is a magic constructor invocable for any `Serializable` type on the classpath — the attack surface is the union of every such type, not just your code.
- Name gadgets / gadget chains and cite a concrete incident (SFMTA Muni, 2016).
- Distinguish RCE-via-gadget-chain from a deserialization bomb (DoS with no gadgets needed) — know why the bomb evades typical detection (small stream, bounded stack).
- Give the mitigation ladder in order: avoid serialization entirely (JSON/protobuf) → never deserialize untrusted data → `ObjectInputFilter` with whitelisting → accept that no filter defeats deserialization bombs.
- Don't say "serialization is fine if you code carefully" — the whole point is that your own care is insufficient because the surface includes code you don't control.

---

## Item 86: Implement Serializable with great caution

### Opening scenario
"Two years after shipping a `Serializable` class with the default serialized form, you want to swap its internal `LinkedList` for an array-backed structure for performance. QA blocks the release: old serialized instances from production now fail to deserialize against the new class. Why did a purely internal refactor become a compatibility break, and what did the original author fail to consider?"

### Follow-up probes
- Why does implementing `Serializable` "decrease the flexibility to change a class's implementation" even though serialization looks like an orthogonal, bolt-on feature?
- What is a serial version UID, how is it computed if you omit one, and what happens on deserialization if the recomputed value doesn't match?
- Why should classes designed for inheritance (Item 19) "rarely implement `Serializable`," yet `Throwable` and `Component` do — what's special about them?
- If a class designed for inheritance is stateful and extendable and serializable, why do you need `readObjectNoData`, and what corner case does it cover?
- Why can't inner (non-static member) classes safely implement `Serializable`?

### Naive attempt
"Serialization is basically free — just add `implements Serializable` to the class declaration. It's one line, so there's no real design cost."

### What breaks
Once a class implements `Serializable` and accepts the default form, "the class's private and package-private instance fields become part of its exported API" (line 5309) — Item 15's information-hiding discipline is defeated. Any later change to internal representation, or even something as innocuous as adding a convenience method, changes the auto-generated SHA-1-based serial version UID, and a mismatch throws `InvalidClassException` at runtime in production, between old and new versions of the class talking to each other. Additionally, since deserialization is an "extralinguistic" hidden constructor, it multiplies bug/security-hole risk (Item 88) and multiplies the testing burden by (serializable classes × releases).

### The recommended approach & why
Treat implementing `Serializable` as "a serious commitment that should be made with great care," on par with a permanent API decision. Always declare an explicit `serialVersionUID` (removes ambiguity and the runtime hashing cost). Weigh costs vs. benefits per class: value classes (`BigInteger`, `Instant`) and collections historically implement it; classes representing active entities (thread pools) should rarely implement it; classes designed for inheritance and interfaces should rarely implement/extend it, unless the class exists specifically to plug into a framework requiring `Serializable`. If a stateful extendable serializable class has invariants that would be violated by fields left at Java's zero/null defaults, add `readObjectNoData` to throw `InvalidObjectException` rather than silently producing an invalid instance.

### Panel-ready answer checklist
- Name the exact cost: serialized form becomes part of the exported API "forever," same as any other public API commitment.
- Explain serialVersionUID generation (SHA-1 over class structure) and the `InvalidClassException` failure mode when it's omitted and the class changes.
- State the inheritance-and-serialization rule and give both counter-example classes (`Throwable`, `Component`) with their actual justification (RMI exception propagation; GUI persistence).
- Mention `readObjectNoData` as the fix for the specific corner case of an existing serializable class gaining a new serializable superclass.
- Flag inner classes as categorically unsafe to serialize (unspecified synthetic field layout) — static member classes are fine.

---

## Item 87: Consider using a custom serialized form

### Opening scenario
"You've got a doubly linked list class, `StringList`, that implements `Serializable` with no custom `writeObject`/`readObject`. Someone reports that serializing a list of ~1,200 strings throws `StackOverflowError`, and separately that the serialized bytes are needlessly huge. Where do both problems come from, and how do you fix them without changing the public API?"

### Follow-up probes
- Why is the default serialized form "reasonable" for a class like `Name` but "awful" for `StringList` — what's the general rule for telling these apart?
- Enumerate all four disadvantages of accepting a bad default serialized form (API lock-in, space, time, stack overflow) — why does *recursive* graph traversal specifically cause the stack overflow?
- If all your fields are `transient`, why must `writeObject`/`readObject` still call `defaultWriteObject`/`defaultReadObject`? What breaks if you skip it?
- For a hash table, why would even a *correct-looking* default serialized form be "a serious bug"?
- What synchronization discipline must a `writeObject` method follow on a thread-safe class, and what deadlock risk does adding synchronization there introduce?

### Naive attempt
"Serialization is just about bytes on the wire — as long as `readObject` reconstructs an object that `equals()` the original, the internal representation used to get there doesn't matter."

### What breaks
For `StringList`, accepting the default form serializes every internal `Entry` node and both directions of its links, permanently locking `StringList.Entry` into the public API, bloating the byte stream (roughly 2x the size, and 2x the serialize time measured by Bloch), and — because default serialization recursively walks the object graph — causing genuine `StackOverflowError`s once the list reaches roughly 1,000–1,800 elements. For a hash table, it's worse than inefficient: bucket placement depends on `hashCode()`, which is "not, in general, guaranteed to be the same from implementation to implementation… or even from run to run" — so the default form can silently produce an object with corrupted invariants after a round trip.

### The recommended approach & why
Don't accept the default serialized form reflexively — "Accepting the default serialized form should be a conscious decision." Use it only when the physical representation already matches the logical content (Bloch's `Name` example: three strings, logically and physically). Otherwise, write a custom form: mark implementation-detail fields `transient`, and implement `writeObject`/`readObject` to emit/consume only the logical data (for `StringList`: size, then each string in order) — still calling `defaultWriteObject`/`defaultReadObject` first, because the serialization spec requires it and it's what preserves forward/backward compatibility when non-transient fields are added later. Every field that can be `transient` should be, including derived/cached fields and JVM-run-specific fields like native pointers. Always declare an explicit serial version UID regardless of which form you choose.

### Panel-ready answer checklist
- State the litmus test: does physical representation equal logical content? If not, write a custom form.
- List concretely all four costs of a bad default form: API lock-in, space, time, and recursive-traversal stack overflow.
- Explain why `defaultWriteObject`/`defaultReadObject` must be called even when every field is transient (compatibility for future non-transient fields; otherwise `StreamCorruptedException`).
- Cite the hash table example as "not just inefficient but a correctness bug," referencing hashCode's non-portability across runs/implementations.
- Know the synchronization rule: `writeObject` must follow the same synchronization discipline as any other method reading full object state, and matching lock-ordering to avoid resource-ordering deadlock.

---

## Item 88: Write readObject methods defensively

### Opening scenario
"Your `Period` class enforces `start ≤ end` in its constructor and defensively copies its `Date` fields — solid, by Item 50's playbook. You make it `Serializable` with the default form and no custom `readObject`. An attacker crafts a byte stream to instantiate a `Period` with `end` before `start`, bypassing your constructor's validation entirely. How is that possible, and once you add a validity check, why is the class *still* exploitable?"

### Follow-up probes
- Why is `readObject` "effectively another public constructor" — what does that imply about the obligations it inherits from Item 49 (validate arguments) and Item 50 (defensive copies)?
- Suppose you add a check `if (start.compareTo(end) > 0) throw new InvalidObjectException(...)` in `readObject`. Walk through why an attacker can still produce a *mutable* `Period` despite that check passing.
- What's the "rogue object reference" / stream-appending trick, and why does it let an attacker reach the private `Date` fields specifically?
- Ordering: why must defensive copying happen *before* the invariant check, not after?
- Why does defensive copying force `Period.start`/`end` to become non-final, and is that an acceptable tradeoff?
- Why must a `readObject` method never invoke an overridable method, directly or indirectly?

### Naive attempt
"I added a validity check in `readObject` that throws `InvalidObjectException` if `start` is after `end` — invariant enforced, done."

### What breaks
The check alone stops invalid *values* but not internal aliasing. An attacker builds a stream containing a valid `Period` instance followed by extra "previous object reference" bytes pointing back at the internal `Date` fields (the `MutablePeriod`/`ElvisStealer`-style attack, lines 5446–5466); after deserialization the attacker holds live references to the same `Date` objects `Period` uses internally, and mutating them (`pEnd.setYear(78)`) mutates the "immutable" `Period` after the fact — demonstrated output: `Wed Nov 22 ... 2017 - Wed Nov 22 ... 1978` then `... 1969`. The instance passed all invariant checks at construction time yet is fully mutable afterward, breaking any downstream code that depends on `Period`'s immutability for correctness or security.

### The recommended approach & why
Every serializable immutable class with private mutable components must defensively copy those components inside `readObject`, and the copy must happen *before* the invariant check, and must not use the component's own `clone` method (same reasoning as Item 50's constructor-level defensive copying). Litmus test Bloch gives: would you be comfortable adding a public constructor that takes every non-transient field as a raw parameter and stores it with zero validation? If not, `readObject` needs the same validation and copying discipline as a constructor. Also never call overridable methods from `readObject` — if overridden, the override runs before the subclass state is deserialized, likely causing failure.

### Panel-ready answer checklist
- Name the exact fix: `s.defaultReadObject()` → defensively copy every mutable-object-reference field → then check invariants, in that order, throwing `InvalidObjectException` on failure.
- Explain the rogue-reference attack conceptually: extra stream bytes let an attacker retain references to fields the constructor-based API never exposes.
- State the litmus test for whether a class needs a custom `readObject`: "would a raw public constructor with no validation be acceptable?"
- Note the fields must become non-final to support the copy — call this out as an explicit, known tradeoff, not an oversight.
- Mention the ban on invoking overridable methods from `readObject`, mirroring the same rule for constructors (Item 19).

---

## Item 89: For instance control, prefer enum types to readResolve

### Opening scenario
"Your `Elvis` singleton uses the classic private-constructor pattern from Item 3. Someone adds `implements Serializable` for a caching layer. Now two different parts of the app end up holding two distinct, non-`==` `Elvis` instances — with different data — even though you added a `readResolve` method that returns `INSTANCE`. How did the singleton guarantee break, given that `readResolve` looks correct?"

### Follow-up probes
- Why does merely adding `Serializable` break singleton-ness even with no custom `readObject` at all?
- Walk through exactly why `readResolve` "returning INSTANCE" is not sufficient by itself — what's the precondition on the instance fields that's being violated?
- Describe the "stealer" attack: why does the stealer's `readResolve` run *before* the singleton's, and why does that ordering matter?
- Why does Bloch say making `Elvis` a single-element enum is strictly better than fixing it by marking `favoriteSongs` transient?
- When is `readResolve` still legitimately needed even in modern code (i.e., when can't you just use an enum)?
- If a `readResolve` method is `protected` or `public` on a non-final class and a subclass doesn't override it, what breaks?

### Naive attempt
"Singleton class implements `Serializable`, I added a `private Object readResolve() { return INSTANCE; }` — instance control preserved."

### What breaks
`readResolve` only replaces the object *reference* returned to the caller after deserialization completes — but "if you depend on readResolve for instance control, all instance fields with object reference types must be declared transient." Because `Elvis.favoriteSongs` (a `String[]`) is non-transient, its contents are deserialized *before* `readResolve` runs. A crafted stream (`ElvisStealer`) hides inside that field with its own `readResolve`, which runs first (due to the circularity setup), copies the reference to the partially-deserialized, not-yet-resolved `Elvis` into a static field, and returns a dummy replacement — netting the attacker a second, live `Elvis` instance. Bloch's own demo produces two Elvis instances with different favorite songs (`[Hound Dog, Heartbreak Hotel]` vs `[A Fool Such as I]`), conclusively breaking the singleton invariant.

### The recommended approach & why
Prefer representing the instance-controlled, serializable class as a single-element enum type: `public enum Elvis { INSTANCE; ... }`. The JVM itself guarantees no other instances can exist (short of privileged reflection abuse via `AccessibleObject.setAccessible`, which implies the attacker already has arbitrary-code-execution-level access anyway) — this sidesteps the entire `readResolve`/field-transience minefield rather than trying to patch around it. `readResolve` is still the right tool only when instances aren't known at compile time (so an enum can't represent them) — in which case every instance field with an object reference type must be `transient`, with no exceptions.

### Panel-ready answer checklist
- State precisely why `readResolve` alone is insufficient: it fires only after fields are already deserialized, so non-transient object-reference fields are the leak.
- Walk through the stealer mechanism: circular reference makes the attacker's `readResolve` execute before the singleton's, letting it capture the not-yet-resolved instance.
- Give Bloch's fix hierarchy: enum type first (compile-time-known instances); `readResolve` + all-transient object fields only when instances aren't known at compile time.
- Note the `AccessibleObject.setAccessible` caveat as the sole (already-game-over) enum loophole.
- Know the accessibility rule for `readResolve` on non-final classes (private/package-private/protected/public each behave differently for subclasses) and the `ClassCastException` risk if a subclass fails to override a protected/public one.

---

## Item 90: Consider serialization proxies instead of serialized instances

### Opening scenario
"You've already added a defensive, validity-checking `readObject` to `Period` (Item 88's fix) — it works, but it forced `start`/`end` to become non-final, breaking true immutability (Item 17), and you're not fully confident you've enumerated every field an attacker could reach. Is there a pattern that gets you both real immutability and stronger security guarantees than a hand-written defensive `readObject`?"

### Follow-up probes
- Describe the four moving parts of the serialization proxy pattern precisely: the nested proxy class, `writeReplace`, the enclosing class's `readObject` override, and the proxy's `readResolve`.
- Why is the proxy's constructor allowed to skip validity checking and defensive copying, when `readObject` on the real class can't?
- Why must the enclosing class's `readObject` throw `InvalidObjectException("Proxy required")` rather than just being absent?
- What's the `EnumSet` example teaching — why can serialization proxies change the deserialized object's concrete class (`RegularEnumSet` → `JumboEnumSet`)?
- What are the two hard limitations of the pattern (extendable classes; circular object graphs)? Why does invoking a method on the not-yet-real object from inside `readResolve` throw `ClassCastException`?
- Bloch measured this pattern as ~14% more expensive to serialize/deserialize than defensive copying — when would you still choose it anyway?

### Naive attempt
"I'll just keep hardening `readObject` with more defensive copies and more invariant checks until I've covered every attack I can think of."

### What breaks
Ad hoc `readObject` hardening is reactive and open-ended — you must personally enumerate "which fields might be compromised by devious serialization attacks" and re-derive validity checks by hand, and it still can't restore `final`-ness (Item 88's fix required making `start`/`end` non-final, undermining Item 17's true immutability guarantee). Each new field or invariant reopens the analysis.

### The recommended approach & why
Use the serialization proxy pattern: a private static nested `SerializationProxy` class holding the enclosing class's logical state, with a constructor that just copies data (no checks needed — it's private, not attacker-facing). The enclosing class implements `writeReplace` to emit a `SerializationProxy` instead of itself, and a `readObject` that unconditionally throws `InvalidObjectException("Proxy required")` — so the "real" class is never the thing being deserialized directly. The proxy implements `readResolve`, which reconstructs the enclosing instance using its *ordinary public API* — e.g. `return new Period(start, end);` — so every invariant the public constructor already enforces is enforced here for free, and fields can stay `final`. As a bonus, because `readResolve` can return any compatible type, the deserialized concrete class doesn't have to match the one that was serialized — which is exactly how `EnumSet` deserializes a `RegularEnumSet` stream into a `JumboEnumSet` if the backing enum grew past 64 constants in between. Limitations: it doesn't work for classes extendable by clients (Item 19), and it breaks down for object graphs with circularities, since a method call on the not-yet-reconstructed object from within `readResolve` throws `ClassCastException`.

### Panel-ready answer checklist
- Name all four pieces correctly: nested proxy class, `writeReplace` on the enclosing class, guarding `readObject` that throws `InvalidObjectException("Proxy required")`, and `readResolve` on the proxy that calls the public constructor.
- Explain why the proxy's own constructor needs no validation/copying (it's a private implementation detail, not attacker-reachable) while the enclosing class's hypothetical `readObject` would.
- State the payoff precisely: invariants come for free because reconstruction goes through the real public constructor — fields stay `final`, true immutability is preserved.
- Cite the `EnumSet` `RegularEnumSet`/`JumboEnumSet` example as proof the pattern supports changing concrete class across the serialize/deserialize boundary.
- Know both limitations cold: incompatible with client-extendable classes; breaks under circular object graphs (`ClassCastException` from touching the not-yet-real object) — and know it costs roughly 14% more than defensive copying, per Bloch's own measurement.
