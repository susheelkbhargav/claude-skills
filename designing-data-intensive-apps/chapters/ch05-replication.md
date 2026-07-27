# Chapter 5: Replication

## Core Idea

Replication — keeping copies of the same data on multiple machines — buys you lower latency (data near users), higher availability (survive node/datacenter failure), and read throughput (scale-out reads). Replicating immutable data is trivial; **all the difficulty is in handling changes**. Three algorithms cover almost every distributed database: single-leader, multi-leader, and leaderless. Each trades write availability against consistency and conflict-handling complexity.

## Frameworks Introduced

**Single-leader replication** (a.k.a. master-slave, active/passive)
- *When*: default choice; easy to reason about, no conflict resolution needed. Used by PostgreSQL, MySQL, MongoDB, Kafka.
- *How*: all writes go to one leader; it ships a replication log (change stream) to followers, which apply writes in the same order. Reads can hit any replica.
- *Why it works*: single node serializes all writes, so there's one authoritative order. *Failure mode*: leader failover. With async replication, the promoted follower may lack the old leader's latest writes — the common fix is discarding them, violating durability promises. Worse: **split brain** (two nodes both believe they're leader, both accept writes → data loss/corruption), and a too-short failover timeout causes unnecessary failovers under load spikes, making a struggling system worse. Some ops teams prefer manual failover for exactly these reasons.

**Quorum reads/writes (leaderless)**
- *When*: high availability and low write latency matter more than strict consistency (Dynamo-style: Cassandra, Riak, Voldemort). No failover needed at all.
- *How*: with n replicas, a write must be acked by w nodes, a read queries r nodes. If **w + r > n**, the read set and write set must overlap, so at least one read replica has the latest value (version numbers pick the winner). Stale replicas are fixed by **read repair** (client writes newer value back on read) and **anti-entropy** (background diff-and-copy process).
- *Why it works*: pigeonhole overlap. *Failure mode*: the overlap guarantee has real holes — sloppy quorums break it, concurrent writes have no defined winner, a write that succeeded on < w replicas isn't rolled back on the nodes where it landed. Treat w + r > n as a probability knob, not an absolute guarantee; you still don't get read-your-writes or monotonic reads.

## Key Concepts

- **Sync vs. async replication**: sync guarantees the follower is up to date but one slow/dead follower blocks all writes; fully async risks losing confirmed writes on leader failure. Practical middle ground: *semi-synchronous* — one sync follower, rest async, so data lives on ≥ 2 nodes.
- **Replication log implementations**: statement-based (breaks on NOW()/RAND(), ordering-sensitive statements), WAL shipping (couples replicas to storage-engine byte format → blocks zero-downtime version upgrades), logical/row-based log (decoupled, version-tolerant, feeds change data capture), trigger-based (flexible, app-level, higher overhead).
- **Replication lag & eventual consistency**: "eventually" is deliberately unbounded — lag is sub-second normally, minutes under load or network trouble. Design for the question "what if lag is an hour?"
- **Read-after-write (read-your-writes) consistency**: a user always sees their own submitted data. Implementations: read your own profile from the leader; read from leader for one minute after any write; or client remembers a timestamp/log-sequence-number of its last write and only reads replicas caught up to it. Cross-device access makes this harder (metadata must be centralized; devices may route to different datacenters).
- **Monotonic reads**: a user never sees time go backwards across successive reads. Cheap fix: pin each user to one replica (hash of user ID).
- **Consistent prefix reads**: causally ordered writes are seen in order. Mainly a sharding problem — independent partitions have no global write order.
- **Multi-leader replication**: each datacenter (or device, or browser tab) has a leader; leaders exchange changes async. Wins: local write latency, datacenter-outage tolerance, offline operation (calendar sync), collaborative editing. Cost: **write conflicts are inevitable** and detection is asynchronous — too late to ask the user. Widely considered "dangerous territory": retrofitted, breaks auto-increment/triggers/constraints.
- **Conflict handling**: best strategy is *avoidance* (route each record's writes to one home leader). Otherwise you need *convergent* resolution: last-write-wins, merge values, or record the conflict for app/user resolution (CouchDB on-read handlers). Research directions: CRDTs, mergeable persistent structures, operational transformation (Google Docs).
- **Last write wins (LWW)**: pick highest timestamp, discard the rest. Achieves convergence *by silently dropping acknowledged writes* — and clock skew can drop even non-concurrent writes. Only safe if keys are written once and immutable (Cassandra's UUID-key recommendation). Cassandra's only conflict mode.
- **Sloppy quorum + hinted handoff**: when a client can't reach the n "home" nodes, accept writes on any reachable w nodes (crash on the neighbor's couch); once the partition heals, hand the writes back home. Raises write availability but it's **not a real quorum** — only a durability assurance; reads may miss the value until handoff completes.
- **Version vectors**: one version counter per replica per key, sent to clients on read and back on write. Lets the server distinguish overwrite (happens-before) from concurrent write (keep both as *siblings*). Deletion needs **tombstones** so removed items don't resurrect on merge.
- **Bootstrapping a new follower**: can't just copy files off a live leader (torn, inconsistent snapshot). Standard procedure: (1) take a consistent snapshot at a specific log position (`pg_basebackup`, MongoDB initial sync), (2) copy snapshot to the new node, (3) follower requests the replication log from that exact position, (4) replay forward to catch up. Risk: if snapshot+catchup takes too long, the leader may have already garbage-collected that log segment — production systems retain segments long enough or parallelize the transfer.
- **Real-system anchors**: single-leader — PostgreSQL (streaming replication), MySQL (binlog), MongoDB (replica sets), Redis (`replicaof`), Kafka (partition leader + ISR followers). Multi-leader — PostgreSQL BDR, MySQL Group Replication, CouchDB/PouchDB, Firebase Realtime DB. Leaderless — Cassandra, Riak, Voldemort, DynamoDB.

## Mental Models

- **Concurrency is about knowledge, not wall-clock time.** A happens-before B iff B knew about / built on A. If neither knew about the other, they're concurrent — regardless of physical timing (network delays make "same time" meaningless). Three cases only: A→B, B→A, or concurrent; concurrent means a conflict must be resolved.
- **Overlap arithmetic.** Every quorum claim reduces to set intersection: w writers + r readers over n homes must share a node. The moment writes can land outside the home set (sloppy quorum, hinted handoff, partial failures), the intersection — and the guarantee — evaporates.
- **A replica set is a promise ledger.** Async replication means the database confirmed writes it can still lose. Every anomaly in this chapter (lost failover writes, stale reads, resurrected cart items) is a confirmed promise the system quietly broke.

## Anti-patterns

- Pretending async replication is synchronous — "recipe for problems down the line."
- LWW on mutable keys: acknowledged writes vanish; clock skew makes it worse (Cassandra's default!).
- Auto-failover with aggressive timeouts under load → failover storms. GitHub incident: stale follower promoted, reused auto-increment keys, leaked private data via a Redis index keyed on those IDs.
- Fencing missing on failover → split brain; STONITH done badly can shoot *both* nodes.
- Trusting multi-leader conflict detection blind: PostgreSQL BDR lacks causal ordering; Tungsten doesn't even detect conflicts. Read the docs, test the guarantees.
- Union-merging siblings that include deletions without tombstones → deleted items reappear (Amazon cart bug).
- All-followers-synchronous: any one node down halts all writes.

## Worked Example

**Quorum reasoning, n=3, w=2, r=2.** User 1234 updates a profile picture; the write goes to all 3 replicas in parallel, but replica 3 is rebooting. Replicas 1 and 2 ack (version 7) → w=2 satisfied, write succeeds; nobody fails over. Replica 3 comes back holding stale version 6. User 2345 now reads: the read fans out, r=2 responses arrive — say replica 1 (v7) and replica 3 (v6). Since w + r = 4 > 3, the read set *must* include at least one of the two written nodes; version numbers reveal v7 wins. The client returns v7 and performs **read repair**, pushing v7 back to replica 3. Tolerance math: w<n means writes survive one node down; r<n means reads do; n=5, w=r=3 tolerates two down. Now break it: a network partition cuts the client off from replicas 1–3 and a **sloppy quorum** accepts the write on two outside nodes — w=2 "succeeded," yet a subsequent r=2 read of the home nodes sees only v6. The overlap argument silently died; you kept durability, not consistency.

## Key Takeaways

1. Single-leader is the default for a reason: one write order, no conflicts. Its price is failover, and failover is where the bodies are buried (lost writes, split brain, timeout tuning).
2. Semi-synchronous (one sync follower) is the pragmatic durability/availability compromise.
3. Replication-lag anomalies have names and targeted fixes: read-your-writes (leader reads / timestamp tracking), monotonic reads (replica pinning), consistent prefix (causal ordering per partition). Cite these precisely in interviews.
4. w + r > n gives probabilistic freshness, not linearizability — enumerate the edge cases (sloppy quorum, concurrent writes, partial write failure) before claiming "quorums are consistent."
5. Multi-leader/leaderless buy write availability by deferring conflicts; someone must converge replicas — LWW (lossy), merge/siblings + version vectors (safe but app must merge), or CRDTs (automatic).
6. Conflict avoidance (one home leader per record) beats conflict resolution whenever routing allows it.

## Problems This Solves (mapped to real system-design examples)

| Mechanism | Problem it solves | Example |
|---|---|---|
| Single-leader + async | Simple default, ordered writes, cheap read scaling | Master→slave scaling, most CRUD backends |
| Semi-sync / ISR (tunable ack modes) | Trade durability vs. latency per write | Distributed message queue (Kafka `ACK=all/1/0`, `replica.lag.max.messages`) |
| Leaderless quorum (w+r>n) | High write availability, no failover step | Dynamo-style key-value store; multi-DC rate-limiter counters |
| Sloppy quorum + hinted handoff | Survive a partition without blocking writes | Key-value store — but it breaks the overlap guarantee you were relying on |
| Version vectors + siblings | Detect concurrent writes instead of silently dropping one | Google-Drive-style sync conflict, surfaced to the user to merge |
| Multi-leader | Local-write latency across DCs / offline clients | Multi-DC document store, collaborative editing, offline-first mobile |
| Logical/row-based replication log → CDC | Keep a derived store (cache, search index) in sync without a fragile dual-write | Hotel reservation inventory cache (Debezium→Redis), async search reindex (DB→Kafka→Elasticsearch) |
| Consensus-backed leader election (Raft/Paxos, not plain heartbeat failover) | Eliminate split-brain entirely, not just reduce its odds | Object-storage placement service, distributed ledger/wallet, matching-engine sequencer |

## Connects To

- **Ch 6 (Partitioning)**: replication assumed the full dataset per node; sharding relaxes that. Consistent-prefix anomalies are fundamentally a cross-partition problem.
- **Ch 7 (Transactions)**: stronger guarantees than app-level workarounds; snapshot isolation for consistent prefixes.
- **Ch 8 (Distributed system troubles)**: unreliable clocks are why LWW and timestamp ordering fail; network partitions drive split brain.
- **Ch 9 (Consistency & Consensus)**: leader election is consensus; linearizability vs. quorums; why even w + r > n timing edge cases break.
- **Ch 11 (Stream processing)**: logical replication logs feed change data capture.
