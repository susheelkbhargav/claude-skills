# Chapter 7: Transactions

## Core Idea

A transaction groups reads and writes into one unit that either commits entirely or aborts entirely, so the application can safely retry instead of reasoning about partial failure. Transactions aren't a law of nature — they exist to simplify the programming model. The chapter's payload is concurrency control: which race conditions exist, which isolation level prevents which, and the three ways to implement serializability. Kleppmann's stance: weak isolation has caused real money loss, and "use an ACID database" is no protection — most "ACID" databases run weak isolation by default.

## Frameworks Introduced

**ACID** (Härder & Reuter, 1983). Use as vocabulary, not a guarantee — Kleppmann calls "ACID compliant" mostly a marketing term (BASE is even vaguer: effectively "not ACID"). Read the letters precisely: **Atomicity** = abortability (writes discarded on fault mid-transaction; retry is safe), *not* concurrency. **Consistency** = application-defined invariants — a property of the app, not the DB; "the C doesn't really belong in ACID." **Isolation** = formally serializability, in practice usually weaker. **Durability** = committed data survives crashes (WAL/fsync/replication — none absolute; SSDs violate fsync under power loss). Failure mode: assuming the "I" means serializable — Oracle's "serializable" is actually snapshot isolation.

**The isolation-level ladder** (read committed → snapshot isolation → serializable). Map each anomaly to the weakest level that prevents it (below). Why it works: each level rules out specific races at increasing cost (locks, version bookkeeping, aborts). Failure mode: trusting names — PG/MySQL "repeatable read" is snapshot isolation, DB2 "repeatable read" is serializability; "nobody really knows what repeatable read means."

## Key Concepts

The anomaly-to-level mapping (interview gold):

1. **Dirty read** — seeing uncommitted writes. Prevented by **read committed**+ (readers get the old committed value while a write is in flight).
2. **Dirty write** — overwriting uncommitted writes (Alice/Bob car sale: Bob wins the listing, Alice gets the invoice). Prevented by nearly all implementations via row-level write locks held to commit.
3. **Read skew (non-repeatable read)** — seeing the DB at different points in time (Alice sees $500 + $400 = $900 mid-transfer). Allowed under read committed; prevented by **snapshot isolation**. Bites hardest in backups and long analytic/integrity queries.
4. **Lost update** — concurrent read-modify-write cycles; later write clobbers earlier (counter 42→43, not 44). Fixes: atomic ops (`SET value = value + 1`), `SELECT ... FOR UPDATE`, automatic detection (PostgreSQL repeatable read aborts; **MySQL/InnoDB does not**), compare-and-set. Locks/CAS assume a single up-to-date copy — useless under multi-leader/leaderless replication (use commutative ops or sibling merges; LWW loses updates).
5. **Write skew** — two transactions read the same objects, then update *different* objects, each invalidating the other's premise. Generalization of lost update. Not caught by snapshot isolation or lost-update detection. Needs **serializable** isolation (or `FOR UPDATE` on the rows the decision depends on).
6. **Phantom** — a write changes the result of another transaction's search query. Worst when the check is for *absence* (room free, username untaken): `FOR UPDATE` has nothing to lock. Needs predicate/index-range locks or serializability. Last resort: **materializing conflicts** (pre-created lock rows, e.g. room×timeslot table) — ugly, concurrency control leaking into the data model.
7. **MVCC** — how snapshot isolation is built: rows versioned with `created_by`/`deleted_by` transaction IDs, update = delete + create, visibility rules hide in-progress and later-started transactions' writes. Mantra: *readers never block writers, writers never block readers*.
8. **Two-phase locking (2PL)** — shared locks to read, exclusive to write, all held until commit (acquire phase, release phase). Readers block writers and vice versa — the opposite of MVCC. Serializable but unstable tail latency and deadlocks (auto-detected, one victim aborted). **Predicate locks** cover objects matching a condition, including rows that don't exist yet — kills phantoms but slow; real systems use **index-range (next-key) locks**, a coarser safe approximation on index entries. 2PL ≠ 2PC.
9. **Actual serial execution** — one transaction at a time on one thread (VoltDB/H-Store, Redis, Datomic). Viable since ~2007: cheap RAM + short OLTP transactions. Requires stored procedures (no interactive round-trips) and in-memory data; capped at one core unless partitioned; cross-partition transactions are orders of magnitude slower.
10. **Serializable snapshot isolation (SSI)** — optimistic: run on a snapshot, detect serialization conflicts at commit, abort losers. Detects (a) stale MVCC reads (an ignored write later committed) and (b) writes affecting prior reads (index-range "tripwires" that notify, not block). PostgreSQL serializable ≥9.1; FoundationDB distributes it. Near-snapshot performance; degrades under high contention (abort storms) and long read-write transactions.

## Mental Models

- **Weak isolation is a contract**: the DB handles some races, you handle the rest — know exactly which anomalies you've signed up to prevent yourself.
- **Decisions on an outdated premise**: the shape of write skew/phantoms is read → decide → write; the premise ("2 doctors on call") can be stale at commit. The DB can't know how you used a query result — only serializability protects the premise.
- **Pessimistic vs optimistic**: 2PL and serial execution wait (serial execution is "pessimism to the extreme" — an exclusive lock on the whole DB); SSI proceeds and aborts on conflict. Low contention favors optimistic; high contention favors pessimistic or commutative atomic ops.

## Anti-patterns

- Trusting "ACID" branding — real Bitcoin-exchange losses happened on "ACID" databases at weak isolation.
- Read-modify-write in app code (ORM habit) instead of atomic DB operations — silent lost updates.
- Check-then-act on absence under snapshot isolation (bookings, usernames, double-spend checks) — phantom write skew.
- Calling single-object CAS "lightweight transactions" — transactions mean multi-object grouping.
- Not retrying aborts (ActiveRecord/Django bubble the exception) — aborts exist to enable retry. But: dedupe (commit ack may be lost), backoff on overload, retry only transient errors, mind side effects (emails).

## Worked Example: doctors on-call write skew

Shift needs ≥1 doctor on call; Alice and Bob are both on call, both sick, both click "go off call" simultaneously. Each transaction: `SELECT count(*) FROM doctors WHERE on_call = true AND shift_id = 1234` → sees 2 → passes the `>= 2` check → `UPDATE doctors SET on_call = false WHERE name = <self> AND shift_id = 1234` → commit. Both commit; zero doctors on call.

Why snapshot isolation misses it: each transaction reads a consistent snapshot where both doctors are on call, and they update *different rows* — no dirty write, no lost update, nothing for detection to catch. Serially, the second would have seen count = 1 and refused; the anomaly exists only under concurrency. That's write skew.

Fixes: serializable isolation — under SSI each update trips the index-range record of the other's read on shift 1234; first commits, second aborts and retries seeing count = 1. Under 2PL the second SELECT blocks. Without serializability: `SELECT ... FOR UPDATE` on the on-call rows works *here* because the rows exist to be locked — unlike the phantom variants. "At least one on call" isn't expressible as a standard constraint.

## Key Takeaways

1. Atomicity = abortability; Consistency is your job; "ACID" says almost nothing about actual isolation.
2. The ladder: read committed kills dirty reads/writes; snapshot isolation (MVCC) adds read-skew protection; only serializable kills write skew and phantoms (lost updates: sometimes detected at snapshot isolation, DB-dependent).
3. Vendor names lie: PG/MySQL "repeatable read" and Oracle "serializable" = snapshot isolation; DB2 "repeatable read" = serializable.
4. Write skew = read-decide-write across objects; phantoms = write skew on absence checks. `FOR UPDATE` fixes the former, not the latter.
5. Three roads to serializability: serial execution (in-memory, single-core cap), 2PL (general, slow tails, deadlocks), SSI (optimistic, aborts under contention).
6. Locks/CAS need a single up-to-date copy — they break under multi-leader/leaderless replication.

## Problems This Solves (mapped to real system-design examples)

| Mechanism | Problem it solves | Example |
|---|---|---|
| Atomic read-modify-write (locks, Lua scripts, sorted sets) | Race on a shared counter under concurrent access | Rate limiter (Redis counter races) |
| Non-atomic multi-store commit made safe (pending→uploaded flag) | Multi-store write isn't wrapped in a single DB transaction, so partial completion must be representable | Google Drive upload (metadata written `pending`, flipped on S3 callback) |
| Offset + downstream write committed atomically | At-least-once delivery must not double-count high-value aggregates | Ad click event aggregation ("a few percent difference = millions of dollars") |
| Explicit lost-update walkthrough → `SELECT FOR UPDATE`, optimistic version column, `CHECK` constraint, idempotency-key unique constraint | Double-booking / double-submission under concurrent requests | Hotel reservation system |
| Double-entry ledger on ACID SQL + idempotency-key unique constraint | Money must never be created or destroyed, retries must not double-move it | Payment system |
| 2PC → TC/C → Saga (guided tour of the distributed-transaction spectrum) | Atomicity across services when no single DB transaction spans them | Digital wallet |
| WAL + transactional mapping-table update during compaction | Merging small objects can't leave the object-to-location mapping inconsistent mid-merge | S3-like object storage (secondary) |

## Connects To

- Ch 5 (Replication): lost updates across replicas, LWW, siblings/merges; durability via replication.
- Ch 6 (Partitioning): why serial execution partitions, cross-partition cost.
- Ch 8–9: distributed faults; distributed transactions, 2PC (≠ 2PL), linearizability (CAP's "consistency").
- Ch 3 (Storage): WAL underpins atomicity/durability; append-only B-trees (CouchDB, Datomic, LMDB) as alternative MVCC.
