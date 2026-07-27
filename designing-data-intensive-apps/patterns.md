# Patterns

## LSM-tree (Ch 3)
**When to use**: write-heavy workloads needing high random-write throughput (Cassandra, RocksDB, LevelDB, HBase).
**How**: writes buffer in an in-memory memtable; flushed as sorted SSTable segments at a few MB; reads check memtable then segments newest→oldest; background compaction merges segments; Bloom filters skip disk for missing keys; append-only log for crash recovery.
**Trade-offs**: turns random writes sequential (fast writes, compression, range queries) vs. compaction competing with live requests (p99 spikes) and write amplification from rewriting.

## B-tree (Ch 3)
**When to use**: the default — read-heavy/mixed workloads, strong transactional semantics.
**How**: fixed-size pages (~4 KB) in a balanced tree, branching factor in the hundreds; update overwrites the leaf in place; full pages split; WAL written before pages for crash safety.
**Trade-offs**: one copy per key → predictable reads, range locks attach to the tree; but every write hits disk twice (WAL + page), in-place mutation needs latches.

## Single-leader replication (Ch 5)
**When to use**: default replication; one authoritative write order, no conflicts.
**How**: all writes to the leader, followers apply the replication log (row-based logical log preferred); semi-synchronous (one sync follower) for durability.
**Trade-offs**: simple reasoning vs. failover danger — lost async writes, split brain, timeout-driven failover storms. Reads on followers see replication lag.

## Multi-leader replication (Ch 5)
**When to use**: multi-datacenter local writes, offline clients, collaborative editing.
**How**: leader per datacenter/device exchanges changes async; conflicts resolved by avoidance (home leader per record), LWW, merge/siblings, or app resolution.
**Trade-offs**: local write latency and DC-outage tolerance vs. inevitable async write conflicts; breaks auto-increment/constraints; "dangerous territory."

## Quorum reads/writes — leaderless (Ch 5)
**When to use**: write availability with no failover (Dynamo-style: Cassandra, Riak).
**How**: n replicas; write acked by w, read from r, with w + r > n so sets overlap; version numbers pick winners; read repair + anti-entropy fix stragglers.
**Trade-offs**: no failover, tunable latency vs. probabilistic freshness only — sloppy quorums, concurrent writes, and partial write failures break the overlap; no read-your-writes.

## Hash partitioning (Ch 6)
**When to use**: default for write-heavy key-value load; even distribution matters most.
**How**: hash the key, assign partitions ranges of hashes; rebalance with fixed partition count (many more partitions than nodes) — never hash mod N.
**Trade-offs**: uniform load vs. no range queries; a single hot key still hammers one partition (salt with random suffix, gather on read).

## Key-range partitioning (Ch 6)
**When to use**: range scans needed (time series, prefixes).
**How**: contiguous sorted key ranges per partition; dynamic split at size threshold (HBase: 10 GB), merge when small; pre-split empty tables.
**Trade-offs**: cheap range scans vs. hot spots on monotonic keys (timestamps). Compound-key fix: hash first component for partition, sort by the rest (Cassandra).

## Secondary indexes, local vs global (Ch 6)
**When to use**: local (document-partitioned) when writes dominate; global (term-partitioned) when reads dominate.
**How**: local — each partition indexes its own docs; global — index partitioned by term, updated async (DynamoDB GSI).
**Trade-offs**: local = one-partition writes, scatter/gather reads; global = one-partition reads, multi-partition async writes. No free option.

## MVCC / snapshot isolation (Ch 7)
**When to use**: long reads (backups, analytics) that must not block or see torn state.
**How**: versioned rows tagged with creating/deleting txids; visibility rules hide uncommitted and later-started transactions.
**Trade-offs**: readers never block writers and vice versa; but misses lost updates (DB-dependent) and write skew entirely.

## Two-phase locking, 2PL (Ch 7)
**When to use**: serializability with general interactive transactions.
**How**: shared locks to read, exclusive to write, all held to commit; index-range locks approximate predicate locks to stop phantoms.
**Trade-offs**: true serializability vs. readers/writers blocking each other, deadlocks, unstable tail latency.

## Serializable snapshot isolation, SSI (Ch 7)
**When to use**: serializability with mostly-low contention (PostgreSQL ≥9.1).
**How**: optimistic — run on a snapshot, detect stale reads and read-write conflicts at commit, abort losers.
**Trade-offs**: near-snapshot performance vs. abort storms under contention; long read-write transactions suffer.

## Serial execution (Ch 7)
**When to use**: short, in-memory OLTP transactions (VoltDB, Redis).
**How**: one transaction at a time per core, via stored procedures; partition for parallelism.
**Trade-offs**: no concurrency bugs at all vs. single-core cap and orders-of-magnitude slower cross-partition transactions.

## Fencing tokens (Ch 8)
**When to use**: any lock/lease guarding a resource where a paused client may wake believing it still holds the lease.
**How**: lock service issues a monotonically increasing token per grant (ZooKeeper zxid); the resource itself rejects writes with tokens lower than one already seen.
**Trade-offs**: stops zombie writers server-side; doesn't stop deliberate forgery (Byzantine).

## Total order broadcast (Ch 9)
**When to use**: replication logs, uniqueness constraints, anything needing one agreed order.
**How**: deliver messages reliably in the same order to all nodes; append a claim, read the log, first claim wins. Equivalent to consensus; implemented by Raft/Paxos/Zab.
**Trade-offs**: linearizable operations buildable on top vs. consensus cost — majority quorum, leader bottleneck.

## Two-phase commit, 2PC (Ch 9)
**When to use**: atomic commit across partitions/heterogeneous systems (XA) — reluctantly.
**How**: coordinator sends prepare; yes votes surrender the right to abort; coordinator logs the decision (the real commit point), then broadcasts it.
**Trade-offs**: atomicity across systems vs. blocking — coordinator crash after prepare leaves participants in doubt, holding locks until it recovers. Not fault-tolerant consensus (no termination).

## Change data capture, CDC (Ch 11)
**When to use**: keeping search indexes, caches, warehouses consistent with a source DB; kills the dual-write problem.
**How**: parse the DB's replication log (Debezium), publish to a partitioned log keyed by primary key; derived systems consume in order — leader/follower for free.
**Trade-offs**: single ordered truth, replayable fan-out vs. async lag (no read-your-writes on derived views).

## Event sourcing (Ch 11)
**When to use**: audit trails, evolving read views, domain logic captured as intent.
**How**: append immutable domain events ("seat reserved"); state = deterministic replay; snapshots only as optimization; validate commands before appending.
**Trade-offs**: full history, new views derivable anytime vs. no compaction (events don't supersede), deletion regulations are hard.

## Log compaction (Ch 11)
**When to use**: a topic must serve as a full durable copy of a dataset (Kafka).
**How**: background thread keeps only the latest value per key, null = tombstone; new consumers replay from offset 0 to bootstrap without snapshots.
**Trade-offs**: log size tracks DB size, not history; only valid where latest-value-wins semantics hold (not event sourcing).

## Reduce-side vs map-side joins (Ch 10)
**When to use**: reduce-side sort-merge — no assumptions about inputs; broadcast hash — small side fits in memory; partitioned hash — inputs co-partitioned on the join key.
**How**: sort-merge — shuffle both sides by join key, secondary sort puts the dimension record first; broadcast — load small input into a hash table in every mapper; partitioned — each mapper hashes only its partition.
**Trade-offs**: sort-merge pays a full shuffle but always works; map-side joins skip the shuffle entirely but demand input properties (size or physical layout metadata).

## Skew handling in joins (Ch 10)
**When to use**: linchpin keys (celebrities) overload one reducer.
**How**: spray the hot key's records across random reducers and replicate the other side to all of them (Pig/Hive skew join); two-stage aggregation for GROUP BY.
**Trade-offs**: removes the straggler bottleneck vs. replication overhead and hot-key bookkeeping.
