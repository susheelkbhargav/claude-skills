# Kleppmann Cheatsheet — interview reasoning aid

## Decision rules
- Reads ≫ writes (orders of magnitude) → precompute at write time (fan-out), because cheap reads beat cheap writes at that ratio; exempt extreme keys (celebrities) and merge at read (Ch 1).
- Stateful data fits one node → scale up, because distribution adds real complexity; distribute for fault tolerance and latency, not fashion (Ch 1, 8).
- Data is a self-contained tree loaded whole → document DB; many-to-many now or later → relational; "anything relates to everything" / variable-length traversals → graph (Ch 2). Data always grows more interconnected — judge by future relationships.
- Internal services → Protobuf/Thrift + gRPC; cross-org APIs → JSON/REST; dynamically generated schemas / big analytic files → Avro (Ch 4). Never reuse a tag number; post-v1 fields must be optional/defaulted.
- Conflict avoidance (one home leader per record) beats conflict resolution whenever routing allows (Ch 5).
- Single hot key → hashing can't save you; salt the key (2-digit suffix → 100 keys), pay gather-on-read (Ch 6).
- Check-then-act on **absence** (username, booking, double-spend) → serializable isolation or materialized conflict rows; `FOR UPDATE` only locks rows that exist (Ch 7).
- Need elapsed time → monotonic clock; need event order → logical clock (Lamport/version vector); wall clocks are confidence intervals (Ch 8).
- Any lock/lease over a shared resource → fencing token checked by the resource, because a GC-paused client wakes believing its lease is valid (Ch 8).
- Need leader election / uniqueness / atomic CAS → don't build consensus; use ZooKeeper/etcd (Ch 9).
- Keeping N systems in sync → never dual-write; one system of record + CDC log, everything else follows (Ch 11).
- Join in batch: small side fits memory → broadcast hash; co-partitioned → partitioned hash; else sort-merge (Ch 10).
- Stream enrichment → CDC-fed local replica, never per-event remote DB queries (Ch 11).

## Decision trees

**Replication mode**: default single-leader (one write order, no conflicts) → need multi-DC local writes/offline → multi-leader (accept conflict resolution) → need no-failover write availability → leaderless quorums (accept probabilistic freshness, app-side merges).

**Isolation level = weakest that kills your anomaly**: dirty read/write → read committed; read skew (backups, long queries) → snapshot isolation; lost update → atomic ops / CAS / `FOR UPDATE` (PG detects at RR, MySQL doesn't); write skew or phantom → serializable only. Roads to serializable: in-memory short txns → serial execution; high contention → 2PL; low contention → SSI.

**Partitioning**: need range scans → key-range (hot-spot risk on monotonic keys) → else hash. Both needed → compound key: hash(first column) picks partition, rest sorts within. Secondary index: write-heavy → local (scatter reads); read-heavy → global/term (async writes).

**Broker**: task queue, order irrelevant, no replay → AMQP/RabbitMQ; data integration, per-key order, replay, derived state → log-based/Kafka.

## Trade-off matrices

| | LSM | B-tree |
|---|---|---|
| Writes | sequential, fast | 2× (WAL+page) |
| Reads | multi-segment + Bloom | one place per key |
| Tail latency | compaction spikes | predictable |
| Transactions | awkward | range locks on tree |

Benchmark **your** workload — no portable LSM/B-tree answer.

| | Single-leader | Multi-leader | Leaderless |
|---|---|---|---|
| Write conflicts | none | inevitable, async | concurrent siblings |
| Failover | dangerous | per-DC | none needed |
| Consistency ceiling | linearizable-ish | no | not linearizable even w+r>n |

| Anomaly | RC | SI | Serializable |
|---|---|---|---|
| Dirty read/write | ✓ | ✓ | ✓ |
| Read skew | ✗ | ✓ | ✓ |
| Lost update | ✗ | DB-dependent | ✓ |
| Write skew / phantom | ✗ | ✗ | ✓ |

Names lie: PG/MySQL "repeatable read" = SI; Oracle "serializable" = SI; DB2 "RR" = serializable. Serializability = ordering across a transaction; linearizability = recency on one object; both = strict serializability. SSI is serializable but **not** linearizable.

## Thresholds & defaults
- w + r > n; typical n=3, w=r=2 (survives 1 down); n=5, w=r=3 (2 down).
- B-tree page ~4 KB; memtable flush at a few MB; HBase partition split at 10 GB; fixed partitions ~1,000 for 10 nodes.
- Clock drift ~200 ppm (Google); NTP over internet ≥~35 ms error; Spanner TrueTime ~7 ms uncertainty.
- ~12 network faults/month per medium datacenter; 10k disks ≈ 1 death/day.
- Twitter: 4.6k tweets/s vs 300k timeline reads/s → write fan-out, avg 75 followers → 345k writes/s.
- Amazon: 100 ms latency ≈ −1% sales; rethink architecture every ~10× load growth.

## Tells & smells
- "We write to the DB, then update the cache/index" → dual-write divergence; use CDC.
- LWW on mutable keys (Cassandra default) → acknowledged writes silently vanish.
- Timeouts + auto-failover + auto-rebalance, all aggressive → cascading failure under load spike.
- `SELECT` count/check then `UPDATE` different rows → write skew; snapshot isolation won't catch it.
- Mean latency on a dashboard → tail (your best customers) hidden; use percentiles, add histograms.
- hash mod N for placement → full reshuffle on resize.
- Client checks its own lease validity → zombie writer; fence at the resource.
- Windowing on processing time → redeploy backlog looks like a traffic spike.
- At-least-once delivery, no idempotence → every retry double-counts.
- ORM read-modify-write → silent lost updates; use atomic DB ops.
