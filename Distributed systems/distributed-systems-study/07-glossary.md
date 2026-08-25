# 07 — Glossary

> Part of the [Distributed Computing Study Pack](README.md). Terms are grouped by theme;
> within a group they build on each other. Cross-references point to the file with the full
> treatment.

## Activities & framing

**Distributed application programming** — building business services spread across machines
(microservices). What most backend/enterprise devs do. See [01](01-taxonomy.md).

**Distributed data processing** — pipelines that transform large datasets across a cluster
(Spark, MapReduce). See [02](02-data-processing-frameworks.md).

**Distributed systems engineering** — building the infrastructure itself: messaging systems,
processing engines, coordination services. See [01](01-taxonomy.md).

## Execution & parallelism

**Sequential / Parallel / Concurrent / Distributed** — one-at-a-time / same-instant-for-speed
/ interleaved-independent-tasks / spread-across-independently-failing-machines. See
[01](01-taxonomy.md).

**Data parallelism** — same operation applied to many data partitions at once.

**Task parallelism** — different operations running at once.

**Pipeline parallelism** — stage *n+1* on the next item while stage *n* runs; emergent from
data+task parallelism.

**Reactive stream** — a stream abstraction (e.g., Rx) that unifies the four execution models
under one programming model, with backpressure and first-class error/completion signals.

**Backpressure** — a consumer signaling a fast producer to slow down, so buffers don't
overflow. Central to data-in-space-time processing.

## Graphs & scheduling

**DAG (Directed Acyclic Graph)** — nodes = computations, edges = data dependencies; the shape
almost every processing engine uses for a job. See [01](01-taxonomy.md).

**Topological sort** — an ordering of a DAG respecting dependencies; reveals independent paths
that can run in parallel.

**Lineage** — Spark's recorded DAG of transformations, used to recompute a lost partition from
its inputs instead of replicating. See [02](02-data-processing-frameworks.md).

## Coordination & topology

**Master–slave** — one master plans/assigns, workers execute (Spark driver/executors). See
[04](04-coordination-and-topology.md).

**Leader–follower** — equal software, one elected leader per unit (Kafka partition); needs a
witness. See [04](04-coordination-and-topology.md).

**Multi-master** — every node accepts writes and peers reconcile (ClickHouse replication).

**Witness / consensus service** — external source of truth for leadership/metadata that
prevents split-brain: ZooKeeper, KRaft, ClickHouse Keeper.

**ZooKeeper** — coordination service historically used by Kafka (and others) for election,
membership, and metadata.

**KRaft** — Kafka's self-managed metadata mode (KIP-500) using Raft consensus, removing the
ZooKeeper dependency.

**ClickHouse Keeper** — a ZooKeeper-compatible coordination service used by ClickHouse.

**Heartbeat** — periodic "I'm alive" signal; missing N of them within a timeout declares a
node dead. Failure detection is poll/pull-based. See [04](04-coordination-and-topology.md),
[06](06-case-studies.md).

**Split-brain** — two nodes both believing they are leader after a partition; the failure
mode a witness/consensus prevents.

**Leader election** — choosing a new leader (via the witness) after heartbeats to the current
leader time out; guarded by an epoch/term to fence the old leader.

## Processing strategy

**Data-in-space** — static dataset swept by compute; batch (Hadoop, Spark batch).

**Data-in-time** — unbounded flow processed as it arrives; streaming.

**Data-in-space-time** — event streams where time/ordering are first-class; needs reactive
semantics + backpressure.

**Static vs dynamic / deterministic vs stochastic / linear vs non-linear** — the three axes
that, combined with the above, span the space of system types. See [01](01-taxonomy.md).

## Client model

**Smart client** — client library that knows cluster topology and does routing/retry/failover
(Kafka clients, Cassandra drivers). Preferred by distributed systems.

**Dumb client** — client that hits one endpoint/proxy and lets the server route; simplest for
web devs, tends to produce the slowest infrastructure. See [01](01-taxonomy.md).

## Frameworks

**MapReduce** — Google's batch model; `map` (per-element mapping) + `reduce` (associative
fold), with a shuffle between. See [02](02-data-processing-frameworks.md).

**Hadoop** — open-source MapReduce; HDFS (storage) + engine. Hadoop 2 added **YARN**,
separating resource management so DAG engines like Spark could run on it.

**HDFS** — Hadoop Distributed File System.

**YARN** — Hadoop 2's resource negotiator; decoupled resource management from the processing
model.

**Spark** — declarative, lazy, DAG-based, master-slave data-processing engine. See
[02](02-data-processing-frameworks.md).

**Transformation (Spark)** — lazy operation (`map`, `filter`, `join`) that records a step.

**Action (Spark)** — eager operation (`count`, `collect`, `write`) that triggers execution.

**RDD / DataFrame / Dataset** — low-level typed functional API / schema'd untyped table /
typed schema'd table (JVM). Higher levels get Catalyst optimization.

**Catalyst** — Spark's query optimizer for DataFrame/Dataset.

**Micro-batch** — Spark Structured Streaming's model: slice the stream into small batches and
run the batch engine on each.

**PySpark** — Python API for Spark; Python UDFs cross a **serialization boundary** to/from the
JVM (a cost), mitigated by Arrow/pandas UDFs.

**Storm** — task/process-parallel stream processor (spouts + bolts); side effects inside
operators are normal.

**Akka** — actor model: isolated actors communicate by async message passing; for
task/computation parallelism, not data parallelism.

**Actor model** — concurrency model of isolated stateful actors exchanging messages, no shared
memory.

**Samza** — asynchronous stream-processing framework, Kafka-integrated, Spark-like.

## Functional-programming machinery

**Referential transparency** — same input → same output, no side effects; the property that
makes retry/speculation/lineage safe. See [03](03-functional-programming-foundations.md).

**The six villains** — IO, service/network calls, null, exceptions, threads, nondeterminism:
forbidden inside pure transformations.

**Option / Maybe** — type representing presence/absence; replaces `null`.

**Either / Try** — types representing success-or-error as *data*; replace thrown exceptions.

**Monoid** — identity + associative combine; makes distributed aggregation legal (`reduce`).

**Functor** — supports `map`; structure-preserving per-element transform (data parallelism).

**Monad** — Functor + `flatMap`; sequential/dependent composition (and Cartesian product).

**Applicative** — combines *independent* computations (`ap`/`mapN`); merges multiple streams,
accumulates all errors.

**Effects** — contained mutation/IO at the edge of a pure program (Spark actions, `IO` types).

**Fold** — reducing a collection to a value via a combine operation; the essence of `reduce`.

## Databases, drivers, protocols

**CLI (Call-Level Interface)** — function-call DB API: JDBC, ODBC, ADO.NET, Python DB-API. Its
core job is language↔driver↔database type mapping. See [05](05-databases-and-wire-protocols.md).

**Driver Manager** — selects the concrete driver from the connection string/URL.

**Wire-level protocol** — how requests/results are encoded, framed, compressed on the
connection. Examples: SQL\*Net (Oracle), TDS (SQL Server), ClickHouse native + HTTP, Kafka's
proprietary format.

**gRPC / Thrift / Avro** — general RPC/serialization frameworks used to build wire protocols
(protobuf-over-HTTP/2, etc.); contrast with Kafka's bespoke format.

**Columnar / OLAP** — storage that keeps each column contiguous, optimized for analytical
aggregation scans (ClickHouse).

**Shard** — a horizontal slice of a dataset on its own node/storage.

**Shared-nothing** — architecture where each node owns its own storage and CPU; no shared
disk. See [05](05-databases-and-wire-protocols.md).

**Distributed table (ClickHouse)** — a table that fans a query out across shards and merges
partial results.

**Storage-compute coupling vs decoupling** — compute co-located with data (ClickHouse, MPP DBs)
vs storage and compute scaled independently (Spark/Databricks over object storage, Snowflake,
Fabric/Synapse).

**Vertical vs horizontal scaling** — bigger nodes (more CPU/RAM per node) vs more nodes.

**Executable specification** — pipeline definition as config: *config-for-configuration*
(source/sink endpoints) and *config-for-action* (the transformation graph); each static or
dynamic. See [05](05-databases-and-wire-protocols.md).

## Reliability primitives

**CRC / checksum** — detects data corruption on the wire.

**Auth code / signature** — authenticates the sender of a message; layered above the
transport, often in the smart client.

**Idempotency** — an operation that can be applied repeatedly with the same effect; the
practical requirement when you must do side effects in an at-least-once system.
