# Chapter 10: Batch Processing

## Core Idea

Batch jobs read a bounded, immutable input, produce a completely new output, and have no other side effects. This one discipline — derived output, immutable input — buys you fault tolerance (retry any task safely), human fault tolerance (bad code? re-run against the same input), and composability (any job's output can feed any future job). MapReduce is "Unix distributed": HDFS is the filesystem, a job is a process, and the shuffle is the always-on `sort` between map and reduce. Everything else in the chapter is join algorithms and the engines that made the sort/materialization optional.

## Frameworks Introduced

**MapReduce + HDFS (Hadoop)**
- *When to use*: huge bounded datasets, arbitrary code (ML feature engineering, index building, ETL into a warehouse), multitenant clusters where tasks get preempted, jobs long enough that at least one task will fail.
- *How*: input directory in HDFS is split by file block; one map task per block, scheduled near the data ("putting the computation near the data"). Mapper extracts key-value pairs; framework partitions by hash of key, sorts each partition to local disk (SSTable-style), reducers fetch and merge — the **shuffle**. Reducer sees each key with an iterator over all its values, writes output to HDFS. Chained jobs = workflows, wired by directory names, orchestrated by schedulers (Airflow, Oozie, Luigi).
- *Why it works*: stateless callbacks + immutable inputs mean the framework can retry any failed task and discard partial output; job output is all-or-nothing. This was designed for Google's mixed-priority datacenters where a 1-hour task had ~5% chance of preemption — aggressive disk materialization is a bet that failure is frequent.
- *Failure mode*: fully materializing intermediate state to HDFS between every job. Downstream jobs wait for every straggler in the upstream job; redundant mapper stages just re-read what a reducer wrote. Slow in the failure-free case.

**Dataflow engines: Spark, Flink, Tez**
- *When to use*: multi-stage workflows where intermediate state is throwaway; iterative/interactive work; anywhere MapReduce would chain 50+ jobs (recommendation pipelines).
- *How*: whole workflow is one job, a DAG of **operators** (generalized map/reduce). Sorting only where needed; partitioned-without-sort and broadcast connections available; intermediate state in memory or local disk, streamed operator-to-operator pipe-style. Tez is a thin library on YARN's shuffle service; Spark and Flink are full frameworks.
- *Why it works / failure mode*: fault tolerance via **recomputation** instead of durability — Spark tracks lineage with RDDs, Flink checkpoints operator state. This only works cleanly if operators are **deterministic**; hash-iteration order, random numbers, or clock reads silently break recovery and force killing downstream operators too. If intermediate data is small or compute is expensive, materializing beats recomputing.

## Key Concepts

1. **Unix philosophy**: do one thing well; expect your output to be another program's input; immutable inputs; separate logic from wiring (stdin/stdout, pipes). `sort` spills to disk and parallelizes — the pipeline scales past memory.
2. **Uniform interface**: in Unix it's a file (byte stream, usually newline-separated ASCII); in Hadoop it's HDFS files (Avro/Parquet replace fragile text parsing).
3. **The shuffle**: partition mapper output by hash of key, sort, copy to reducers, merge. Deterministic, no randomness despite the name. It is what "brings related data to the same place" — the mapper's key is a destination address.
4. **Reduce-side join (sort-merge)**: mappers over both inputs emit join key; shuffle co-locates them; secondary sort puts the dimension record (e.g. user profile) first so the reducer holds one record in memory while streaming the facts. No assumptions about input; pays full shuffle cost.
5. **Map-side joins**: no reducers, no sort. **Broadcast hash join** — small input loaded into an in-memory hash table in every mapper of the large input (Hive "MapJoin", Pig "replicated join"). **Partitioned hash join** — both inputs pre-partitioned identically (same key, hash, partition count); each mapper hashes only its partition (Hive "bucketed map join"). **Map-side merge join** — inputs also sorted: stream-merge without memory limits.
6. **Skew handling**: linchpin objects (celebrities) overload one reducer, and the job is only as fast as its slowest reducer. Fix: send the hot key's records to random reducers and replicate the other join side to all of them (Pig/Hive *skew join*), or aggregate in two stages for GROUP BY.
7. **Batch output = derived data**: search indexes (Google's original use — documents in, immutable index files out), or read-only key-value stores built inside the job and bulk-loaded (Voldemort, Terrapin, ElephantDB, HBase bulk load). Never write record-at-a-time to an external DB from a job: it's slow, hammers the DB, and leaks side effects that break the all-or-nothing guarantee.
8. **MapReduce vs MPP databases**: MPP = monolithic, schema-first, SQL, in-memory, abort-and-retry whole queries. Hadoop = raw byte storage ("data lake", sushi principle: raw data is better, schema-on-read), arbitrary code, task-level retry, disk-eager. They're converging: Hive/Spark grew cost-based optimizers, columnar reads, vectorized execution.
9. **Pregel / BSP** for graphs: vertex holds state across iterations, sends messages along edges, delivered in the next superstep; checkpointed for fault tolerance. But if the graph fits on one machine, single-node processing usually wins — cross-machine message traffic dominates.
10. **Declarative creep**: specify joins relationally and the optimizer picks the join algorithm; keep callback UDFs for the code SQL can't express. Best of both.

## Mental Models

- **Hadoop = distributed Unix.** HDFS is the filesystem, a MapReduce job is a process that always pipes through `sort`, workflows are pipelines that write every stage to a temp file. Dataflow engines put the actual pipes back.
- **Keys are addresses.** Emitting `(key, value)` is sending a message; the shuffle is the delivery network. Joins, GROUP BY, sessionization, and Pregel are all the same move: bring related data to one place, then run simple single-threaded logic.
- **Fault tolerance is a bet on failure rate.** Materialize when tasks die often (preemption-heavy clusters); recompute when they don't. Determinism is the precondition for cheap recomputation.
- **Sorting vs hashing is the working-set question.** Fits in memory → hash table (Ruby script, broadcast join). Doesn't → sort-merge with sequential disk I/O (Unix `sort`, the shuffle). Same trade-off at every scale.

## Anti-patterns

- Querying a remote DB per record inside a job — network round-trips kill throughput and make the job non-deterministic. Snapshot the DB into the distributed filesystem instead.
- Writing job output directly into a live database — overwhelms it, and partial failures leave visible half-written state.
- Ignoring skew: one celebrity key makes the whole workflow wait on one reducer.
- Non-deterministic operators (hash-order iteration, unseeded RNG, clock reads) under recomputation-based recovery — corrupts downstream state, forces cascading restarts.
- Iterative graph algorithms as chained MapReduce jobs — every iteration rereads and rewrites the entire dataset even when almost nothing changed.
- Distributing a graph job that fits on one machine.

## Worked Example: log analysis, three ways

**Unix**: top-5 URLs from an nginx log:
`cat access.log | awk '{print $7}' | sort | uniq -c | sort -rn | head -5`
awk extracts the key (map), `sort` brings equal URLs adjacent (shuffle), `uniq -c` aggregates adjacent runs (reduce). Scales past RAM because `sort` spills sorted segments to disk and merges them — the SSTable trick. The in-memory alternative (a hash-table counter script) wins only while distinct URLs fit in memory.

**MapReduce**: same job, one machine → thousands. Mapper emits URL as key; framework sorts; reducer counts. The "top 5" needs a second job — a second sort stage = another MapReduce job chained via HDFS directory.

**Now join in user profiles** (which age groups view which pages). Activity events carry only `user_id`; profiles live in a user database.
- Default: **reduce-side sort-merge join**. One mapper set over events, one over the profile snapshot, both keyed by user_id; secondary sort delivers the profile record first; reducer holds one date-of-birth and streams events, emitting (url, viewer-age). Works on any input, costs a full shuffle of both sides.
- Profile table fits in memory → **broadcast hash join**: map-only job, each mapper loads all profiles into a hash table, streams events. No shuffle at all.
- Both inputs already partitioned by user_id from prior jobs → **partitioned hash join**: each mapper loads only its partition's profiles.
Decision rule: small side fits in memory → broadcast; co-partitioned inputs → partitioned hash; otherwise → sort-merge. This is exactly what Hive/Spark cost-based optimizers automate.

## Key Takeaways

1. The two problems every distributed batch framework solves are **partitioning** (bring related data together) and **fault tolerance** (retry safely) — everything else is optimization.
2. Immutable inputs + side-effect-free tasks give exactly-once *output* semantics: the final result equals a failure-free run even though tasks were retried. Stronger than anything an online service gets.
3. Know the three joins cold — sort-merge (no assumptions, full shuffle), broadcast hash (small side in memory), partitioned hash (co-partitioned inputs) — and the input properties that select each.
4. Physical layout is metadata that matters: partition count, key, and sort order of a dataset (Hive metastore/HCatalog) determine which join is even legal.
5. MapReduce materializes all intermediate state to HDFS; Spark/Flink/Tez treat the workflow as one DAG, keep intermediates local, and recover by recomputation — faster when failures are rare, dependent on determinism.
6. Batch output is derived data, rebuilt wholesale from systems of record. If it's wrong, fix the code and re-run — the "database inside the job, bulk-swap on serve" pattern (Voldemort) makes rollback trivial.
7. Batch input is *bounded*, so jobs finish. Remove that assumption and you get stream processing.

## Problems This Solves (mapped to real system-design examples)

| Mechanism | Problem it solves | Example |
|---|---|---|
| Hash-dedup pipeline over a huge derived corpus | "URL seen? / content seen?" at 1B pages/month, 30PB corpus — can't check this online per-request | Web crawler |
| DAG scheduler + retry-from-persisted-intermediate (MapReduce failure model) | A long multi-stage pipeline (transcoding) must resume from where it failed, not restart from scratch | Youtube-style video transcoding |
| Batch-rebuilt immutable read-only index artifact, swapped in atomically | Real-time updates to the serving structure are infeasible at this write volume; rebuild wholesale instead | Search autocomplete (weekly trie rebuild) |
| Offline pipeline transforming raw data into an immutable, multi-resolution derived artifact | TB-scale unstructured input must become a queryable structure without live-updating it in place | Google Maps (routing tiles baked offline, served from S3) |
| Backfill / reconciliation batch job (Lambda vs Kappa) | Stream processing alone can't cheaply fix historical mistakes or handle very late data | Ad click event aggregation |
| Nightly settlement-file reconciliation as derived-data auditing | Verify a real-time system's ledger against an independent batch-computed source of truth | Payment system (secondary) |

## Connects To

- **Ch 3 (Storage)**: the shuffle's sort-spill-merge is SSTable/LSM machinery; sequential I/O beats random access at every layer.
- **Ch 4 (Encoding)**: Avro/Parquet as the typed replacement for Unix's untyped text interface.
- **Ch 6 (Partitioning)**: hash-of-key partitioning and hot-spot skew reappear verbatim in the shuffle.
- **Ch 11 (Stream Processing)**: same computations over unbounded input; jobs never complete; Pregel's message-passing and incremental index updates foreshadow it.
- **Ch 12 (Future)**: derived data vs systems of record becomes the organizing principle for whole architectures.
