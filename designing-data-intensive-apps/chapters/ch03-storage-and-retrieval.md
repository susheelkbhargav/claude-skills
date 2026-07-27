# Chapter 3: Storage and Retrieval

## Core Idea

Storage engines split along two axes. First: **log-structured** (append-only, never modify in place — Bitcask, LSM-trees, LevelDB, RocksDB, Cassandra, HBase, Lucene) vs. **update-in-place** (fixed-size pages overwritten on disk — B-trees, standard in virtually every relational database). Second: **OLTP** (many small key lookups; disk *seek time* is the bottleneck) vs. **OLAP** (few huge scans; disk *bandwidth* is the bottleneck — where column-oriented storage wins). And one universal trade: well-chosen indexes speed up reads, but **every index slows down writes** — which is why you choose indexes manually from your query patterns.

## Frameworks Introduced

**LSM-tree (Log-Structured Merge-Tree)** — *When to use:* write-heavy workloads needing high random-write throughput. *How:* writes go to an in-memory balanced tree (the **memtable**); past a threshold (a few MB) it's flushed to disk as an **SSTable** (Sorted String Table — key-sorted, each key once per segment); reads check memtable, then segments newest-to-oldest; background **compaction** merges segments mergesort-style, keeping only the newest value per key and honoring tombstones (deletion records); a separate unsorted append-only log restores the memtable after a crash. *Why it works:* it systematically turns random writes into sequential writes — much faster on both spinning disks and SSDs; sorting enables sparse in-memory indexes (one key per few KB), efficient merging, block compression, and range queries. *Failure modes:* lookups of nonexistent keys check every segment down to the oldest — fixed with **Bloom filters** (memory-efficient set approximation that says a key definitely isn't there, skipping disk reads); compaction competes with live requests for disk, so high-percentile latency can spike.

**B-tree** — *When to use:* the default; read-heavy or mixed workloads, and anywhere strong transactional semantics matter. *How:* break the database into fixed-size pages (traditionally 4 KB), forming a balanced tree; each page holds k keys and k+1 child references (branching factor in the hundreds), height O(log n); updates overwrite the leaf page in place; a full page splits into two half-full pages and the parent is updated. *Why it works:* pages match the hardware's block structure; every key exists in exactly one place, so range locks for transaction isolation attach directly to the tree. *Failure modes:* a crash mid-split corrupts the index (orphan pages) — hence the **write-ahead log (WAL/redo log)**: every modification is appended to the WAL before touching pages, so every write hits disk at least twice; in-place mutation requires latches (lightweight locks) for concurrent access.

## Key Concepts

- **Write amplification**: one logical write → multiple physical disk writes. B-trees pay it via WAL + page (+ splits); LSM-trees pay it via repeated background rewriting during compaction. Especially costly on SSDs, which wear out after limited overwrites. Kleppmann's verdict: neither side clearly wins — benchmark your own workload.
- **LSM vs. B-tree rule of thumb**: LSM-trees faster for writes, B-trees thought faster for reads and more *predictable* (no compaction pauses; one copy per key). Benchmarks are inconclusive and workload-sensitive.
- **Secondary index**: keys not unique; store either a list of row IDs per key (postings list) or append a row ID to make keys unique. Both B-trees and LSM structures work.
- **Heap file vs. clustered vs. covering index**: heap file = rows stored unordered, indexes hold references (avoids duplication across indexes); **clustered index** = row stored inside the index (InnoDB primary key; its secondary indexes point at the primary key, not a heap location); **covering index** = some columns stored in the index so certain queries are answered from the index alone. Clustered/covering speed reads at the cost of storage and write overhead.
- **Concatenated (multi-column) index**: phone-book ordering (lastname, firstname) — great for prefix queries, useless for the second column alone. 2D range queries (lat/long) need space-filling curves or R-trees.
- **OLTP vs. OLAP**: OLTP — small number of records per query fetched by key, random low-latency writes, end users, latest state, GB–TB. OLAP — aggregates over huge scans, bulk ETL loads, internal analysts, history of events, TB–PB.
- **Data warehouse + star schema**: read-only copy of all OLTP systems, loaded via **ETL** (Extract-Transform-Load). Center: **fact table** (one row per event; often 100+ columns; big enterprises hold tens of petabytes) surrounded by **dimension tables** (the who/what/where/when/why). **Snowflake schema** = further-normalized dimensions; star preferred for analyst simplicity.
- **Column-oriented storage**: store each column's values in its own file, same row order in every file. A query touching 4–5 of 100+ columns reads only those files. Compresses aggressively — **bitmap encoding** per distinct value plus **run-length encoding**; a sorted low-cardinality column can compress billions of rows to a few KB — and enables vectorized processing (tight loops over L1-cache-sized compressed chunks, bitwise AND/OR directly on bitmaps). Weak spot is writes: you can't insert into compressed sorted columns in place — the fix is the LSM idea again (buffer in memory, bulk-merge into new column files; Vertica does exactly this).
- **Materialized views / data cubes**: precomputed aggregate grids (SUM by date × product × …) make specific queries instant but lose flexibility (can't query a non-dimension attribute); warehouses keep raw data and use cubes only as a boost.

## Mental Models

- **Use append-only logs when you want fast, crash-safe writes**: sequential I/O beats random I/O on every medium; immutable segments make concurrency and recovery trivial (no half-overwritten values).
- **Use the index-cost lens when tuning**: every index is a read/write trade. Ask "what query pattern pays for this index?" before adding it.
- **Use "seek time vs. bandwidth" to classify a workload**: if the bottleneck is finding records (seeks) → OLTP engine, indexes matter. If it's moving bytes (bandwidth) → columnar, compression matters, indexes largely don't.
- **Use B-trees when predictability and transactions dominate; LSM when write throughput dominates** — then verify empirically, because "there is no quick and easy rule."

## Anti-patterns

- Ad-hoc analytics on the production OLTP database — huge scans wreck latency for concurrent transactions; the whole reason warehouses exist.
- Hash indexes when keys don't fit in RAM, or for range queries (Bitcask's constraint; a range scan degenerates to one lookup per key).
- Row-oriented storage for wide fact tables — loading 100+ attribute rows to use 3 columns burns disk bandwidth.
- Indexing everything "just in case" — pure write overhead.
- Treating LSM-vs-B-tree benchmarks as portable — results are workload- and tuning-sensitive; test with *your* workload.

## Worked Example

Start with the world's simplest database — two bash functions:

```bash
db_set () { echo "$1,$2" >> database; }
db_get () { grep "^$1," database | sed -e "s/^$1,//" | tail -n 1; }
```

Writes are O(1) appends (it's a log); reads are O(n) scans. **Step 1 — hash index (Bitcask model):** in-memory hash map of key → byte offset; reads become one seek. To stop the log growing forever, split into segments and run **compaction**: keep only the newest value per key, merge segments in a background thread while old segments serve reads, atomically swap. Limits: all keys must fit in RAM; no range queries. **Step 2 — SSTables:** require segments sorted by key. Merging becomes streaming mergesort (works even when files exceed memory; when a key appears in two segments, the more recent segment wins), the index goes sparse (one key per few KB, scan the gap), blocks compress. **Step 3 — LSM-tree:** to get sorted segments from unsorted writes, buffer in a memtable, flush to SSTable at a few MB, keep a crash-recovery log, compact in the background, add Bloom filters for missing keys. This is LevelDB/RocksDB/Cassandra/HBase (terms from Google's Bigtable paper; the structure from O'Neil's LSM-tree) — and the same idea, applied to column files, is how columnar warehouses accept writes.

## Key Takeaways

1. Sequential beats random: log-structured engines win writes by turning random writes into sequential ones.
2. Indexes trade write speed for read speed — choose them from query patterns, never by default.
3. B-tree = one copy per key, in-place updates, WAL, predictable, transaction-friendly. LSM = append-only, compaction, higher write throughput, compaction-induced tail latency. Both suffer write amplification; benchmark to decide.
4. OLTP is seek-bound (index by key); OLAP is bandwidth-bound (scan compactly). Separate the systems.
5. Columnar storage + bitmap/run-length compression + sort order is the OLAP answer; its write path reuses the LSM idea.
6. Star schema (fact + dimension tables) is the near-universal analytics model; ETL feeds it. Keep raw data; cubes are only a boost.

## Problems This Solves (mapped to real system-design examples)

| Mechanism | Problem it solves | Example |
|---|---|---|
| LSM write path (commit log → memtable → SSTable) + Bloom filter reads | Write-heavy key-value workload needs high throughput without random-write cost | Leaderless key-value store |
| Bloom filter as an existence check before a point lookup | Avoid a wasted lookup on a huge sparse key space | URL shortener (`shortURL → longURL`) |
| In-memory index rebuilt at startup, no incremental persistence | Structure only makes sense in memory (quadtree over live location data); rebuild is cheap, incremental update isn't | Proximity service |
| Purpose-built time-series engine: delta-of-delta encoding + downsampling | Metric points are mostly-monotonic timestamps with tiny deltas — generic storage wastes bytes at this volume | Metrics monitoring & alerting |
| Sorted set (hashmap + skip list) for O(log n) rank | Relational nested-count-query for "what's my rank" doesn't scale past one table | Real-time leaderboard |
| LSM justification for a custom write-heavy index | Sequential-write engine needed for a bespoke search structure, same reasoning as Cassandra/RocksDB | Distributed email service (custom search engine) |
| mmap append log + embedded LSM (RocksDB) + periodic snapshots | Need both a durable sequential log and fast point reads on the same node | Digital wallet |
| WAL segments exploiting sequential disk + OS page cache | Broker must sustain high write throughput without seek-bound I/O | Distributed message queue |

## Connects To

- **Ch 2 (Data Models)**: same question from the database's side — Ch 2 was how you hand data over; Ch 3 is how the engine stores and finds it.
- **Ch 4 (Encoding)**: on-disk formats (SSTable layouts, Parquet's Dremel-derived columnar format) are encoding decisions.
- **Ch 7 (Transactions)**: B-trees' one-place-per-key property lets range locks attach to the tree; copy-on-write B-trees (LMDB) feed into snapshot isolation.
- **Ch 10–11 (Batch/Stream)**: ETL is batch processing; the append-only log reappears as the backbone of stream systems.
- Interview hooks: "design a KV store" → walk the db_set → hash index → SSTable → LSM ladder; "why is Cassandra write-fast?" → sequential I/O + memtable; "why is Redshift scan-fast?" → columnar + compression.
