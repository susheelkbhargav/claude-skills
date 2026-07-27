# Drill Progress Tracker

Updated by the coach at the end of every session. `Status` is the coach's judgment of the candidate's last answer on that item — not self-reported.

**Status legend**: `untested` (never drilled) · `weak` (missed contract detail, follow-up, or "why not X") · `solid` (answered like a senior panel would accept) · `solid*` (solid, but hasn't been re-tested in 3+ sessions — due for a spot check)

## Resurface queue
<!-- Coach appends item numbers here when Status → weak. Remove a row once it flips back to solid. Pull from the TOP of this list at the start of each new session before drilling untested items. -->

- Item 46 (streams: side effects + short-circuit matching) — drilled 2026-07-24, weak

## Item Status

| Item | Title | Chapter | Status | Last Drilled | Notes |
|---|---|---|---|---|---|
| 1 | Consider static factory methods instead of constructors | ch01 | untested | — | |
| 2 | Consider a builder when faced with many constructor parameters | ch01 | untested | — | |
| 3 | Enforce the singleton property with a private constructor or an enum type | ch01 | untested | — | |
| 4 | Enforce noninstantiability with a private constructor | ch01 | untested | — | |
| 5 | Prefer dependency injection to hardwiring resources | ch01 | untested | — | |
| 6 | Avoid creating unnecessary objects | ch01 | untested | — | |
| 7 | Eliminate obsolete object references | ch01 | untested | — | |
| 8 | Avoid finalizers and cleaners | ch01 | untested | — | |
| 9 | Prefer try-with-resources to try-finally | ch01 | untested | — | |
| 10 | Obey the general contract when overriding equals | ch02 | untested | — | PRIORITY |
| 11 | Always override hashCode when you override equals | ch02 | untested | — | PRIORITY |
| 12 | Always override toString | ch02 | untested | — | PRIORITY |
| 13 | Override clone judiciously | ch02 | untested | — | PRIORITY |
| 14 | Consider implementing Comparable | ch02 | untested | — | PRIORITY |
| 15 | Minimize the accessibility of classes and members | ch03 | untested | — | PRIORITY |
| 16 | In public classes, use accessor methods, not public fields | ch03 | untested | — | PRIORITY |
| 17 | Minimize mutability | ch03 | untested | — | PRIORITY |
| 18 | Favor composition over inheritance | ch03 | untested | — | PRIORITY |
| 19 | Design and document for inheritance or else prohibit it | ch03 | untested | — | PRIORITY |
| 20 | Prefer interfaces to abstract classes | ch03 | untested | — | PRIORITY |
| 21 | Design interfaces for posterity | ch03 | untested | — | PRIORITY |
| 22 | Use interfaces only to define types | ch03 | untested | — | PRIORITY |
| 23 | Prefer class hierarchies to tagged classes | ch03 | untested | — | PRIORITY |
| 24 | Favor static member classes over nonstatic | ch03 | untested | — | PRIORITY |
| 25 | Limit source files to a single top-level class | ch03 | untested | — | PRIORITY |
| 26 | Don't use raw types | ch04 | untested | — | PRIORITY |
| 27 | Eliminate unchecked warnings | ch04 | untested | — | PRIORITY |
| 28 | Prefer lists to arrays | ch04 | untested | — | PRIORITY |
| 29 | Favor generic types | ch04 | untested | — | PRIORITY |
| 30 | Favor generic methods | ch04 | untested | — | PRIORITY |
| 31 | Use bounded wildcards to increase API flexibility | ch04 | untested | — | PRIORITY (PECS) |
| 32 | Combine generics and varargs judiciously | ch04 | untested | — | PRIORITY |
| 33 | Consider typesafe heterogeneous containers | ch04 | untested | — | PRIORITY |
| 34 | Use enums instead of int constants | ch05 | untested | — | PRIORITY |
| 35 | Use instance fields instead of ordinals | ch05 | untested | — | PRIORITY |
| 36 | Use EnumSet instead of bit fields | ch05 | untested | — | PRIORITY |
| 37 | Use EnumMap instead of ordinal indexing | ch05 | untested | — | PRIORITY |
| 38 | Emulate extensible enums with interfaces | ch05 | untested | — | PRIORITY |
| 39 | Prefer annotations to naming patterns | ch05 | untested | — | PRIORITY |
| 40 | Consistently use the Override annotation | ch05 | untested | — | PRIORITY |
| 41 | Use marker interfaces to define types | ch05 | untested | — | PRIORITY |
| 42 | Prefer lambdas to anonymous classes | ch06 | untested | — | PRIORITY |
| 43 | Prefer method references to lambdas | ch06 | untested | — | PRIORITY |
| 44 | Favor the use of standard functional interfaces | ch06 | untested | — | PRIORITY |
| 45 | Use streams judiciously | ch06 | untested | — | PRIORITY |
| 46 | Prefer side-effect-free functions in streams | ch06 | weak | 2026-07-24 | Thought anyMatch returns element; missed short-circuit trace both times |
| 47 | Prefer Collection to Stream as a return type | ch06 | untested | — | PRIORITY |
| 48 | Use caution when making streams parallel | ch06 | untested | — | PRIORITY |
| 49 | Check parameters for validity | ch07 | untested | — | |
| 50 | Make defensive copies when needed | ch07 | untested | — | |
| 51 | Design method signatures carefully | ch07 | untested | — | |
| 52 | Use overloading judiciously | ch07 | untested | — | |
| 53 | Use varargs judiciously | ch07 | untested | — | |
| 54 | Return empty collections or arrays, not nulls | ch07 | untested | — | |
| 55 | Return optionals judiciously | ch07 | untested | — | |
| 56 | Write doc comments for all exposed API elements | ch07 | untested | — | |
| 57 | Minimize the scope of local variables | ch08 | untested | — | |
| 58 | Prefer for-each loops to traditional for loops | ch08 | untested | — | |
| 59 | Know and use the libraries | ch08 | untested | — | |
| 60 | Avoid float and double if exact answers are required | ch08 | untested | — | |
| 61 | Prefer primitive types to boxed primitives | ch08 | untested | — | |
| 62 | Avoid strings where other types are more appropriate | ch08 | untested | — | |
| 63 | Beware the performance of string concatenation | ch08 | untested | — | |
| 64 | Refer to objects by their interfaces | ch08 | untested | — | |
| 65 | Prefer interfaces to reflection | ch08 | untested | — | |
| 66 | Use native methods judiciously | ch08 | untested | — | |
| 67 | Optimize judiciously | ch08 | untested | — | |
| 68 | Adhere to generally accepted naming conventions | ch08 | untested | — | |
| 69 | Use exceptions only for exceptional conditions | ch09 | untested | — | |
| 70 | Use checked exceptions for recoverable conditions and runtime exceptions for programming errors | ch09 | untested | — | |
| 71 | Avoid unnecessary use of checked exceptions | ch09 | untested | — | |
| 72 | Favor the use of standard exceptions | ch09 | untested | — | |
| 73 | Throw exceptions appropriate to the abstraction | ch09 | untested | — | |
| 74 | Document all exceptions thrown by each method | ch09 | untested | — | |
| 75 | Include failure-capture information in detail messages | ch09 | untested | — | |
| 76 | Strive for failure atomicity | ch09 | untested | — | |
| 77 | Don't ignore exceptions | ch09 | untested | — | |
| 78 | Synchronize access to shared mutable data | ch10 | untested | — | PRIORITY |
| 79 | Avoid excessive synchronization | ch10 | untested | — | PRIORITY |
| 80 | Prefer executors, tasks, and streams to threads | ch10 | untested | — | PRIORITY |
| 81 | Prefer concurrency utilities to wait and notify | ch10 | untested | — | PRIORITY |
| 82 | Document thread safety | ch10 | untested | — | PRIORITY |
| 83 | Use lazy initialization judiciously | ch10 | untested | — | PRIORITY |
| 84 | Don't depend on the thread scheduler | ch10 | untested | — | PRIORITY |
| 85 | Prefer alternatives to Java serialization | ch11 | untested | — | |
| 86 | Implement Serializable with great caution | ch11 | untested | — | |
| 87 | Consider using a custom serialized form | ch11 | untested | — | |
| 88 | Write readObject methods defensively | ch11 | untested | — | |
| 89 | For instance control, prefer enum types to readResolve | ch11 | untested | — | |
| 90 | Consider serialization proxies instead of serialized instances | ch11 | untested | — | |

## Session Log
<!-- One line per session: date, items drilled, outcomes. Newest first. -->

- 2026-07-24: Item 46 (anyMatch/allMatch short-circuit + side effects) — weak. Candidate prepping for Java/Spring interview next week; weak areas self-reported: Optional, streams, comparators, collectors.
