# Drill Progress Tracker

**Status legend**: `untested` · `weak` (missed mechanism/direction, needed hints) · `solid`

## Resurface queue (drill these first next session, different scenario angle)
1. ch09 — linearizability vs serializability (no prior exposure, needs real study first)
2. ch03 — LSM-tree vs B-tree mechanism (no prior exposure, needs real study first)
3. ch07 — write skew (SI doesn't catch it) — got "disjoint rows" right, inverted the conclusion
4. ch10 — batch join selection (map-side vs reduce-side) — spotted size asymmetry, inverted which join it implies
5. ch02 — document vs graph — spotted many-to-many, wrong reasoning for why graph (variable-depth traversal is the real tell, not many-to-many alone)
6. ch08 — fencing tokens — mechanism landed, needed 2 hints to get there
7. ch06 — range partitioning hot spots — needed the definition explained from scratch
8. ch04 — forward vs backward compatibility direction — inverted which side breaks
9. ch01 — percentile definition + probability math for tail latency amplification (1-0.99^5, not 20%)
10. ch11 — dual-write/CDC — names solid, mechanism (WAL tail, offsets, idempotent replay) needed pulling

## Pattern flagged (recurring, not chapter-specific)
Candidate reliably **spots the correct distinguishing fact** but sometimes **inverts what it implies** (ch02, ch07, ch10). Worth calling out explicitly in depth round — the instinct for "what's the relevant signal" is good; the last inferential step needs slowing down.

## Item Status

| Ch | Topic | Status | Last Drilled | Note |
|---|---|---|---|---|
| 1 | Percentiles / tail latency amplification | weak | 2026-07-24 | Named tail latency amplification correctly; math wrong (said 20% chance for 5×p99=100ms, actual 1-0.99^5≈5%) |
| 2 | Document vs relational vs graph | weak | 2026-07-24 | MVP reasoning solid. "Many-to-one expensive in Postgres" is wrong — Postgres joins are cheap; real tell for graph is unbounded/variable-depth traversal |
| 3 | LSM-tree vs B-tree | weak | 2026-07-24 | No attempt ("not sure"). Full teach given: sequential vs random writes, memtable/SSTable/compaction, write-heavy→LSM rule |
| 4 | Encoding: backward vs forward compatibility | weak | 2026-07-24 | Picked wrong direction — thought old app crashes reading new field (forward compat, actually fine w/ JSON ignore-unknown-fields). Real break: new server treating new field as required breaks on old clients/data (backward compat) |
| 5 | Replication: read-your-writes | **solid** | 2026-07-24 | Named anomaly, picked mechanism (route own-content reads to leader), correctly reasoned the leader-load trade-off unprompted on second push |
| 6 | Partitioning: hash vs range, hot spots | weak | 2026-07-24 | Scatter-gather answer correct. Didn't know what range partitioning was — had to be taught before reaching the hot-spot-on-timestamp trap and compound-key fix |
| 7 | Transactions: write skew (doctors on-call) | weak | 2026-07-24 | Correctly identified disjoint rows = no dirty write, then inverted conclusion (thought SI would still block it). This is the book's canonical write-skew example — should be automatic |
| 8 | Distributed systems: fencing tokens | weak | 2026-07-24 | Needed 2 hints (GC-pause mechanism, then "check if lock active" ≠ fix) before landing on fencing tokens / storage-side token rejection |
| 9 | Linearizability vs serializability | weak | 2026-07-24 | No attempt ("not sure"). Full teach given — this is a top-tier senior interview distinction, needs dedicated study before next drill |
| 10 | Batch: map-side vs reduce-side joins | weak | 2026-07-24 | Spotted "lookup table is small" correctly, concluded reduce-side (backwards) — small side that fits in memory → map-side broadcast hash join |
| 11 | Streaming: dual-write problem / CDC | weak | 2026-07-24 | Named problem and fix unprompted (first clean answer of the session). Mechanism (WAL tail, log as single source of truth, idempotent replay via offsets) needed pulling |

## Session Log

- 2026-07-24: Full breadth round, ch01-ch11, one drill each. 1/11 solid (ch05), 10/11 weak. Candidate prepping for Anzenna AI (Jul 28-29, Go+system design), Kai cybersecurity (Aug 3, Go+system design+devops), Baselayer CTO (Aug 3, own project). Session paused for candidate to self-revise before depth round. Next session: start with resurface queue in priority order above — ch09 and ch03 need fresh material (no prior exposure), ch07/ch10/ch02 need the "spots signal, inverts conclusion" pattern addressed directly.
