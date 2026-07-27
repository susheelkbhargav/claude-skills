---
name: designing-data-intensive-apps
description: "Knowledge base from \"Designing Data-Intensive Applications\" by Martin Kleppmann. Use when applying Kleppmann's frameworks for replication, partitioning, transactions, consistency, batch/stream processing, storage engines — for system design interviews, studying the book, or referencing its concepts."
---

<!-- argument-hint: [topic, framework name, or chapter number] -->

# Designing Data-Intensive Applications
**Author**: Martin Kleppmann | **Pages**: ~491 (this copy: ch1–11, no ch12) | **Chapters**: 11 | **Generated**: 2026-07-24

## How to Use This Skill

- **Without arguments** — load core frameworks below for reference
- **With a topic** — ask about `replication`, `transactions`, `kafka`; the agent finds and reads the relevant chapter
- **With chapter** — ask for `ch05`; loads that chapter file
- **Interview drill** — ask "drill me on <topic>": agent opens with a scenario/symptom (never the concept name), candidate answers first, agent pushes back like a skeptical panelist, then reveals the concept using the chapter file
- **Browse** — "what chapters do you have?"

For topics not in Core Frameworks below, read the relevant chapter file before answering.

---

## Core Frameworks & Mental Models

### The three concerns (ch1)
Every data system is judged on **Reliability** (works correctly despite faults — build fault-*tolerance*, don't prevent faults), **Scalability** (describe load with *load parameters*, measure with *percentiles* — p95/p99, not averages; tail latency amplification: one slow backend call drags the whole user request), **Maintainability** (operability, simplicity via good abstractions, evolvability).

### Data model choice (ch2)
- Use **document model** when data is self-contained tree (locality, schema-on-read), one-to-many, rarely joined.
- Use **relational** when many-to-many relationships dominate and joins are common.
- Use **graph** when *anything* can relate to *anything* (many-to-many taken to the extreme).
- Failure mode of documents: many-to-many creeps in → app-level joins, denormalization drift.

### Storage engine trade-off (ch3)
- **LSM-trees**: sequential writes, high write throughput; cost = compaction (write amplification, background I/O interfering with p99). Reads need memtable + SSTables + Bloom filters.
- **B-trees**: read-predictable, one place per key (easier transactional locking); cost = random writes, page splits.
- Rule: write-heavy → LSM; read-latency-predictable / strong transactional locking → B-tree.
- **OLTP vs OLAP**: separate them; warehouse = star schema + column-oriented storage + compression (bitmap/RLE).

### Evolvability of data (ch4)
Rolling upgrades force **backward compatibility** (new code reads old data) *and* **forward compatibility** (old code reads new data). Avro's writer's/reader's schema resolution; Protobuf/Thrift tag numbers — never reuse or renumber tags. Three dataflow modes: via database, via services (REST/RPC), via async messages.

### Replication (ch5) — top interview chapter
- **Single-leader**: simple, ordered writes; failover pitfalls (split brain, lost writes on async lag).
- **Multi-leader**: multi-datacenter, offline clients; cost = write conflicts, resolve with LWW (loses data!), version vectors, or CRDT-style merges.
- **Leaderless (Dynamo-style)**: quorums **w + r > n**; sloppy quorums + hinted handoff break the guarantee; read repair + anti-entropy.
- Replication-lag anomalies to name on demand: **read-your-own-writes**, **monotonic reads**, **consistent prefix reads**.

### Partitioning (ch6)
- **Key-range**: efficient range scans; risk = hot spots on sequential keys (timestamps!). Fix: compound key (sensor_id, timestamp).
- **Hash**: even spread; lose range queries.
- Secondary indexes: **local/document-partitioned** (write cheap, read = scatter/gather) vs **global/term-partitioned** (read cheap, write fans out).
- Rebalancing: fixed partition count or dynamic; **never hash mod N**.

### Transactions (ch7)
Anomaly → weakest isolation level that stops it:
- dirty read/write → **read committed**
- read skew (non-repeatable read) → **snapshot isolation** (MVCC)
- lost update → atomic ops, or SI with lost-update detection, or explicit locks
- **write skew / phantoms** → only **serializable** (2PL, actual serial execution, or SSI — optimistic)
Doctors-on-call is the canonical write-skew story: two transactions read the same predicate, each writes disjoint rows, invariant broken.

### Distributed systems reality (ch8)
Partial failure is the defining property. Unbounded network delays (timeout is the only detector), clock skew (time-of-day vs monotonic clocks; never order events by wall clock), process pauses (GC, VM migration) — a node can pause *mid-critical-section* holding a lease. Defense: **fencing tokens** (monotonically increasing, storage rejects stale). Assume crash-recovery, non-Byzantine. State guarantees as **safety** (nothing bad ever) vs **liveness** (eventually good).

### Consistency & consensus (ch9)
- **Linearizability**: single-copy illusion, recency guarantee — about single-object reads/writes. **Serializability**: transactions behave as in *some* serial order — about multi-object transactions. Different axes; combined = strict serializability. Interviewers love this distinction.
- CAP: during a partition, choose linearizability or availability; Kleppmann: mostly unhelpful — network delay (not partitions) is why systems drop linearizability.
- Causal consistency is the strongest level that doesn't sacrifice latency/availability; Lamport timestamps give total order but can't detect concurrency; version vectors can.
- **2PC** blocks if coordinator crashes after prepare (participants stuck "in doubt") — that's why consensus (Raft/Paxos = total order broadcast) beats 2PC; leader election and atomic commit reduce to consensus. ZooKeeper = consensus-as-a-service (locks, leases, leader election, membership).

### Batch (ch10)
Unix philosophy → MapReduce → dataflow engines (Spark/Flink). Joins: **reduce-side sort-merge** (general, slow), **map-side broadcast hash** (small table), **map-side partitioned hash** (both pre-partitioned same way). Derived data is recomputable — fault tolerance by recomputation, immutable inputs.

### Streams (ch11)
- **Log-based broker (Kafka)**: partitioned append-only log, consumer offsets, replayable, order preserved per partition. **AMQP-style**: per-message ack, load-balanced, not replayable. Replay/fan-out/order → log; task queue → AMQP.
- **Dual-write problem**: app writing DB + search index separately = inconsistency. Fix: **CDC** (one system of record, others follow the log) or **event sourcing**.
- Time: event time vs processing time; windows (tumbling/hopping/sliding/session); stragglers.
- Exactly-once = at-least-once + idempotence (offset stored with output).

---

## Chapter Index

| # | Title | Key Frameworks |
|---|-------|----------------|
| [ch01](chapters/ch01-reliable-scalable-maintainable.md) | Reliable, Scalable, Maintainable Applications | load parameters, percentiles, fault tolerance |
| [ch02](chapters/ch02-data-models-query-languages.md) | Data Models and Query Languages | document vs relational vs graph, schema-on-read |
| [ch03](chapters/ch03-storage-and-retrieval.md) | Storage and Retrieval | LSM vs B-tree, SSTables, column storage, OLTP/OLAP |
| [ch04](chapters/ch04-encoding-and-evolution.md) | Encoding and Evolution | backward/forward compat, Avro/Protobuf, rolling upgrade |
| [ch05](chapters/ch05-replication.md) | Replication | leader models, quorums, lag anomalies, conflicts |
| [ch06](chapters/ch06-partitioning.md) | Partitioning | range vs hash, hot spots, secondary indexes, rebalancing |
| [ch07](chapters/ch07-transactions.md) | Transactions | isolation levels, anomalies, MVCC, 2PL, SSI |
| [ch08](chapters/ch08-trouble-with-distributed-systems.md) | Trouble with Distributed Systems | partial failure, clocks, pauses, fencing tokens |
| [ch09](chapters/ch09-consistency-and-consensus.md) | Consistency and Consensus | linearizability, CAP, total order broadcast, 2PC, Raft/ZooKeeper |
| [ch10](chapters/ch10-batch-processing.md) | Batch Processing | MapReduce, joins, dataflow engines, derived data |
| [ch11](chapters/ch11-stream-processing.md) | Stream Processing | Kafka logs, CDC, event sourcing, windows, exactly-once |

## Topic Index

- **ACID / isolation levels** → ch07
- **Avro / Protobuf / Thrift / schema evolution** → ch04
- **B-trees / LSM-trees / SSTables** → ch03
- **CAP theorem** → ch09
- **CDC (change data capture)** → ch11
- **Clocks / time / GC pauses** → ch08
- **Column storage / data warehouse / OLAP** → ch03
- **Conflict resolution / LWW / version vectors** → ch05, ch09
- **Consensus / Raft / Paxos / ZooKeeper** → ch09
- **Consistent hashing / rebalancing** → ch06
- **Document vs relational vs graph** → ch02
- **Event sourcing** → ch11
- **Exactly-once semantics** → ch11
- **Failover / split brain** → ch05
- **Fencing tokens** → ch08
- **Hot spots / skew** → ch06, ch10
- **Idempotence** → ch11
- **Joins (batch: map-side/reduce-side)** → ch10
- **Kafka / log-based brokers** → ch11
- **Linearizability vs serializability** → ch09, ch07
- **Lost update / write skew / phantoms** → ch07
- **MapReduce / Spark / dataflow** → ch10
- **MVCC / snapshot isolation** → ch07
- **Partitioning (sharding)** → ch06
- **Percentiles / tail latency** → ch01
- **Quorums (w+r>n)** → ch05
- **Read-your-writes / monotonic reads** → ch05
- **Replication (single/multi-leader, leaderless)** → ch05
- **Scalability / load parameters** → ch01
- **Secondary indexes (partitioned)** → ch06
- **Serializability (2PL / SSI / serial execution)** → ch07
- **Stream joins / windows** → ch11
- **Total order broadcast** → ch09
- **Two-phase commit (2PC)** → ch09
- **Unreliable networks / partial failure** → ch08

## Supporting Files

- [glossary.md](glossary.md) — all key terms with definitions
- [patterns.md](patterns.md) — all techniques and design patterns
- [cheatsheet.md](cheatsheet.md) — decision rules and trade-off tables (pre-interview skim)

---

## Scope & Limits

Covers ch1–11 of the book only (this PDF lacks ch12, "The Future of Data Systems"). Book published 2017: no mention of newer systems (e.g. Kubernetes-era operators, newer Kafka exactly-once internals, CRDT products) — frameworks still apply, name-drop carefully. For hands-on implementation, combine with project-specific skills.
