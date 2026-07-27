# Chapter 11: Stream Processing

## Core Idea

Batch assumes bounded input; real data is unbounded. Stream processing is the incremental counterpart: process every event as it happens instead of chunking by day. The deep insight: **databases and streams are two sides of the same coin** — a replication log is already a stream of write events, so exposing it (CDC) or modeling the app on events (event sourcing) keeps any number of derived systems (caches, indexes, warehouses) consistent with one system of record. "The truth is the log. The database is a cache of a subset of the log." (Pat Helland)

## Frameworks Introduced

**Apache Kafka (log-based message broker)** — also Amazon Kinesis Streams, Twitter DistributedLog.
- *When to use*: high throughput, ordering matters within a key, consumers need replay, or the stream feeds derived state (indexes, caches, other streams).
- *How*: producers append to a partitioned, append-only log on disk; each partition assigns a monotonically increasing **offset** per message. Consumer groups are assigned whole partitions; each consumer reads its partitions sequentially, single-threaded, periodically checkpointing its offset.
- *Why it works*: the per-partition offset replaces per-message acks — broker behaves like a leader database, consumer like a follower resuming from a log sequence number. That kills ack bookkeeping and enables batching/pipelining: millions of msgs/sec while writing everything to disk (partitioning for throughput, replication for fault tolerance). Consuming is read-only; replay is just resetting an offset.
- *Failure modes*: parallelism capped at partition count; one slow message head-of-line-blocks its partition; a consumer lagging past segment retention silently misses messages (monitor lag); dying after processing but before committing the offset means reprocessing on restart — plan for at-least-once plus idempotence.

**AMQP/JMS-style brokers (RabbitMQ, ActiveMQ, IBM MQ, Azure Service Bus)** — broker assigns *individual* messages, consumer acks each, broker deletes on ack.
- *When to use*: task queues — expensive per-message work, order unimportant, no replay needed. Think async RPC.
- *Failure mode*: load balancing + redelivery reorders messages (crashed consumer's unacked m3 redelivered after m4); consuming is destructive — no re-runs, and a new consumer sees no history.

Also named: stream processors Storm, Spark Streaming (microbatch), Flink (checkpointing), Samza, Kafka Streams, Google Cloud Dataflow; CDC tools Debezium, Maxwell (MySQL binlog), Bottled Water (Postgres WAL), Kafka Connect; Esper (CEP), Event Store.

## Key Concepts

1. **Partitioned logs & consumer offsets** — total order *within* a partition only, none across partitions. Offset checkpointing = the entire consumer-progress protocol.
2. **Log compaction** — background process keeps only the latest value per key (null = tombstone/delete). Compacted log size tracks database size, not write history, so a topic can hold a *full copy of a database* forever. Kafka supports this natively.
3. **Dual-write problem** — app writes DB, then index, then cache: concurrent clients interleave (DB ends with B, index with A — permanently inconsistent, no error raised), and partial failures desync systems. Root cause: no single leader ordering writes.
4. **Change data capture (CDC)** — parse the DB's replication log, publish changes as a stream: DB becomes leader, derived systems become followers applying the *same* order. Needs an initial snapshot tied to a log position — or a compacted topic instead.
5. **Event sourcing** — same log idea at the application level: append-only log of immutable, intent-carrying domain events ("seat reserved"), state derived by deterministic replay. Later events don't supersede earlier ones, so compaction doesn't apply — snapshot state as a performance optimization only.
6. **Commands vs events** — a command may fail validation; once accepted it becomes an immutable event, a fact. Consumers cannot reject events — validate synchronously *before* the append, or split into tentative + confirmation events.
7. **Event time vs processing time** — windowing on the processor's clock creates artifacts (a restarted processor's backlog looks like a request spike). Window on event timestamps; you can never be sure a window is complete — **straggler** events arrive late. Either drop them (and track how many) or emit a correction. Device clocks lie: log three timestamps (occurred and sent per device clock, received per server clock) to estimate the offset.
8. **Window types** — *tumbling* (fixed, non-overlapping), *hopping* (fixed length, overlaps by hop size), *sliding* (all events within an interval of each other, no fixed boundaries), *session* (no fixed length; closes on inactivity gap).
9. **Stream joins** — *stream-stream* (window join: search + click events by session ID, both sides indexed in local state); *stream-table* (enrichment via a CDC-fed local replica instead of hammering a remote DB); *table-table* (join two changelogs to maintain a materialized view, e.g. Twitter timelines). All three: keep state from one input, probe it from the other. Cross-stream ordering is undefined, so joins are nondeterministic on replay.
10. **Exactly-once semantics** — the *visible effect* is as-if-once even though retries reprocess. Microbatching (Spark Streaming) and checkpointing (Flink barriers) give it inside the framework, but not once output escapes (DB writes, emails). Then you need atomic commit of output + state + offset, or **idempotence**: write the Kafka offset alongside the value in the target DB and skip already-applied updates. Idempotence assumes deterministic processing, identical replay order (a log gives you this), and fencing against zombie nodes.

## Mental Models

- **Streams and tables are calculus duals**: state is the integral of the change stream; the changelog is the derivative of state. A materialized join follows the product rule — (u·v)′ = u′v + uv′.
- **Broker as leader, consumer as follower**: a log-based broker is database replication wearing a different hat; offsets are log sequence numbers.
- **CEP inverts the database**: databases store data and treat queries as transient; CEP stores queries long-term and streams data past them.
- **The accountant's ledger**: never erase — append a compensating event. Immutability buys auditability, recovery from buggy deploys, and new read views derived from history (CQRS: write shape ≠ read shape, so denormalizing read views is safe).

## Anti-patterns

- **Dual writes** to keep systems in sync — races and partial failures guarantee silent divergence. Use CDC / a single ordered log.
- **Windowing by processing time** when lag is possible — redeploys manufacture fake traffic spikes.
- **At-least-once redelivery without idempotence** — every retry double-counts.
- Embedding search details in click events **instead of joining** — loses no-click searches; click-through rate becomes uncomputable.
- Querying a **remote DB per event** for enrichment — slow, overloads the DB; keep a CDC-fed local replica.
- Assuming an event log satisfies **deletion regulations** — you need excision-style history rewrite, and truly deleting is hard (SSDs, backups).

## Worked Example: CDC keeping a search index in sync

1. OLTP database is the system of record; clients write only to it. Its replication log already totally orders every write.
2. A CDC connector (Debezium-style, parsing the binlog/WAL) publishes each change — keyed by primary key, value = full new row — to a Kafka topic partitioned by key. Same key → same partition → per-key ordering preserved.
3. The search index is just a consumer: apply changes in log order and its state converges to the DB's (state machine replication). Cache and warehouse are additional consumers of the same topic — fan-out is free on a log.
4. Bootstrapping a new index: a raw log is truncated, so you'd normally need a snapshot tied to a log offset. Instead enable **log compaction**: each event carries the full latest row, so compaction keeps only the latest value per live key — the compacted topic *is* a full copy of the database. A new consumer starts at offset 0, scans to the head, and is caught up: no snapshot coordination, no load on the source DB.
5. Fault tolerance: commit offsets after applying writes; on crash a few events replay, and "set key=row" is naturally idempotent — exactly-once effect without transactions.
6. Caveat: CDC is async — replication lag applies, so a user may not see their own write in search results (read-your-writes).

## Key Takeaways

1. Two broker families, two contracts: AMQP/JMS = per-message ack, destructive, unordered under redelivery, good for task queues; log-based (Kafka) = partitioned, ordered, replayable, good for data integration and derived state.
2. Database writes are themselves an event stream; CDC + a log fixes the dual-write problem by making one system leader and everything else followers.
3. Log compaction turns a topic into a durable full dataset — the load-bearing trick behind Kafka-as-storage and cheap rebuilds of derived views.
4. State is a cache of the log's latest values; keep the log and you can re-derive anything, run old and new views side by side, and recover from bad deploys.
5. Distinguish event time from processing time; pick window semantics deliberately and decide your straggler policy up front.
6. All three join types reduce to "maintain state from one side, probe from the other" — and are order-sensitive across streams.
7. Exactly-once is a visible-effect guarantee built from at-least-once delivery + (atomic commit or idempotent writes keyed by offset), not a magic delivery mode.

## Problems This Solves (mapped to real system-design examples)

| Mechanism | Problem it solves | Example |
|---|---|---|
| At-least-once delivery + idempotent dedup via event ID | Retried sends to a flaky third party (APNS/FCM) must not double-notify the user | Notification system |
| Fanout workers as a stream-maintained materialized view | Precompute each user's feed instead of a fan-out read query at request time | News feed system |
| Per-recipient inbox queue + consumer-offset sync across devices | Multiple devices need independent, resumable read positions on the same message history | Chat system |
| Unbounded per-user pub/sub stream | Continuous location updates, not request/response | Nearby friends |
| Kafka stream feeding downstream consumers | Location/traffic updates must reach ETA computation and other consumers without polling | Google Maps |
| Partitions, offsets, consumer groups, exactly-once — the chapter's core machinery, used directly | The system *is* a message queue | Distributed message queue |
| Windowed aggregation with an explicit choice of where to aggregate (agent, pipeline, or query time) | Metrics ingestion must roll up before storage without losing the ability to re-slice | Metrics monitoring & alerting |
| Watermarks, tumbling/sliding windows, exactly-once via offset-atomic-with-write | A few percent double-counting error is a real financial loss, and late events are guaranteed | Ad click event aggregation |
| CDC (Debezium-style) keeping a derived cache in sync | Cache must reflect DB state without a fragile dual-write, tolerating a bounded staleness window | Hotel reservation (Redis inventory cache) |
| Async reindexing via Kafka | Search index must stay consistent with the system of record without writing to both in the request path | Distributed email service (Elasticsearch reindex) |
| Retry queue / dead-letter queue, multi-receiver pub-sub | Side effects on payment events must be retried safely and fanned out to multiple independent consumers | Payment system |
| Event sourcing + CQRS, deterministic replay | Full audit trail and the ability to reconstruct any past state exactly | Digital wallet |
| Event sourcing over an append-only log, subscriber replay | Deterministic matching engine state must be reconstructable and replayable for audit/replay | Stock exchange |

## Connects To

- **Ch 3 (Storage)**: log-structured engines and compaction — same log mechanics.
- **Ch 5 (Replication)**: replication logs, log sequence numbers, lag, read-your-writes — broker/consumer is leader/follower replication.
- **Ch 6 (Partitioning)**: topic partitioning is the same idea applied to logs.
- **Ch 7/9 (Transactions, Consistency)**: atomic commit/2PC for exactly-once output; total order broadcast = the log abstraction; fencing zombie processors.
- **Ch 8 (Distributed troubles)**: unreliable clocks and networks are why event time is hard.
- **Ch 10 (Batch)**: stream = batch with unbounded input; map-side hash joins → stream-table joins; immutable inputs → replayable logs; exactly-once is the same output-discarding discipline, done incrementally.
