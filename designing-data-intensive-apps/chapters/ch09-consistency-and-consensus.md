# Chapter 9: Consistency and Consensus

## Core Idea

The best way to build fault-tolerant distributed systems is to find general-purpose abstractions with useful guarantees, implement them once, and let applications rely on them — the same move transactions made for single-node faults. The key abstraction here is **consensus**: getting all nodes to agree on something. The chapter climbs a ladder: linearizability (strongest single-object guarantee), causal ordering (a weaker, cheaper middle ground), total order broadcast, and finally consensus itself — and shows these top rungs are all equivalent problems in disguise.

## Frameworks Introduced

**Linearizability** — *When to use:* locking/leader election, uniqueness constraints (usernames, seat booking), cross-channel timing dependencies (e.g., a message queue racing a file store). *How:* make the system appear as if there is only a single copy of the data, and all operations on it are atomic; once any client reads a new value, all subsequent reads (by any client) must return it — a **recency guarantee**. *Why it works:* every operation takes effect atomically at some point between its start and end, so there is a single timeline all clients agree on. *Failure mode:* it's expensive. Single-leader replication is only *potentially* linearizable; multi-leader is not; Dynamo-style leaderless quorums are not, even with strict quorums (w + r > n) — an execution can still return old and new values to concurrent readers. Attiya & Welch proved linearizable response times are inevitably high in variable-delay networks; even multicore RAM isn't linearizable.

**Two-phase commit (2PC)** — *When to use:* atomic commit across multiple nodes/partitions, or heterogeneous systems via XA. *How:* a **coordinator** (transaction manager) sends *prepare* to all participants; each replies yes only if it can definitely commit (writes durably, surrenders the right to abort); if all say yes, the coordinator writes the decision to its log (the true commit point), then sends commit/abort. *Failure mode:* see Worked Example — 2PC is a **blocking** atomic commit protocol. It provides safety but not termination; it is not fault-tolerant consensus.

## Key Concepts

- **Linearizability vs. serializability** (interview-sharp): *Serializability* is an isolation property of *transactions* — multi-object, guarantees transactions behave as if executed in *some* serial order. *Linearizability* is a *recency guarantee* on reads/writes of a single object (a "register") — no transactions, no ordering across objects. Both together = **strict serializability (strong-1SR)**. 2PL-based serializability is typically linearizable; **SSI is serializable but not linearizable** (it reads from a consistent snapshot, which is stale by design).
- **CAP theorem, and Kleppmann's critique:** during a network partition you choose linearizability or availability. But CAP is "either Consistent or Available *when Partitioned*" — it considers one consistency model (linearizability) and one fault (partitions), says nothing about delays, crashes, or trade-offs in normal operation. Kleppmann: mostly of historical interest; avoid CAP for precise reasoning. Note CP/AP labels mislead: many systems are neither.
- **Causal consistency:** the strongest consistency model that doesn't lose availability under partitions. Causality gives a *partial* order (concurrent ops are incomparable); linearizability implies causality (total order). Many apps that "need" linearizability actually only need causal consistency.
- **Lamport timestamps vs. version vectors:** a Lamport timestamp is (counter, node ID); each node/client carries the max counter seen, giving a *total order consistent with causality*. Version vectors instead *detect* whether two operations are concurrent or causally ordered. Lamport timestamps are compact but can't tell you concurrency — and can't enforce uniqueness constraints online, because the total order is only final after collecting all operations.
- **Total order broadcast:** messages delivered reliably, in the same order, to all nodes — the order is fixed at delivery time (stronger than timestamp ordering). It's exactly a replication log. Linearizable compare-and-set can be built on it (append claim, read log, first claim for the username wins) and vice versa — both are **equivalent to consensus**.
- **FLP result:** consensus is impossible with a deterministic algorithm in the asynchronous model if a node may crash — but adding timeouts or randomness makes it solvable in practice. Great theoretical importance, limited practical restriction.
- **Fault-tolerant consensus:** properties = uniform agreement, integrity, validity, **termination** (the liveness property 2PC lacks). Algorithms: Viewstamped Replication, **Paxos** (Multi-Paxos for sequences), **Raft**, **Zab**. Most decide a *sequence* of values, i.e., they implement total order broadcast directly.
- **Epochs and quorums:** each leader election gets a monotonically increasing **epoch number** (ballot in Paxos, view in VSR, term in Raft); a leader must get a quorum to confirm no higher-epoch leader exists before deciding. Two overlapping quorums (election + proposal) prevent split brain.
- **Exactly-once message processing:** atomically committing the message acknowledgment and the database writes (heterogeneous transaction) means a message is *effectively* processed exactly once, even with retries.
- **ZooKeeper/etcd:** consensus outsourced. Provide linearizable atomic operations (compare-and-set → locks/leases with **fencing tokens** via monotonic zxid), ephemeral nodes (session-based liveness), change notifications — used for leader election, partition assignment, service discovery, membership services. Only the atomic ops truly require consensus.

## Mental Models

- **The single-copy illusion:** linearizability = "behave as though there is only one copy of the data and all operations are atomic." Everything about its cost follows from maintaining that illusion across replicas.
- **Consensus reductions:** linearizable compare-and-set, atomic commit, total order broadcast, locks/leases, uniqueness constraints, and leader election are all *reducible to consensus* — solve one, you've solved them all. This is why ZooKeeper sits under half the infrastructure you use.
- **Consensus uses a leader, but electing a leader needs consensus:** not a contradiction — a chicken-and-egg escape via epoch numbers. You don't need a unique leader forever, only a unique leader *per epoch*.

## Anti-patterns

- Reasoning about system design with "CP vs AP" labels — most real systems fit neither cleanly.
- Assuming strict quorum reads/writes (w + r > n) give linearizability. They don't, without read repair synchronously before returning (and even then no linearizable CAS).
- Using timestamps/Lamport clocks to enforce uniqueness constraints — the winner is only known after the fact.
- Believing a leader is unique because a lock says so, without fencing tokens — a paused (GC) old leader can wake and corrupt data.
- Leaving XA in-doubt transactions to rot: orphaned transactions hold row locks forever, blocking everything, until an admin manually resolves them; heuristic decisions break atomicity.
- Treating 2PC as fault-tolerant consensus. It has safety, not termination.

## Worked Example: 2PC coordinator crash after prepare

App completes writes on databases 1 and 2, then the coordinator runs phase 1: *prepare* to both. Both vote "yes" — each has durably written the transaction and irrevocably promised to commit if told to; each has surrendered its right to abort. The coordinator writes "commit" to its log, sends commit to database 2 (which commits), then **crashes before sending commit to database 1** (Figure 9-10).

Database 1 is now **in doubt** (uncertain). It cannot unilaterally abort — database 2 may have committed, and unilateral abort breaks atomicity. It cannot unilaterally commit — maybe another participant voted no. It can't even ask database 2 safely in general (it doesn't know the coordinator's decision). The *only* way forward is to **wait for the coordinator to recover** and read its decision from its transaction log — the commit point of 2PC is really a single-node atomic commit on the coordinator. Meanwhile the in-doubt participant holds its locks, blocking every other transaction touching those rows. This is why 2PC is a *blocking* protocol; 3PC "fixes" it only by assuming bounded delays and a perfect failure detector, which real networks don't provide.

Bonus (linearizability violation): Alice and Bob both check the 2014 World Cup final score. Alice refreshes, sees the result, exclaims it; Bob then refreshes and gets a stale "game in progress" from a lagging replica. Because Bob's query *started after* Alice's completed (via a side channel — her voice), returning the older value violates linearizability's recency guarantee.

## Key Takeaways

1. Linearizability = recency on a single object; serializability = serial-order illusion for multi-object transactions. Know the distinction cold; combined they're strict serializability.
2. Wide swaths of coordination — leader election, atomic commit, locks, uniqueness, total order broadcast, linearizable CAS — are all equivalent to consensus.
3. 2PC gives atomic commit but blocks on coordinator failure; fault-tolerant consensus (Paxos/Raft/Zab/VSR) adds termination via majority quorums and epochs.
4. Causal consistency is the practical middle ground: preserves what users perceive as "order" without linearizability's latency and availability cost.
5. FLP says consensus is impossible in the pure asynchronous model — timeouts and randomness rescue it in practice.
6. Don't implement consensus yourself: use ZooKeeper/etcd, and use fencing tokens with any lock or lease.
7. If your system has a leader and can't tolerate its loss, you either wait, pick manually (failover), or run a consensus algorithm — and consensus needs a majority alive.

## Problems This Solves (mapped to real system-design examples)

| Mechanism | Problem it solves | Example |
|---|---|---|
| Total order broadcast via a sequencer stamping every event | Matching engine must be deterministic and replayable — every replica must apply operations in the identical order | Stock exchange |
| Raft/Paxos consensus (not plain heartbeat failover) for leader election | Eliminate split-brain entirely, not just reduce its odds, on a system holding money/placement state | S3-like storage's placement service, digital wallet's replicated log, stock exchange's failover |
| ZooKeeper/etcd for coordination (locks, leader election, membership) | Many independent servers need one agreed-on assignment (which server owns which channel/shard/target) | Chat system (server assignment), nearby friends (pub/sub channel ring), distributed message queue (broker metadata + leader election), metrics monitoring (collector coordination) |
| Two-phase commit for cross-service atomicity | Multiple services must commit or abort together despite no shared DB transaction | Hotel reservation (2PC vs Saga), digital wallet |
| CAP / CP-vs-AP framing made explicit | Forces a stated choice between strict consistency and availability during a partition | Leaderless key-value store |
| Per-partition local order vs. global order (Lamport-style reasoning) | Global total order is overkill when only per-entity order matters | Chat system (per-channel sequence number over global Snowflake ID), unique-ID generator |

## Connects To

- **Ch 5 (Replication):** single-leader logs ≈ total order broadcast; leaderless quorums and LWW explain why Dynamo-style systems aren't linearizable.
- **Ch 7 (Transactions):** serializability, 2PL, SSI — the isolation half of strict serializability; single-node atomic commit underlies 2PC.
- **Ch 8 (Distributed troubles):** process pauses, unbounded delays, and clock skew are exactly why fencing tokens and epochs exist.
- **Ch 11 (Stream processing):** total order broadcast as a log foreshadows event logs; exactly-once semantics returns there.
