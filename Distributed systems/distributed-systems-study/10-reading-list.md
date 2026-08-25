# 10 — Reading List

> Part of the [Distributed Computing Study Pack](README.md). Each entry says **what it is** and
> **why to read it** for the goals of this pack. Names and titles below were verified where the
> source was approximate; verification notes are inline and summarized in the
> [README](README.md#verification--flagged-items).

## Books

### Designing Data-Intensive Applications — Martin Kleppmann (O'Reilly)
The single best map of this whole territory: replication, partitioning, transactions,
consistency models, batch vs stream processing, and consensus — all vendor-neutral. Read it to
turn the lenses in [01](01-taxonomy.md) and the topologies in [04](04-coordination-and-topology.md)
into deep, first-principles understanding.
*Verification:* the source said "Klubman/Klubmann" — that is a transcription of **Kleppmann**.
Corrected here.

### Database Internals — Alex Petrov (O'Reilly)
A two-part deep dive: Part I on storage engines (B-trees, LSM-trees, columnar layouts) and
Part II on distributed systems (replication, consensus, failure detection). Read it alongside
[05](05-databases-and-wire-protocols.md) to understand *why* a columnar OLAP engine like
ClickHouse aggregates the way it does, and to make heartbeats/consensus concrete.
*Verification:* the source said "Database Systems by Alex Petrov" — the correct title is
**Database Internals**. Corrected here.

### Patterns of Distributed Systems — Unmesh Joshi (Addison-Wesley Signature Series / Fowler, 2023)
A pattern catalog (Write-Ahead Log, Leader and Followers, Heartbeat, Quorum, Lease, HLC, etc.)
observed across real open-source systems, developed with Martin Fowler. Read it to put names to
the mechanisms in [04](04-coordination-and-topology.md) and [06](06-case-studies.md) — it is the
"design patterns" book for this domain.
*Related:* the patterns first appeared as articles on **martinfowler.com** (Fowler's bliki);
browsing the "Patterns of Distributed Systems" series there is a free on-ramp.
*Verification:* author **Unmesh Joshi**, Addison-Wesley Signature Series (Fowler), published
2023 — confirmed.

## Courses

### MIT 6.824 — Distributed Systems (Prof. Robert Morris)
The canonical graduate course: ~20 paper-driven lectures (GFS, Raft, Spanner, ZooKeeper, etc.)
and hands-on labs implementing a fault-tolerant, replicated key-value store **in Go** (including
Raft). Do the labs, not just the lectures — building Raft is the fastest way to internalize
leader election, heartbeats, and consensus from [04](04-coordination-and-topology.md). Materials
are public at the MIT PDOS course site (and an older 6.824 offering is on MIT OpenCourseWare).
*Verification:* taught by **Robert Morris** (Robert Tappan Morris), labs in Go — confirmed.

### NPTEL — Distributed Systems (Prof. Rajiv Misra, IIT Patna)
A free lecture series covering logical time, mutual exclusion, leader election, consensus, and
cloud/big-data topics (GFS/HDFS/Spark). A good structured, exam-style complement to 6.824 if you
prefer guided lectures. Available on the NPTEL platform.
*Verification:* instructor is **Prof. Rajiv Misra of IIT Patna** (he earned his PhD at IIT
Kharagpur — the likely source of the "Roorkee/Kharagpur/Patna" confusion in the original talk).
The instructor and institution are confirmed; if a specific *offering/course number* matters to
you, confirm it on the current NPTEL listing, as offerings are re-run under different codes.

## Papers (foundational, freely available)

### MapReduce: Simplified Data Processing on Large Clusters — Dean & Ghemawat (Google, 2004)
The origin of the map/reduce model in [02](02-data-processing-frameworks.md). Read it to see how
purity + re-runnable tasks buy fault tolerance on commodity hardware.

### The Google File System — Ghemawat, Gobioff, Leung (2003)
The storage layer MapReduce assumed; the intellectual ancestor of HDFS.

### Resilient Distributed Datasets (RDDs) — Zaharia et al. (Spark, NSDI 2012)
Why Spark uses **lineage** for fault tolerance instead of replication, and how laziness enables
optimization — the theory behind [02](02-data-processing-frameworks.md).

### In Search of an Understandable Consensus Algorithm (Raft) — Ongaro & Ousterhout (2014)
The consensus protocol behind Kafka's **KRaft** and many witnesses in
[04](04-coordination-and-topology.md). Deliberately written to be understandable — a great first
consensus paper before Paxos.

### The Reactive Manifesto & Reactive Streams
For the reactive/backpressure model referenced in [01](01-taxonomy.md) and
[03](03-functional-programming-foundations.md): the Reactive Manifesto (responsive, resilient,
elastic, message-driven) and the Reactive Streams spec (the interop standard behind Project
Reactor, Akka Streams, RxJava).

## Reference docs (for grounding, not narrative)

- **Apache Spark** docs — Structured Streaming (micro-batch), the SQL/DataFrame API, and the
  tuning guide (broadcast, `mapPartitions`, Arrow/pandas UDFs).
- **Apache Kafka** docs / **KIP-500** — the ZooKeeper → KRaft transition.
- **ClickHouse** docs — `Distributed` and `Replicated*` table engines, ClickHouse Keeper,
  sharding vs replication, native vs HTTP interfaces.
- **Microsoft Reactive Extensions (Rx)** and **LINQ** docs — the pull (`IEnumerable`) vs push
  (`IObservable`) duality behind the analogy in [02](02-data-processing-frameworks.md).

## A suggested order

1. Skim **DDIA** chapters on replication/partitioning/stream processing (breadth).
2. Do **MIT 6.824** Lab 2 (Raft) (depth on consensus).
3. Read the **MapReduce** and **RDD** papers (data-processing lineage).
4. Read **Patterns of Distributed Systems** as a pattern reference while building.
5. Dip into **Database Internals** Part II when the storage/consensus mechanics get concrete.
