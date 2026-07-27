# Chapter 12: Serialization (Items 85–90)

## Core Idea
Java's built-in serialization is a persistent security and maintenance liability — deserialization is an attack surface that runs code on hostile bytes. Avoid it where possible; when unavoidable, defend aggressively with custom forms, defensive `readObject`, and serialization proxies.

## Frameworks Introduced
- **Item 85 — Prefer alternatives to Java serialization**: the fundamental problem is that `readObject` is a "magic constructor" that can instantiate almost any type — enabling gadget-chain attacks and deserialization bombs. Prefer cross-platform structured data formats (JSON, Protocol Buffers). If you can't avoid Java serialization, never deserialize untrusted data; use `ObjectInputFilter` (JEP 290) to whitelist classes.
- **Item 86 — Implement `Serializable` with great caution**: implementing it is a lasting commitment — it locks the internal representation into your public API, increases bug/security surface, and complicates evolution. Not to be done lightly; avoid for classes designed for inheritance and for inner classes.
- **Item 87 — Consider using a custom serialized form**: the default form is only acceptable when the physical representation matches the logical content. Otherwise define a custom form: mark fields `transient`, implement `writeObject`/`readObject` that call `defaultWriteObject`/`defaultReadObject` then handle logical state; declare an explicit `serialVersionUID`.
- **Item 88 — Write `readObject` methods defensively**: `readObject` is effectively another public constructor fed arbitrary bytes — validate invariants and make **defensive copies of mutable components** (into `final` fields where possible). Don't call overridable methods from `readObject`.
- **Item 89 — For instance control, prefer enum types to `readResolve`**: singletons that implement `Serializable` can be duplicated on deserialization; `readResolve` can fix it but requires all fields be `transient` (else a stolen-reference attack). A single-element **enum** provides ironclad, attack-proof instance control for free.
- **Item 90 — Consider serialization proxies instead of serialized instances**: the **serialization proxy pattern** — a private static nested class capturing the logical state, with `writeReplace` on the enclosing class returning the proxy and `readResolve` on the proxy reconstructing via the normal (public) constructor. Add `readObject` on the enclosing class that always throws, blocking attackers. Greatly reduces `readObject` risk; doesn't work for extendable classes or objects with circular references.

## Key Concepts
- **Serialization / deserialization**: converting objects to/from a byte stream; `readObject` reconstructs without invoking a normal constructor.
- **Gadget chain / deserialization bomb**: crafted byte streams that execute code or exhaust resources during deserialization.
- **`serialVersionUID`**: version identifier; declare it explicitly to control compatibility.
- **`transient`**: field excluded from the default serialized form.
- **`writeReplace` / `readResolve`**: hooks to substitute a different object on write/read (basis of the proxy pattern).
- **Serialization proxy**: a compact stand-in class representing the logical state of an instance.

## Mental Models
- Treat `readObject` as a *public constructor accepting attacker-controlled input* — validate and defensively copy exactly as you would there (and more).
- The default serialized form is a *contract*: it freezes your private fields into your API. Only accept it when physical == logical representation.
- Prefer *not* implementing `Serializable`; each class that does is permanent attack + maintenance surface.
- When you must serialize a class with invariants, the serialization proxy pattern is the safest default.

## Anti-patterns
- **Deserializing untrusted data** with Java serialization: remote code execution risk — "there is no reason to use it in any new system."
- **Default serialized form for a logical structure** (e.g. a linked list serialized as raw node links): brittle, bloated, ties API to representation.
- **Omitting `serialVersionUID`**: auto-generated UID changes with any class edit → breaks compatibility unexpectedly.
- **`readObject` without validation/defensive copies**: attacker forges invalid or aliased internals (the mutable-`Date` attack, again, but via bytes).
- **`readResolve` singleton with non-`transient` reference field**: stolen-reference attack.
- **Calling overridable methods from `readObject`**: subclass runs before deserialization completes.

## Code Examples
```java
// Item 90 — serialization proxy pattern
private static class SerializationProxy implements Serializable {
    private final Date start, end;
    SerializationProxy(Period p){ this.start = p.start; this.end = p.end; }
    private Object readResolve(){ return new Period(start, end); } // uses PUBLIC ctor → invariants enforced
    private static final long serialVersionUID = 234098243823485285L;
}
private Object writeReplace(){ return new SerializationProxy(this); } // never serialize the real instance
private void readObject(ObjectInputStream s) throws InvalidObjectException {
    throw new InvalidObjectException("Proxy required"); // block direct-byte attacks
}
```
- **What it demonstrates**: deserialization routes through the normal constructor via the proxy, so all invariants/defensive copies apply automatically, and forged streams are rejected.

```java
// Item 88 — defensive readObject when NOT using a proxy
private void readObject(ObjectInputStream s) throws IOException, ClassNotFoundException {
    s.defaultReadObject();
    start = new Date(start.getTime());          // defensive copy of mutable components
    end   = new Date(end.getTime());
    if (start.compareTo(end) > 0) throw new InvalidObjectException(start + " after " + end); // revalidate
}
```

## Reference Tables
| Goal | Technique | Item |
|---|---|---|
| Avoid the risk entirely | JSON / Protocol Buffers | 85 |
| Decide to serialize at all | weigh permanent commitment | 86 |
| Representation ≠ logical | custom form + `transient` + explicit UID | 87 |
| Safe reconstruction | defensive `readObject` | 88 |
| Singleton integrity | single-element `enum` | 89 |
| General robust serialization | serialization proxy pattern | 90 |

## Worked Example
Why a `Serializable` singleton needs help (Item 89). `Elvis` is a classic singleton (`public static final Elvis INSTANCE`). Making it `Serializable` breaks the singleton: each deserialization creates a *new* `Elvis`. Adding `private Object readResolve(){ return INSTANCE; }` returns the canonical instance and lets the created one be GC'd — *but* if `Elvis` has any non-`transient` object reference field, an attacker can craft a stream that steals the reference to the transient instance before `readResolve` runs. The bulletproof fix is to not fight serialization at all:
```java
public enum Elvis {
    INSTANCE;
    private final String[] favoriteSongs = {"Hound Dog", "Heartbreak Hotel"};
    public void printFavorites(){ System.out.println(Arrays.toString(favoriteSongs)); }
}
```
The JVM guarantees enum instance control across serialization and reflection — no `readResolve`, no attack surface. Prefer enums for any serialization-safe singleton.

## Key Takeaways
1. Avoid Java serialization in new systems; never deserialize untrusted bytes — prefer JSON/protobuf.
2. Implementing `Serializable` is a permanent API + security commitment — do it deliberately.
3. Use a custom serialized form (with `transient` + explicit `serialVersionUID`) unless physical == logical representation.
4. Treat `readObject` as a public constructor: validate invariants and defensively copy mutable components.
5. Use single-element enums for serialization-safe instance control instead of `readResolve`.
6. The serialization proxy pattern is the safest general technique — route deserialization through the real constructor and block direct byte attacks.

## Connects To
- **Ch8 (Item 50)**: defensive copies apply identically in `readObject`.
- **Ch2 (Item 3)** / **Ch6 (Item 34)**: enum singletons solve instance control.
- **Ch4 (Item 17)**: immutable classes serialize most safely.
- **Ch10 (Item 76)**: `readObject` validation preserves invariants like failure atomicity.
