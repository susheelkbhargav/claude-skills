# Chapter 8: The Trouble with Distributed Systems

## Core Idea
A single computer is deterministic: it works or it crashes, never something in between. A distributed system loses that idealization — you get **partial failures** that are non-deterministic, and the three unreliable pillars everything rests on are: the network (packets lost or arbitrarily delayed), clocks (out of sync, jumping forward or back), and processes (paused at any point for arbitrary lengths of time). A node can know nothing for sure; it can only guess from messages it receives or fails to receive. Reliability must be engineered on top of unreliable components — like TCP over IP — by making assumptions explicit (a **system model**) and designing algorithms proven correct within it.

## Frameworks Introduced

- **System models (timing × node failure)**: formalize what an algorithm may assume.
  - Timing: **synchronous** (bounded network delay, bounded process pauses, bounded clock error — unrealistic for practical systems), **partially synchronous** (behaves synchronously *most* of the time, but bounds are occasionally shattered arbitrarily — the realistic model), **asynchronous** (no timing assumptions at all, not even a clock/timeouts — very restrictive).
  - Node failures: **crash-stop** (a failed node never returns), **crash-recovery** (nodes crash and may return; stable storage survives, in-memory state is lost), **Byzantine** (nodes may do anything, including lying).
  - When to use: state your model before claiming any algorithm is correct — most real server-side systems are best modeled as **partially synchronous with crash-recovery faults**.
  - Why it works / failure mode: it distills messy reality into a tractable set of faults you can reason about and prove properties against. Failure mode: reality leaks — e.g., crash-recovery assumes stable storage survives, but disks get corrupted and firmware bugs lose data (GitHub's Jan 2016 incident), silently breaking quorum algorithms that rely on nodes remembering what they stored.

- **Safety vs liveness properties**: define algorithm correctness precisely.
  - **Safety** = "nothing bad happens": if violated, you can point at the exact moment it broke, and the damage cannot be undone (e.g., fencing-token *uniqueness*, *monotonic sequence*). **Liveness** = "something good eventually happens": may not hold now, but hope remains (e.g., *availability*; "eventually" is the giveaway — eventual consistency is a liveness property).
  - How: require safety to hold **always**, even if all nodes crash or the network fails entirely; allow caveats on liveness (e.g., a response is required only if a majority is up and the network eventually recovers).
  - Why it works: it lets you demand hard guarantees where corruption is irreversible while accepting best-effort where waiting is tolerable.

## Key Concepts
- **Unbounded delays / timeouts**: asynchronous packet networks give no delivery-time guarantee. No response is indistinguishable among: request lost, request queued, remote node crashed, node paused (GC), response lost, response delayed. Timeouts are the only detection tool, but they can't distinguish node failure from network failure; too-short timeouts falsely declare live nodes dead, shifting load and risking cascading failure. No "correct" timeout exists — measure round-trip distributions and jitter, or adapt continuously (Phi Accrual failure detector, used in Akka and Cassandra).
- **Queueing is the source of delay variability**: switch queues under congestion, OS run queues, VM scheduling pauses, TCP flow control and retransmission. Delays explode near capacity; noisy neighbors in multitenant clouds make them unpredictable. Concrete data: ~12 network faults/month in one medium datacenter (half isolating a machine, half a whole rack); a switch software upgrade delayed packets over a minute; NICs that drop inbound packets while sending outbound fine.
- **Packet vs circuit switching**: telephone circuits reserve fixed bandwidth end-to-end, giving *bounded delay*; Ethernet/IP are packet-switched, optimized for bursty traffic and utilization at the cost of queueing. Variable delay is a cost/utilization trade-off, not a law of nature.
- **Time-of-day vs monotonic clocks**: time-of-day (wall-clock) clocks are NTP-synced but can be forcibly reset and jump backwards — unsuitable for measuring elapsed time. Monotonic clocks (`CLOCK_MONOTONIC`, `System.nanoTime()`) always move forward and are right for timeouts/durations, but absolute values are meaningless across machines.
- **Clock skew and drift**: quartz drifts with temperature — Google assumes 200 ppm (6 ms drift per 30 s sync interval; 17 s per day). NTP over the internet: ~35 ms minimum error, spikes to ~1 s under congestion. **Leap seconds** produce 59- or 61-second minutes and have crashed major systems; mitigate by *smearing* the adjustment over a day. A clock reading is really a **confidence interval**, not a point — Spanner's TrueTime returns [earliest, latest] and waits out the interval before committing so transaction timestamps reflect causality (GPS/atomic clocks per datacenter keep uncertainty ~7 ms).
- **Timestamps cannot order events**: last-write-wins (LWW, in Cassandra/Riak) with time-of-day timestamps silently drops causally-later writes when the later writer's clock lags — data vanishes with no error. Use **logical clocks** (counters tracking happens-before) for ordering, not physical clocks.
- **Process pauses**: stop-the-world GC (minutes have been observed), VM suspension/live migration, laptop lid closes, CPU steal time, synchronous disk I/O and page faults/thrashing, SIGSTOP. A thread can be preempted at any point for any duration and resume without noticing — so a lease-validity check made before processing may be stale by the time the write executes.
- **Byzantine faults**: nodes that "lie" — send arbitrary corrupted or malicious messages (Byzantine Generals Problem, Lamport). Tolerance matters in aerospace (radiation flips memory) and mutually-untrusting multi-org systems (blockchain). Most datacenter systems rightly assume **non-Byzantine** (unreliable-but-honest) nodes: you control all nodes, BFT protocols need a >2/3 supermajority of correct nodes, and an attacker who compromises one node can likely compromise all (same software) — so auth, access control, and firewalls remain the real defense. Still guard against *weak lying*: application-level checksums, input sanitization, NTP clients querying multiple servers and discarding outliers.

## Mental Models
- **The truth is defined by the majority**: a node cannot trust its own judgment — it may be semi-disconnected (receiving but unable to send), or wake from a 1-minute GC pause not realizing it was declared dead. Decisions (including "is node X dead?") belong to a **quorum**; there can only be one majority, and a node the majority declared dead must step down even if it feels perfectly alive.
- **Distributed systems are multithreading without shared memory**: arbitrary pauses and interleavings, but none of the tools (mutexes, atomics) translate — only messages over an unreliable network. Assume execution can stop mid-function while the rest of the world moves on.
- **Reliable from unreliable, up to a limit**: TCP over IP, error-correcting codes over noisy channels. The higher layer removes some fault classes (loss, duplication, reordering) but not others (delay) — know which faults your abstraction actually absorbs.
- **Cloud vs supercomputer philosophy**: HPC checkpoints and escalates partial failure to total failure (stop everything, restart from checkpoint); internet services must tolerate failed nodes while serving — at scale, "something is always broken," so giving up is not a strategy.

## Anti-patterns
- Comparing a lease expiry set by a remote clock against the local time-of-day clock — clock skew silently breaks the lease protocol.
- Trusting a client's own belief that it holds a lock/lease/leadership without server-side verification (split-brain writers).
- LWW conflict resolution on time-of-day timestamps — causally later writes silently dropped.
- Timeouts as constants picked by intuition rather than from measured delay distributions.
- Assuming a node is either working or dead — "limping" nodes (a Gigabit NIC dropping to 1 Kb/s from a driver bug) are worse than cleanly failed ones.
- Relying on synchronized clocks without monitoring clock offsets — a drifting clock causes silent data loss, not a crash, so evict nodes whose clocks drift too far.
- Leaving network-fault handling undefined/untested: clusters have deadlocked permanently and even deleted all data on netsplit (Elasticsearch, Redis Sentinel incidents). Deliberately inject faults (chaos-monkey style).

## Worked Example: Fencing tokens (the lease + GC pause corruption)
Scenario: a file in a storage service must be written by one client at a time, enforced by leases from a lock service (HBase actually had this bug). Client 1 acquires the lease, then hits a stop-the-world GC pause longer than the lease. The lease expires; client 2 acquires it and writes. Client 1 wakes, still believing its lease is valid (nothing told the paused thread time passed), writes too — the writes clash and corrupt the file. Client-side lease checking cannot fix this: the check itself can be stale by the time the write lands.

Fix: the lock service returns a **fencing token** with every grant — a number that increases monotonically with each grant (ZooKeeper's `zxid` or node version `cversion` work). Every write must carry its token, and the **storage service itself** checks and rejects any write whose token is lower than one already processed. Client 1 holds token 33, client 2 gets 34 and writes; client 1's late write with 33 is rejected. Required properties: *uniqueness* and *monotonic sequence* (safety — must never break), *availability* (liveness). Key insight: the resource takes an active role — never assume clients are well-behaved. Limit: fencing stops nodes acting in error, not nodes deliberately lying with a forged token — that's Byzantine territory.

## Key Takeaways
1. Partial failure is the defining property of distributed systems: operations non-deterministically fail, go slow, or vanish — and you may never learn whether they succeeded.
2. No response tells you nothing about why; timeouts are the only failure detector and they conflate node death with network trouble.
3. Never use time-of-day clocks for elapsed time or event ordering; monotonic clocks for durations, logical clocks for ordering, and treat any wall-clock reading as a confidence interval.
4. Any thread can pause arbitrarily long at any point (GC, VM suspend, paging) and resume unaware — protect resources server-side with fencing tokens, not client-side lease checks.
5. A node's self-assessment is worthless; truth is a quorum decision, and outvoted nodes must step down.
6. Design against an explicit system model (usually partially synchronous + crash-recovery); require safety always, liveness conditionally.
7. If you can stay on a single machine, do — distribution is justified by fault tolerance and latency (data near users), not fashion; but bounded-delay networks and hard real-time responses are possible only at utilization/cost most systems won't pay.

## Problems This Solves (mapped to real system-design examples)

| Mechanism | Problem it solves | Example |
|---|---|---|
| Named clock-sync assumption ("NTP-synchronized clocks") | ID generator must produce roughly time-ordered IDs without coordination — and must own up to what happens when clocks drift | Unique-ID generator (Snowflake's 41-bit timestamp) |
| GC-pause / tail-latency awareness | A stop-the-world pause at the wrong moment blows the p99 SLA | Stock exchange (explicit p99-latency + GC-pause callout) |
| Heartbeat + tuned timeout for liveness | Distinguish "offline" from "just slow" without false-positive churn | Chat system presence; metrics monitoring's pull-model scrape doubling as health check |
| Short timeouts + defense against indistinguishable-slow-vs-dead servers | Crawler must not let one unresponsive host or spider trap stall the whole frontier | Web crawler |
| Accepting retry ≠ knowing failure (can't tell a failed send from a slow one) | Third-party API call outcome is genuinely unknown until it resolves | Notification system (APNS/FCM/SendGrid retries) |
| Fencing tokens on the resource, not client-side lease checks | A paused leader/lock-holder waking up late must not corrupt state | Underpins every leader-election use in ch09's examples (S3 placement service, digital wallet, stock exchange) |

## Connects To
- **Ch 5 (Replication)**: leader failover, replication lag, and LWW conflicts are where these faults first bite.
- **Ch 7 (Transactions)**: snapshot isolation's monotonic transaction IDs are what Spanner's TrueTime approximates across datacenters.
- **Ch 9 (Consistency & Consensus)**: the payoff — quorums, consensus, and total ordering are the algorithms that provide guarantees within the partially-synchronous/crash-recovery model this chapter motivates.
