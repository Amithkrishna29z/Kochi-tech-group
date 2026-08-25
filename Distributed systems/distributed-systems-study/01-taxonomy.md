# 01 — A Taxonomy of Distributed Computing

> Part of the [Distributed Computing Study Pack](README.md). See also
> [02 — Data-Processing Frameworks](02-data-processing-frameworks.md),
> [04 — Coordination & Topology](04-coordination-and-topology.md).

## The problem this file solves

Most engineers learn distributed systems as a pile of product names — Kafka, Spark,
Hadoop, Flink — and memorize which buttons each one has. That knowledge evaporates the
moment a new tool appears. The goal here is the opposite: build a small set of *lenses*
you can point at any distributed system so that a tool you have never seen becomes
"oh, it's a master-slave, data-parallel, batch system with a smart client" — and now you
already know its failure modes and its API shape.

## First, un-conflate three activities

People say "I work with distributed systems" to mean three quite different jobs. Keep them
separate; the skills and the failure modes differ.

| Activity | What you build | Typical role | Examples |
|---|---|---|---|
| Distributed **application** programming | Business services that happen to be spread across machines | Backend / enterprise dev | Microservices, REST/gRPC APIs, sagas |
| Distributed **data processing** | Pipelines that transform large datasets | Data engineer | Spark jobs, MapReduce, Flink pipelines |
| Distributed **systems** engineering | The infrastructure the other two run on | Infra / platform / DB engineer | Kafka, ZooKeeper, ClickHouse, schedulers |

A historical note that explains today's job market: the Kafka/Hadoop/Spark wave (roughly
2006–2014) put *data processing* and *systems engineering* in the spotlight first. The
microservices wave after ~2015 made *distributed application programming* a mainstream
skill. If you are a Spring Boot developer, you have mostly been doing the third row's
consumer and the first row's producer — this pack fills in the middle and the bottom.

The thesis of the whole pack: **learn to reason about and construct distributed
applications holistically**, not to memorize one tool. The rest of this file gives you the
lenses to do that.

## Lens 1 — Execution model

Four words get used loosely; pin them down:

- **Sequential** — one thing at a time, one worker.
- **Parallel** — many things at the *same* time to go faster, usually on one machine's
  many cores, working toward one result. The problem is *decomposed*.
- **Concurrent** — many things *in progress* (interleaved) that may or may not run at the
  same instant; about *structure and dealing with independent tasks*, not necessarily speed.
- **Distributed** — parallel/concurrent work spread across machines that fail
  independently and communicate only by messages over an unreliable network.

The useful claim: **one programming model — a stream / reactive-stream model such as Rx
(Reactive Extensions) — can express all four**, *provided you actually know what the
underlying system does with your code*. A `map` over a stream can run sequentially, across
threads, or across a cluster; the code reads the same, but the guarantees differ. The
danger is writing code as if it were sequential when the runtime is distributed. Much of
[03 — FP Foundations](03-functional-programming-foundations.md) is about the discipline
that keeps that abstraction honest.

## Lens 2 — Type of parallelism

- **Data parallelism** — same operation applied to many partitions of data in parallel
  (Spark `map`/`filter` over partitions; SIMD in spirit).
- **Task parallelism** — different operations run at the same time (Storm bolts doing
  different work; an actor system).
- **Pipeline parallelism** — stage 1 processes record *n+1* while stage 2 processes record
  *n* (a CPU pipeline, a Unix pipe). Treat pipeline parallelism as an *emergent property*
  of data + task parallel systems rather than a separate category — it is what you get when
  a chain of operators each keep busy on different items.

Knowing which kind a framework offers tells you where it will bottleneck: data-parallel
systems bottleneck on skew and shuffles; task-parallel systems bottleneck on the slowest
task and on coordination.

## Lens 3 — Computation as a graph (the DAG)

Nearly every distributed processing system models a job as a **Directed Acyclic Graph
(DAG)**: nodes are computations, edges are data dependencies. This is the same idea behind
build systems (`make`, Bazel) and job schedulers (Airflow).

Why it matters: a **topological sort** of the DAG yields an execution order, and — more
importantly — it reveals **independent paths** that can run in parallel. Spark's Catalyst/
DAG scheduler, Flink's job graph, and MapReduce's map→shuffle→reduce are all DAGs. Once you
see "it's a DAG," you know the engine can (a) reorder/fuse operations, (b) parallelize
independent branches, and (c) recompute a lost node from its inputs (lineage-based recovery).

## Lens 4 — Coordination topology

Who is in charge, and what happens when that node dies? Three archetypes (full treatment in
[04 — Coordination & Topology](04-coordination-and-topology.md)):

- **Master–slave** — one master plans and assigns; workers execute. Example: Spark
  (driver + executors). Simple, but the master is a special node.
- **Leader–follower** — peers of equal software, but one is *elected* leader per unit
  (per partition in Kafka). Needs a **witness / consensus service** to elect and to break
  ties — historically ZooKeeper for Kafka, now KRaft; ClickHouse can use ClickHouse Keeper.
  Election typically rides on **heartbeats**.
- **Multi-master** — every node accepts writes; peers reconcile. Example: ClickHouse
  replication; often uses gRPC for inter-node coordination. Highest availability, hardest
  consistency story.

## Lens 5 — Processing strategy (space, time, space-time)

A compact way to classify what a system does with data:

- **Data-in-space** — the dataset is static and sits still; you sweep compute over it.
  This is **batch** (Hadoop MapReduce, Spark batch).
- **Data-in-time** — data arrives as an unbounded flow; you process it as it comes. This is
  **streaming**.
- **Data-in-space-time** — event streams where *ordering and time* are first-class and
  producers can outrun consumers, so you need **reactive** semantics and **backpressure**.

Layer three more axes on top and you get a combinatorial space of system "types":

- **static vs dynamic** (fixed graph/endpoints vs graph that changes at runtime),
- **deterministic vs stochastic** (same input → same output, or not),
- **linear vs non-linear** (a straight pipeline vs feedback/branching/joins).

You do not memorize the grid; you use it to *place* a new tool. "Flink is dynamic,
data-in-time, mostly deterministic, non-linear (has joins/windows)" tells you a lot before
you read a page of its docs.

## Lens 6 — Client model: smart client vs dumb client

Where does the routing/retry/coordination logic live — in the client library or on the
server?

- **Smart client** — the client library knows the cluster topology and does the routing,
  partition selection, ret/failover, and sometimes read-from-replica / write-to-primary
  logic. Kafka's consumer/producer, Cassandra drivers, and Mongo drivers are smart clients.
  Distributed systems *prefer* this because it removes a server-side hop and lets the client
  make locality decisions.
- **Dumb client** — the client just opens a connection and sends requests to a single
  endpoint (a load balancer or a proxy) and lets the server figure everything out. This is
  the default mental model for web/enterprise developers, and it tends to produce the
  **slowest** distributed infrastructure, because every routing decision costs an extra
  network hop and a central chokepoint.

The trade-off is real, not free: a smart client is more complex, must be updated when the
protocol changes, and pushes reliability concerns (CRC checks, auth codes, retries) up into
the client/higher layers. But it buys flexibility such as *write-to-master / read-from-slave*
without a coordinator in the hot path.

**Interview signal.** A strong answer never starts with a product name. It starts with the
lenses: "What's the execution model, what kind of parallelism, is it a DAG, what's the
coordination topology, batch or streaming, smart or dumb client?" — and *then* names the
tool that fits. Being able to say "pipeline parallelism is really emergent from data+task
parallelism" or "a topological sort of the DAG is what exposes parallel paths" marks
someone who understands mechanism, not marketing.
