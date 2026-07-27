---
name: bloch-effective-java
description: "Knowledge base from \"Effective Java\" (3rd ed.) by Joshua Bloch. Use when applying Bloch's item-by-item best practices for object creation, generics, enums, lambdas/streams, methods, exceptions, concurrency, and serialization, studying the book, or referencing its 90 items."
---

<!-- argument-hint: [topic, item number, framework name, or chapter number] -->

# Effective Java (3rd Edition)
**Author**: Joshua Bloch | **Pages**: ~413 | **Chapters**: 11 (Items 1–90) | **Generated**: 2026-07-21

## How to Use This Skill

- **Without arguments** — load the core best-practice rules below for reference.
- **With a topic** — ask about `builder`, `PECS`, `equals/hashCode`, `optional`, `defensive copy`, `volatile`; I find and read the relevant chapter.
- **With an item/chapter** — ask for `Item 31` or `ch05`; I load that chapter file.
- **Browse** — ask "what chapters do you have?" for the full index.

When you ask about a topic beyond the Core rules below, I read the relevant chapter file before answering.

---

## Core Rules & Mental Models

Bloch's book is 90 numbered "items," each a specific, defended best practice. The highest-leverage ones:

**Object creation (Ch2)**
- Prefer **static factory methods** (named, instance-controlled) and **builders** (many/optional params) over telescoping constructors and setters (Items 1–2).
- Single-element **`enum`** is the best singleton (Item 3). **Inject dependencies**; don't hardwire resources (Item 5).
- Deterministic cleanup = `AutoCloseable` + **try-with-resources**; avoid finalizers/cleaners (Items 8–9).

**Objects & classes (Ch3–4)**
- Override `equals` → **always** override `hashCode` (`31*result + fieldHash`); make `toString` informative (Items 10–12).
- **Minimize accessibility** (private by default) and **minimize mutability** (immutable unless proven otherwise) (Items 15, 17).
- **Favor composition + forwarding over inheritance** — cross-package inheritance is fragile (Item 18).
- **Prefer interfaces to abstract classes**; pair with a skeletal `Abstract*` implementation (Item 20).

**Generics (Ch5)**
- Never use **raw types**; eliminate/narrowly-suppress unchecked warnings (Items 26–27).
- **PECS**: Producer-`extends`, Consumer-`super` — the key wildcard rule for flexible APIs (Item 31).

**Enums & annotations (Ch6)**
- **`enum` over `int` constants**; attach data + constant-specific behavior; never derive meaning from `ordinal()` (Items 34–35).
- Use **`EnumSet`/`EnumMap`** instead of bit fields / ordinal-indexed arrays (Items 36–37). Always `@Override` (Item 40).

**Lambdas & streams (Ch7)**
- Lambdas > anonymous classes > (for the trivial) method references; reuse **standard functional interfaces** (Items 42–44).
- Keep stream functions **side-effect-free** — compute in **collectors**, not `forEach` (Item 46). Return `Collection`, not `Stream` (Item 47).

**Methods (Ch8)**
- **Validate parameters** at entry; **defensive-copy** mutable in/out (copy *before* validating) (Items 49–50).
- Never return `null` for a collection — return **empty**; use `Optional` for single maybe-absent values only (Items 54–55).

**Exceptions (Ch10)**
- Exceptions for **exceptional conditions only**; checked = recoverable, unchecked = programming error (Items 69–70).
- **Translate** low-level exceptions to your abstraction + chain the cause; never swallow silently (Items 73, 77).

**Concurrency (Ch11)**
- Synchronize **both reads and writes** of shared mutable data (exclusion *and* visibility). `volatile` = visibility only; `Atomic*` for compound ops (Item 78).
- **Never call an alien method while holding a lock** (Item 79). Prefer executors and `java.util.concurrent` over raw threads and `wait`/`notify` (Items 80–81).

**Serialization (Ch12)**
- **Avoid Java serialization** in new systems; never deserialize untrusted bytes (Item 85). If unavoidable, use the **serialization proxy pattern** and defensive `readObject` (Items 88, 90).

---

## Chapter Index

| # | Title | Key Items |
|---|-------|-----------|
| [ch02](chapters/ch02-creating-destroying-objects.md) | Creating and Destroying Objects | static factory (1), builder (2), enum singleton (3), DI (5), try-with-resources (9) |
| [ch03](chapters/ch03-methods-common-to-all-objects.md) | Methods Common to All Objects | equals (10), hashCode (11), toString (12), Comparable (14) |
| [ch04](chapters/ch04-classes-and-interfaces.md) | Classes and Interfaces | minimize accessibility (15), immutability (17), composition (18), interfaces (20) |
| [ch05](chapters/ch05-generics.md) | Generics | no raw types (26), lists>arrays (28), generic methods (30), PECS (31), type tokens (33) |
| [ch06](chapters/ch06-enums-and-annotations.md) | Enums and Annotations | enums (34), EnumSet (36), EnumMap (37), annotations (39), @Override (40) |
| [ch07](chapters/ch07-lambdas-and-streams.md) | Lambdas and Streams | lambdas (42), method refs (43), standard fns (44), side-effect-free (46) |
| [ch08](chapters/ch08-methods.md) | Methods | validate params (49), defensive copy (50), signatures (51), empty>null (54), Optional (55) |
| [ch09](chapters/ch09-general-programming.md) | General Programming | scope (57), for-each (58), no double for money (60), primitives>boxed (61) |
| [ch10](chapters/ch10-exceptions.md) | Exceptions | exceptional only (69), checked vs unchecked (70), translation (73), atomicity (76) |
| [ch11](chapters/ch11-concurrency.md) | Concurrency | synchronize shared data (78), avoid excess sync (79), executors (80), utilities (81) |
| [ch12](chapters/ch12-serialization.md) | Serialization | prefer alternatives (85), custom form (87), defensive readObject (88), proxy (90) |

## Topic Index

- **Autoboxing / boxed primitives** → ch09 (Item 61)
- **Builder** → ch02 (Item 2)
- **compareTo / Comparable / Comparator** → ch03 (Item 14)
- **Composition vs inheritance** → ch04 (Item 18)
- **Defensive copy** → ch08 (Item 50), ch12 (Item 88)
- **Dependency injection** → ch02 (Item 5)
- **EnumSet / EnumMap** → ch06 (Items 36–37)
- **Enums** → ch06 (Item 34), ch02 (Item 3), ch12 (Item 89)
- **equals / hashCode** → ch03 (Items 10–11)
- **Exception translation / chaining** → ch10 (Item 73)
- **Failure atomicity** → ch10 (Item 76)
- **Generics / wildcards / PECS** → ch05 (Item 31)
- **Immutability** → ch04 (Item 17)
- **Lambdas / method references** → ch07 (Items 42–43)
- **Lazy initialization** → ch11 (Item 83)
- **Optional** → ch08 (Item 55)
- **Raw types** → ch05 (Item 26)
- **Serialization proxy** → ch12 (Item 90)
- **Singleton** → ch02 (Item 3), ch12 (Item 89)
- **Static factory** → ch02 (Item 1)
- **Streams / collectors** → ch07 (Items 45–47)
- **Synchronization / volatile** → ch11 (Item 78)
- **try-with-resources** → ch02 (Item 9)
- **Type tokens** → ch05 (Item 33)
- **Validation of parameters** → ch08 (Item 49)

## Supporting Files

- [glossary.md](glossary.md) — key terms with definitions and item references
- [patterns.md](patterns.md) — reusable techniques/patterns (Builder, PECS, proxy, forwarding, …)
- [cheatsheet.md](cheatsheet.md) — decision rules and "if you see X, do Y" tables by topic

---

## Scope & Limits

Covers *Effective Java* 3rd ed. (Java 9-era; Items 1–90). Some advice predates newer features (`records`, sealed classes, pattern matching, virtual threads) — treat records as first-class immutable value carriers (Ch4) and virtual threads as an executor option (Ch11) where relevant. For hands-on codebase work, combine with project-specific tooling. Related skill: `effective-java-interview-coach` (Socratic drilling on the same material).
