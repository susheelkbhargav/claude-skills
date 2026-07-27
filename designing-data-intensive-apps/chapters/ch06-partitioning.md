# Chapter 6: Partitioning

## Core Idea

Replication (Ch5) copies data; partitioning splits it. When a dataset or query load is too big for one machine, break it into partitions (a.k.a. shards in MongoDB/Elasticsearch, regions in HBase, tablets in BigTable, vnodes in Cassandra/Riak) so each record belongs to exactly one partition, and spread partitions across nodes in a shared-nothing cluster. The whole game is spreading data and load *evenly*: unfair partitioning is called **skew**, and a partition with disproportionately high load is a **hot spot** — in the worst case one node does all the work while nine sit idle. Partitioning is usually combined with replication (each node leads some partitions, follows others), but the two choices are mostly independent.

## Frameworks Introduced

**Partitioning by key range**
- *When to use*: you need efficient range scans (time-series reads, prefix lookups) and can tolerate hot-spot risk.
- *How*: assign each partition a contiguous range of sorted keys (like encyclopedia volumes). Boundaries adapt to data density, chosen manually or automatically. Keys stay sorted within a partition (SSTable-style), so range queries hit one or few partitions. Used by BigTable, HBase, RethinkDB, early MongoDB.
- *Why it works / failure mode*: works because sort order is preserved — the key doubles as a concatenated index. Fails when the access pattern concentrates on one end of the range: a timestamp key means all of today's writes land on today's partition while the rest idle.

**Partitioning by hash of key**
- *When to use*: default for write-heavy key-value workloads where even distribution matters more than range queries.
- *How*: hash each key (MD5/FNV — doesn't need to be cryptographic), assign each partition a range of *hashes*. Similar keys get scattered uniformly.
- *Why it works / failure mode*: a good hash turns skewed inputs into a uniform distribution. Fails on (a) range queries — adjacent keys are scattered, so MongoDB in hash mode sends range queries to all partitions; Riak/Couchbase/Voldemort just don't support them on the primary key — and (b) a single hot key: hashing identical keys gives identical hashes, so a celebrity's ID still hammers one partition.

**Consistent hashing caveat**: Karger et al.'s consistent hashing (pseudo-random boundaries, built for CDN caches) rebalances poorly for databases and is rarely used in practice — Cassandra abandoned it pre-1.2. "Consistent" here has nothing to do with replica or ACID consistency. Kleppmann's advice: avoid the term; say *hash partitioning*.

## Key Concepts

1. **Skew / hot spots**: uneven load defeats partitioning. Hashing reduces but can't eliminate it — for a single hot key the application must intervene, e.g. append a 2-digit random suffix to spread writes over 100 keys (cost: reads must gather all 100 and you must track which keys are split).
2. **Compound primary key (Cassandra)**: hash only the first column to pick the partition; remaining columns are a concatenated sort index within it. `(user_id, update_timestamp)` gives even distribution across users plus efficient per-user time-range scans.
3. **Document-partitioned (local) secondary index**: each partition indexes only its own documents. Writes touch one partition; reads must **scatter/gather** across all partitions — prone to tail latency amplification. Used by MongoDB, Riak, Cassandra, Elasticsearch, SolrCloud, VoltDB.
4. **Term-partitioned (global) secondary index**: index partitioned by the indexed term itself (or its hash). Reads hit one partition; a single document write may touch many index partitions, so updates are typically asynchronous (DynamoDB GSIs) — synchronous would need a distributed transaction.
5. **Rebalancing**: moving data between nodes as load/nodes change. Requirements: fair load after, keep serving reads/writes during, move no more data than necessary.
6. **hash mod N (anti-pattern)**: `hash(key) mod N` reassigns most keys whenever N changes — rebalancing becomes catastrophically expensive.
7. **Fixed number of partitions**: create far more partitions than nodes up front (e.g. 1,000 for 10 nodes); new nodes steal whole partitions. Key→partition mapping never changes, only partition→node. Riak, Cassandra ≥1.2, Elasticsearch, Couchbase.
8. **Dynamic partitioning**: split a partition when it exceeds a size threshold (HBase default 10 GB), merge when it shrinks — B-tree-like. Partition count adapts to data volume. Caveat: an empty database starts with one partition (one busy node) — mitigate with **pre-splitting**.
9. **Request routing / service discovery**: three options — (1) client hits any node, which forwards; (2) a partition-aware routing tier (mongos, moxi); (3) partition-aware clients. The hard part is keeping everyone's view of partition→node assignment in agreement.
10. **Coordination**: most systems delegate to ZooKeeper (HBase, SolrCloud, Kafka; Espresso via Helix); Cassandra/Riak use gossip instead, trading node complexity for no external dependency. Node IPs change slowly enough that DNS suffices for that layer.

## Mental Models

- **Each partition is a small database of its own** — independence is what makes scaling linear; anything crossing partitions (global indexes, multi-key writes) is where the pain lives.
- **The encyclopedia shelf**: key-range partitioning is volumes on a shelf — sorted, easy to find, but uneven unless boundaries follow the data.
- **Move the mapping, not the data**: good rebalancing schemes (fixed partitions, dynamic splits) change only partition→node assignment; hash mod N fails because it entangles the two.
- **Pick your pain: write-cheap/read-expensive (local index) or read-cheap/write-expensive (global index)** — secondary indexes force this trade; there is no free option.

## Anti-patterns

- **hash mod N** for node assignment — nearly all keys move when the cluster resizes.
- **Monotonic keys under key-range partitioning** (timestamps, auto-increment) — all writes hit the newest partition.
- **Trusting "consistent hashing" in DB docs** — the Karger scheme doesn't rebalance well for databases; what's actually meant is usually hash partitioning with fixed partitions.
- **Fully automatic rebalancing + automatic failure detection**: a slow node gets declared dead, rebalancing dumps its load on the rest, overloading them → cascading failure. A human commit step (Couchbase/Riak/Voldemort style) prevents operational surprises.
- **DIY secondary indexes on a key-value store** in application code — race conditions and partial write failures silently desync index and data.

## Worked Example

Sensor network, key = measurement timestamp (`year-month-day-hour-minute-second`).

1. **Key-range partitioning**: keys sorted, so "all readings for March" is one cheap range scan. But writes arrive in real time — every insert lands in *today's* partition. One node saturates on writes; the rest idle. Classic hot spot from a monotonically increasing key.
2. **Hash partitioning**: hash the timestamp — writes now spread uniformly. But the sort order is gone: "all readings from 17:08:10 to 17:09:00" must query every partition.
3. **Compound-key fix (Cassandra-style)**: key = `(sensor_name, timestamp)`. Hash `sensor_name` to pick the partition; sort by `timestamp` within it. With many concurrent sensors, write load spreads evenly; per-sensor time-range queries are single-partition and efficient. Cost: a multi-sensor time-range query becomes one range query per sensor name — you've traded a global scan for N targeted scans, which is fine when N is bounded.

This is the interview move: identify the hot key structure, split "which partition" from "sort within partition."

## Key Takeaways

1. Key-range gives cheap range scans but risks hot spots; hash gives even load but kills range queries. Compound keys buy both, scoped: hash the first component, sort by the rest.
2. Hashing can't fix a single hot key — that's application-level (key salting), with read-side gather cost.
3. Local secondary indexes: fast writes, scatter/gather reads. Global (term-partitioned): fast reads, multi-partition (usually async) writes.
4. Rebalance by moving whole partitions: fixed-count for hash partitioning, dynamic split/merge for key ranges. Never hash mod N.
5. Routing is service discovery; the authoritative partition map lives in ZooKeeper-like coordination or gossip, and everyone must agree.
6. Keep a human in the loop for rebalancing — automation plus failure detection can cascade.

## Problems This Solves (mapped to real system-design examples)

| Mechanism | Problem it solves | Example |
|---|---|---|
| Consistent-hash ring + virtual nodes | Rebalance without moving most keys when node count changes | Consistent-hashing service itself; leaderless KV store's replica placement |
| Hash sharding + read replicas | Point-lookup index too big for one node, read-heavy | URL shortener (`shortURL → longURL` map) |
| Per-host queue partitioning (one worker owns a host) | Politeness + even crawl load without a shared lock | Web crawler's URL frontier |
| Compound key: hash partition col + sort/cluster col | Even write spread *and* cheap per-entity range scans | Chat system `(channel_id, message_id)`; email service `user_id` partition with denormalized read/unread tables |
| Range partitioning + hot-range mitigation, shard-map manager | Prefix/range queries at scale without one range absorbing all load | Search autocomplete (trie shards `a-m`/`n-z`, further split on skew) |
| Space-filling curve (geohash / quadtree / S2) as a 1-D index | Multi-dimensional (lat/lon) queries on a system built for 1-D sorted keys | Proximity service, Google Maps tile addressing |
| Topic partitions + consumer-group rebalancing | Parallelize an ordered log while keeping per-key order | Distributed message queue |
| Range-vs-hash partitioning by score, scatter/gather top-N | Global ranking query across shards without one shard owning it all | Real-time leaderboard |
| Consistent-hash placement groups + metadata sharded by `hash(bucket, object)` | Spread object storage across nodes while keeping listing/pagination sane | S3-like object storage |
| Hash sharding by a natural key (`hotel_id`) | Even load without breaking the single-shard-per-entity invariant transactions rely on | Hotel reservation system |
| Collector placement on a hash ring | Avoid two collectors scraping the same target | Metrics monitoring & alerting |

## Connects To

- **Ch5 (Replication)**: partitions are replicated; each node leads some, follows others. Orthogonal choices.
- **Ch3 (Storage)**: sorted keys within a partition = SSTables; dynamic splitting mirrors B-tree node splits; concatenated/multi-column indexes.
- **Ch7 & Ch9 (Transactions, Consistency/Consensus)**: synchronous global index updates and agreeing on partition assignment both need distributed transactions/consensus — which is why global indexes go async and routing goes through ZooKeeper.
- **Ch10 (Batch)**: MPP warehouses parallelize joins/aggregations across partitions, far beyond single-key NoSQL access.
