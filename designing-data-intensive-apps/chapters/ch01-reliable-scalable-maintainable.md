# Chapter 1: Reliable, Scalable, and Maintainable Applications

## Core Idea
Data-intensive apps are composed from general-purpose building blocks (databases, caches, search indexes, stream/batch processors), and once you stitch them together you become a data system designer whose three governing concerns are reliability, scalability, and maintainability.

## Frameworks Introduced
- **The Three Concerns (Reliability / Scalability / Maintainability)**: Kleppmann's frame for evaluating any data system.
  - When to use: any system design discussion — use it as the top-level checklist before diving into technology choices.
  - How: (1) Reliability — the system works correctly (right function, desired performance) even under adversity: hardware faults, software faults, human error. (2) Scalability — as load grows (data volume, traffic, complexity), there are reasonable ways to cope; never a yes/no label. (3) Maintainability — many people over time (engineering + ops) can work on it productively, via operability, simplicity, evolvability.
  - Why it works / failure mode: it separates orthogonal risks so you don't over-invest in one (e.g., scaling machinery) while ignoring another (e.g., ops toil). Failure mode: treating them as buzzwords instead of asking the concrete questions behind each.

- **Load parameters → performance → coping strategies (the scalability method)**: You cannot say "X is scalable"; you must ask "if the system grows in a particular way, what are our options for coping?"
  - When to use: any capacity/scaling question, especially in interviews — state load parameters first.
  - How: (1) Describe current load with a few numbers — **load parameters** (requests/sec, read/write ratio, active users in a chat room, cache hit rate, follower distribution). Pick the ones your bottleneck depends on; sometimes a few extreme cases dominate, not the average. (2) Describe performance: throughput for batch systems, **response time distribution** (percentiles, not means) for online systems. (3) Ask two questions: with resources fixed, how does performance degrade as a load parameter grows? And how much resource must you add to hold performance constant? (4) Choose coping approach: scale up vs scale out, elastic vs manual.
  - Why it works / failure mode: architectures are built around assumptions of which operations are common vs rare — the load parameters. Wrong assumptions = scaling effort wasted or counter-productive. Expect to rethink architecture roughly every order-of-magnitude load increase.

- **Operability / Simplicity / Evolvability (the maintainability triad)**: three design principles to avoid creating legacy software.
  - When to use: reviewing designs for long-term cost — most software cost is maintenance, not initial development.
  - How: Operability = make routine ops easy (visibility/monitoring, automation support, no single-machine dependency, good defaults with overrides, predictable behavior). Simplicity = remove **accidental complexity** (complexity not inherent in the problem, only in the implementation — Moseley & Marks) via good **abstractions** (e.g., SQL hides on-disk structures, concurrency, crash recovery). Evolvability = make change easy; agility at the data-system level, tightly linked to simplicity.

## Key Concepts
- **Fault vs failure**: a fault is one component deviating from spec; a failure is the whole system stopping to provide service — design fault-tolerance so faults don't become failures.
- **Fault-tolerant / resilient**: anticipates and copes with certain types of faults (never all possible faults).
- **Load parameters**: the few numbers that describe load on your system; the right ones depend on your architecture.
- **Fan-out**: number of requests to other services needed to serve one incoming request (borrowed from electronics).
- **Response time vs latency**: response time is what the client sees (service time + network + queueing delays); latency is the time a request waits to be handled.
- **Percentiles (p50/p95/p99/p999)**: response-time thresholds that a given fraction of requests beat; median (p50) = typical, high percentiles = tail experience.
- **Head-of-line blocking**: a few slow requests hold up processing of subsequent ones due to limited parallelism, inflating client-observed response times.
- **Tail latency amplification**: with multiple parallel backend calls per user request, the request waits for the slowest call, so even rare slow backend calls make many end-user requests slow.
- **Scaling up vs scaling out**: vertical (bigger machine) vs horizontal (**shared-nothing** across machines); good architectures pragmatically mix both.
- **Accidental complexity**: complexity from implementation, not the problem itself — the target of simplification.

## Mental Models
- Think of reliability as "continuing to work correctly even when things go wrong" — then enumerate the fault classes: hardware (random, uncorrelated), software (systematic, correlated across nodes, dormant until an unusual trigger), human (config errors are the leading cause of outages; hardware only 10–25%).
- Deliberately increase the fault rate (Netflix Chaos Monkey, randomly killing processes) — many critical bugs are poor error handling, and only exercised fault-tolerance machinery is trusted machinery.
- Use percentiles, never averages, for response time: the mean doesn't tell you how many users experienced a delay. And never average percentiles across machines/time — aggregate by adding histograms.
- Think of a write-time vs read-time trade-off dial: when reads outnumber writes by orders of magnitude, precompute at write time (fan-out) even though it multiplies write work.

## Anti-patterns
- **Attaching "scalable" as a one-dimensional label**: meaningless without stating which load parameter grows and what the coping options are.
- **Reporting mean response time**: hides the tail that your most valuable users hit (Amazon: slowest requests often come from customers with the most data/purchases).
- **Averaging percentiles**: mathematically meaningless; add histograms instead (use forward decay, t-digest, or HdrHistogram for cheap ongoing computation).
- **Load-testing with a client that waits for each response before sending the next**: artificially shortens queues and skews measurements; the load generator must send independently of response time.
- **Chasing extreme tails blindly**: Amazon targets p999 but judged optimizing p9999 too expensive — tail costs rise while benefits diminish (random events outside your control dominate).
- **Interfaces so restrictive people work around them**: minimizing human error via lockdown backfires; balance "easy to do the right thing" against escape hatches.
- **Premature scaling in an unproven product**: iterating on features usually beats scaling for hypothetical load; architecture built on wrong load-parameter assumptions is wasted or counter-productive effort.

## Worked Example
**Twitter home timelines (Nov 2012 data).** Load: post tweet averages 4.6k req/s (12k peak); home-timeline reads 300k req/s. 12k writes/sec alone is easy — the challenge is **fan-out** (each user follows and is followed by many).
- **Approach 1 (read-time join)**: posting inserts into a global tweets collection; reading a timeline joins tweets from everyone the user follows. Twitter's first version — struggled under home-timeline read load.
- **Approach 2 (write-time fan-out)**: keep a per-user timeline cache (mailbox); posting inserts the tweet into every follower's cache, making reads cheap. Works because tweet rate is ~two orders of magnitude below read rate — do more work at write time, less at read time. Cost: average 75 followers turns 4.6k tweets/sec into 345k cache writes/sec, and the distribution is wildly skewed — some users have >30M followers, so one tweet can mean >30M writes, against a ~5-second delivery target.
- **Hybrid (final)**: fan out for most users, but exempt celebrities; their tweets are fetched at read time and merged in — consistently good performance. Key lesson: the follower distribution per user (weighted by tweet frequency) is the decisive load parameter.

## Key Takeaways
1. Design fault-tolerance so component faults don't become system failures; you can only tolerate certain fault types, so name them (hardware, software, human).
2. Prefer software fault-tolerance over pure hardware redundancy at scale — with 10k disks (MTTF 10–50 years) expect roughly one disk death per day, and cloud VMs vanish without warning; machine-loss tolerance also enables rolling patches with zero downtime.
3. Guard against humans, the leading outage cause: minimize error opportunities via good abstractions, decouple sandboxes (real data, no real users) from production, test at all levels, make rollback fast and rollouts gradual, and instrument telemetry.
4. Start every scalability discussion by naming load parameters; know whether averages or extreme cases drive your bottleneck.
5. Measure response time as a distribution on the client side; set SLOs/SLAs in percentiles (e.g., median < 200 ms and p99 < 1 s, up 99.9% of the time). 100 ms slower cost Amazon 1% of sales; a 1 s slowdown cuts a satisfaction metric 16%.
6. Keep stateful data on a single node (scale up) until cost or HA forces distribution; stateless services distribute easily, stateful ones add real complexity. There is no magic scaling sauce — 100k req/s of 1 kB differs completely from 3 req/min of 2 GB despite equal throughput.
7. Optimize for maintenance, the majority of software cost: build in operability, cut accidental complexity with abstractions, and design for evolvability.

## Connects To
- **Ch 2**: data models and query languages are the first layer of abstraction chosen when composing a data system.
- **Ch 5–6 (Replication/Partitioning)**: the scale-out / shared-nothing path introduced here; rebalancing partitions revisits elastic vs manual scaling.
- **Part III (Derived data)**: the Figure 1-1 composite system (DB + cache + index + queue kept in sync by application code) is the problem those chapters systematize.
- **The Tail at Scale (Dean & Barroso)**: the canonical treatment of tail-latency amplification across parallel backend calls.
- **SRE practice**: SLOs/SLAs, telemetry, chaos engineering, and gradual rollouts are the operational embodiment of this chapter's reliability and operability principles.
