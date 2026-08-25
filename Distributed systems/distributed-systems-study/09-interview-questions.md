# 09 — Interview Questions with Model Answers

> Part of the [Distributed Computing Study Pack](README.md). Each item has a **model answer**
> and a **"what they're really testing"** line. Practice speaking the answer aloud; the goal
> is to lead with mechanism and lenses, not product names.

## 1. "Walk me through how Spark runs a job."

**Model answer.** You write transformations (`map`, `filter`, `join`) which are **lazy** —
they build a **DAG** of steps but run nothing. When you call an **action** (`count`,
`collect`, `write`), the **driver** (master) uses the DAG to plan execution: the Catalyst
optimizer (for DataFrames/Datasets) fuses steps and pushes down filters, the DAG scheduler
splits the plan into stages at shuffle boundaries, and tasks are distributed to **executors**
(workers) that process **partitions** in parallel. If a partition is lost, Spark **recomputes**
it from **lineage** rather than replicating. It's a **master–slave, data-parallel, DAG** engine.

*What they're really testing:* whether you understand lazy evaluation, the DAG, and
lineage-based recovery — not whether you've memorized the API.

## 2. "Why can't I call a REST API inside a Spark `map`?"

**Model answer.** Transformations must be **referentially transparent** — a pure function of
their input — because Spark relies on that to **retry** failed tasks, run **speculative**
copies, reorder work, and **recompute** from lineage. A network call breaks all of those: it's
non-repeatable, can fail partially, and makes re-runs produce different effects. If you do it
anyway, Spark effectively **degrades to a scheduler** that runs your side effects an
unpredictable number of times. If you truly need per-record lookups, batch them at
`mapPartitions`, make them **idempotent**, broadcast static reference data, and accept you've
stepped outside the purity contract — or use a framework built for side effects (Storm).

*What they're really testing:* do you understand *why* purity is required (fault-tolerance
mechanisms), or do you just know the rule?

## 3. "You need zero-downtime for a stateful stream. Kafka, Spark Streaming, or Flink?"

**Model answer.** Derive it from **topology**. Kafka is **leader–follower per partition**:
followers continuously replicate the leader's log, so a failure is handled by **promoting an
in-sync follower** — recovery is near-instant and lossless because the replica is already
current. Spark/Flink achieve fault tolerance by **checkpointing state** and, on failure,
**restoring the snapshot and replaying**, which is heavier and can lose the window since the
last checkpoint. For strict zero-downtime on stateful streams, leader–follower replication is
the natural fit; I'd still tune Flink's checkpoints and hot-standbys if the processing logic
required Flink's stateful operators. The point is to match **recovery mechanism to
requirement**.

*What they're really testing:* can you reason from requirement → topology → tool, rather than
reciting a feature matrix.

## 4. "How does a cluster know a node has died?"

**Model answer.** By **heartbeats / periodic polling** with a timeout — a node that misses N
heartbeats within the session timeout is declared dead. Detection is **pull-based** because a
failed or partitioned node can't push a notification; a dead node and a slow node look
identical from outside. Declaring death triggers **leader election** through a witness
(ZooKeeper/KRaft in Kafka). The tuning trade-off: short timeouts fail over fast but produce
false positives on slow nodes; long timeouts are stable but recover slowly.

*What they're really testing:* whether you grasp that failure detection is fundamentally
polling, and the timeout trade-off.

## 5. "Why does leader election need ZooKeeper (or KRaft/Keeper)? Can't the nodes just vote?"

**Model answer.** Without an external source of truth you risk **split-brain**: after a network
partition, two nodes can each believe they're leader and both accept writes, corrupting state.
A consensus/witness service provides a single agreed answer to "who is leader," stores cluster
metadata, and issues an **epoch/term** so a stale old leader is fenced off. Kafka used
**ZooKeeper** for this and now offers **KRaft** (self-managed Raft metadata, no ZooKeeper);
ClickHouse uses ZooKeeper or **ClickHouse Keeper**.

*What they're really testing:* understanding of split-brain and why consensus is a separate
concern from the data path.

## 6. "Explain JDBC vs a wire protocol like TDS."

**Model answer.** **JDBC** is a **Call-Level Interface** — the function-call API your code uses
(connect, prepare, execute, iterate), whose main job is mapping language types ↔ driver types ↔
database types. The **wire protocol** (e.g., **TDS** for SQL Server, **SQL\*Net** for Oracle) is
the layer *below* the driver: how requests and result sets are encoded, framed, and compressed
on the established connection. The CLI is **portable** — the same JDBC code works across
databases — while the **wire protocol is what actually differs** and is where throughput/latency
are won or lost. The **driver manager** picks the concrete driver from the connection string.

*What they're really testing:* whether you can separate the API layer from the transport layer.

## 7. "When would a columnar database replace a Spark/Hadoop pipeline?"

**Model answer.** When the workload is **distributed aggregation over large tables** — reporting,
OLAP, dashboards. A sharded columnar DB like ClickHouse already does map/reduce internally: each
**shard** computes a **partial aggregate** (an associative **monoid**) and the initiator merges
them, co-locating compute with storage (**shared-nothing**), so there's no data movement to a
separate compute tier. You can front it with a proxy that speaks JDBC/ODBC so BI tools think
they're talking to a database while the DB does the distributed compute. The trade-off axis is
**storage-compute coupling**: ClickHouse couples them (fast, less elastic); Spark/Snowflake/
Fabric decouple them (elastic, but move data to compute). It's workload-shaped, not "always
better than Spark."

*What they're really testing:* recognizing distributed aggregation as a monoid and understanding
the coupling trade-off — depth beyond "ClickHouse is fast."

## 8. "Design the topic structure for per-user order events in Kafka."

**Model answer.** Use a **single topic** (e.g., `orders`) partitioned by a key, with the
**user id as the message key** so all of a user's events land on one partition (per-key
ordering) — *not* a topic per user. Topic-per-entity means **unbounded** metadata, partitions,
and file handles as users grow, which destabilizes the cluster. Reserve separate topics for
separate **schemas/semantics**, not separate **instances**. Size partitions for throughput and
consumer parallelism.

*What they're really testing:* whether you understand topic cost/cardinality and the role of the
message key in partitioning and ordering.

## 9. "What is backpressure and when do you need it?"

**Model answer.** Backpressure is a **consumer signaling a fast producer to slow down** so
buffers don't overflow and memory doesn't blow up. You need it in **data-in-space-time**
processing — unbounded event streams where producers can outpace consumers. Reactive-stream
libraries (Rx, Reactive Streams / Project Reactor, Akka Streams) make it first-class. Without
it, a fast source either drops data or OOMs the consumer.

*What they're really testing:* awareness that streaming isn't just "process as it arrives" —
flow control is a real concern.

## 10. "Contrast the actor model (Akka) with data-parallel processing (Spark)."

**Model answer.** **Akka/actors** are **task/computation parallelism**: isolated actors hold
private state and communicate by **async messages**, with supervision for failure — great for
concurrent, event-driven *applications and systems*, but there's no partitioned dataset or
lineage recovery. **Spark** is **data parallelism**: the same pure operation applied across
**partitions** of a dataset with DAG scheduling and lineage-based recovery — great for crunching
large static or micro-batched data. Different tools for different shapes of problem: message-
driven stateful logic vs bulk dataset transformation.

*What they're really testing:* whether you can place tools on the parallelism lens rather than
seeing them as interchangeable "big data" tools.

## 11. "You notice your app has grown its own heartbeats and routing table. Thoughts?"

**Model answer.** That's a sign **distributed-systems concerns have leaked into application
code** — the app is now doing failure detection, membership, and routing, i.e., it's becoming a
distributed system. Sometimes that's legitimate (a **smart client** *should* own routing). But
often it should be delegated to an existing primitive: service discovery, a service mesh, a
coordination service (ZooKeeper/etcd/Keeper), or a load balancer with health checks. The point
is to make it a **deliberate decision** about ownership, not an accidental reimplementation of
solved problems.

*What they're really testing:* maturity — recognizing when you're rebuilding infrastructure and
knowing the off-the-shelf alternatives.

## 12. "Explain Monoid → Functor → Monad → Applicative in terms a data engineer cares about."

**Model answer.** **Monoid** = identity + associative combine → **distributable aggregation**
(`reduce`, `sum`), because associativity lets partial results merge in any order. **Functor** =
`map` → **per-element transform** with no interaction (data parallelism). **Monad** = `flatMap`
→ **sequential/dependent** composition (step 2 depends on step 1's result; also flattens
nested computations). **Applicative** = combine **independent** computations (`mapN`) → **merge
multiple streams** or accumulate **all** validation errors at once. In SQL: `sum` = monoid,
aggregation over an expression = functor, multi-relation joins = applicative, `UPDATE` =
effects.

*What they're really testing:* whether you can connect FP abstractions to real distributed/data
operations instead of reciting definitions.

---

### How to practice

Pick a question, set a 2-minute timer, and answer out loud leading with the **lens or
mechanism** first ("this is a leader–follower topology question…"), then the specifics, then a
product name. If you reach for a product name first, restart. Cross-check any fact against the
relevant concept file and [10 — Reading List](10-reading-list.md).
