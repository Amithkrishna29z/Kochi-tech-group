# 08 — Flashcards

> Part of the [Distributed Computing Study Pack](README.md). Atomic Q/A pairs for spaced
> repetition — one idea per card. Cover the answer, recall, check. Grouped by source file.

## Taxonomy ([01](01-taxonomy.md))

Q: Name the three activities people conflate under "distributed systems."
A: Distributed application programming, distributed data processing, distributed systems engineering.

Q: Difference between concurrent and parallel?
A: Parallel = same-instant execution for speed; concurrent = multiple tasks in progress/interleaved (structure), not necessarily simultaneous.

Q: What single programming model is claimed to unify sequential/parallel/concurrent/distributed?
A: A stream / reactive-stream model (e.g., Rx) — provided you know what the runtime actually does.

Q: Three types of parallelism?
A: Data parallelism, task parallelism, pipeline parallelism (pipeline is emergent from data+task).

Q: What data structure models almost every distributed computation?
A: A Directed Acyclic Graph (DAG).

Q: What does a topological sort of the job DAG reveal?
A: An execution order and the independent paths that can run in parallel.

Q: The three coordination topologies?
A: Master–slave, leader–follower, multi-master.

Q: Three processing strategies by data's relationship to time?
A: Data-in-space (batch), data-in-time (streaming), data-in-space-time (event streams needing reactive/backpressure).

Q: Smart client vs dumb client — which do distributed systems prefer, and why?
A: Smart client; it routes/retries at the endpoint, avoiding an extra hop and central chokepoint. Dumb clients produce the slowest infrastructure.

## Frameworks ([02](02-data-processing-frameworks.md))

Q: What FP operations give MapReduce its name?
A: map (per-element mapping domain→range) and reduce (associative fold/aggregation).

Q: What did Hadoop 2 add and why did it matter?
A: YARN — separated resource management from the processing model, letting DAG engines like Spark run on the cluster.

Q: In Spark, transformations vs actions?
A: Transformations are lazy and record the DAG; actions trigger execution.

Q: Why is Spark's laziness valuable?
A: Seeing the whole DAG before running lets Catalyst optimize (fuse steps, push down filters, pick joins).

Q: How does Spark recover a lost partition?
A: Recompute it from lineage (the recorded DAG), not by replication.

Q: RDD vs DataFrame vs Dataset?
A: RDD = low-level typed functional; DataFrame = schema'd untyped rows (optimizable); Dataset = typed schema'd (JVM), optimizable + compile-time types.

Q: Six things forbidden inside a Spark transformation?
A: IO, service/network calls, null, exceptions, threads, nondeterminism.

Q: What happens if you break purity inside a Spark transformation?
A: You lose lineage/retry/speculation guarantees — Spark "degrades" to a mere scheduler (Storm-like) running side effects unpredictably.

Q: Is Spark streaming event-at-a-time?
A: No — it's micro-batch (Structured Streaming slices the stream into small batches).

Q: What cost does PySpark add, and how is it mitigated?
A: A JVM↔Python serialization boundary for Python UDFs; mitigate with built-in/SQL functions and Arrow/pandas (vectorized) UDFs.

Q: In which framework is calling a service inside an operator normal?
A: Storm (bolts) — unlike idiomatic Spark.

Q: What model does Akka implement, and for what kind of parallelism?
A: The actor model (isolated actors, async messages); task/computation parallelism, not data parallelism.

## FP foundations ([03](03-functional-programming-foundations.md))

Q: Define referential transparency.
A: An expression can be replaced by its value without changing the program — same input → same output, no side effects.

Q: Why does referential transparency matter for distribution?
A: It makes retry, speculative execution, reordering/parallelizing, and lineage recomputation safe.

Q: FP replacement for null?
A: Option/Maybe (or Null Object pattern).

Q: FP replacement for exceptions as control flow?
A: Either / Try — errors as data.

Q: What is a Monoid and why does it enable distributed aggregation?
A: Identity + associative combine; associativity lets partial results combine in any grouping to the same total.

Q: What does a Functor give you?
A: `map` — structure-preserving per-element transform (data parallelism).

Q: Monad vs Applicative in one line?
A: Monad `flatMap` = sequential/dependent composition; Applicative `ap`/`mapN` = combine independent computations (merge streams, accumulate all errors).

Q: SQL analogy for Monoid, Functor, Applicative, Effects?
A: `sum/max` = Monoid; aggregation over an expression = Functor; multi-relation joins = Applicative; UPDATE/INSERT = Effects.

Q: Why can Rx compose event streams where plain streams can't?
A: Rx is higher on the ladder (applicative/effect-aware): merge/combineLatest/zip/retry and first-class error/completion signals.

## Coordination & topology ([04](04-coordination-and-topology.md))

Q: How do distributed systems detect a failed node?
A: Heartbeats / periodic polling with a timeout — pull-based, because a dead node can't push.

Q: Why does leader–follower need a witness/consensus service?
A: To elect a leader and store metadata without split-brain (two leaders after a partition).

Q: What did Kafka use for coordination historically, and what replaces it?
A: ZooKeeper historically; KRaft (self-managed Raft metadata) removes the ZooKeeper dependency.

Q: ClickHouse's ZooKeeper alternative?
A: ClickHouse Keeper (ZooKeeper-compatible).

Q: Why does leader–follower give near-automatic zero-downtime?
A: Failover promotes an already-in-sync follower — no state reload.

Q: Multi-master's main strength and main cost?
A: Strength: write-anywhere availability/locality. Cost: hardest consistency (conflict reconciliation).

Q: Heartbeat timeout trade-off?
A: Short = fast failover but false positives; long = stable but slow recovery.

## Databases & wire protocols ([05](05-databases-and-wire-protocols.md))

Q: What is a Call-Level Interface, with examples?
A: A function-call DB API — JDBC, ODBC, ADO.NET, Python DB-API. Core job: language↔driver↔DB type mapping.

Q: What does the driver manager do?
A: Selects the concrete driver based on the connection string/URL.

Q: What is a wire-level protocol? Give two examples.
A: How requests/results are encoded/framed/compressed on the connection — SQL\*Net (Oracle), TDS (SQL Server).

Q: Does Kafka use gRPC?
A: No — Kafka uses its own proprietary binary wire protocol.

Q: Which layer is portable across databases, and which actually differs?
A: The CLI is portable; the wire protocol is what differs.

Q: Why is a columnar (OLAP) DB fast at aggregations?
A: It stores each column contiguously, scanning only the columns a query needs.

Q: What is "shared-nothing"?
A: Each node owns its own storage and CPU; no shared disk.

Q: What distributed-compute idea is "latent" inside a sharded ClickHouse?
A: Each shard computes a partial aggregate (a monoid) and the initiator merges them — map/reduce at the storage layer.

Q: Storage-compute coupled vs decoupled — one example each?
A: Coupled: ClickHouse/MPP DBs. Decoupled: Spark/Databricks over object storage, Snowflake, Fabric/Synapse.

Q: Two kinds of executable specification in a pipeline?
A: Config-for-configuration (source/sink endpoints) and config-for-action (the transformation graph); each static or dynamic.

## Case studies ([06](06-case-studies.md))

Q: Why is a topic-per-entity (unbounded topic names) design risky in Kafka?
A: Unbounded metadata/file-handle/partition growth → instability. Put the entity id in the message key instead.

Q: Which topology matches a "zero-downtime stateful streaming" requirement, and why?
A: Leader–follower — recovery is promotion of a live in-sync replica, not a snapshot restore.

Q: Core principle about failure notifications?
A: Detect failure by polling (heartbeats); a dead component can't announce its own death.

Q: What does an app growing heartbeats/membership/routing tables signify?
A: Distributed-systems concerns have leaked into app code — decide whether a coordination primitive should own them.
