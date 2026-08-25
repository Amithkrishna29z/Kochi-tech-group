# 02 — Distributed Data-Processing Frameworks

> Part of the [Distributed Computing Study Pack](README.md). Read
> [01 — Taxonomy](01-taxonomy.md) first; the correctness rules here rest on
> [03 — FP Foundations](03-functional-programming-foundations.md).

## The problem this file solves

You have more data than one machine can process in acceptable time. You want to spread the
work across a cluster *without* hand-writing socket code, retries, and failure recovery. A
data-processing framework is the reusable machinery for that: you describe *what* to compute,
it decides *how and where* to run it, and it recovers when a machine dies. This file walks
the lineage — MapReduce → Hadoop → Spark — and then the task-parallel cousins (Storm, Akka,
Samza), using the lenses from [01](01-taxonomy.md).

## MapReduce — the functional-programming origin

MapReduce began at Google (Dean & Ghemawat, 2004) as a batch framework, but its *name and
shape come from functional programming*:

- **map** — in FP, a function is a mapping from a domain set to a range set. `map(f)` applies
  `f` independently to every element. Independence is the whole point: independent elements
  can go to different machines. (`domain → range`, element by element.)
- **reduce** — a **fold**/aggregation: combine many values into fewer using an associative
  operation (sum, max, concat). Because the operation is associative (and ideally has an
  identity — a **monoid**, see [03](03-functional-programming-foundations.md)), partial
  reductions can happen on each machine and then be combined.

The programming model is intentionally tiny: `map` produces `(key, value)` pairs, the
framework **shuffles** so all values for a key land together, and `reduce` aggregates per
key. Between them sits map → **shuffle/sort** → reduce — a three-stage DAG. That constraint
is what let Google run it fault-tolerantly on cheap hardware: any failed map or reduce task
is just re-run, because it is a pure function of its input.

## Hadoop — the open-source port, and its scaling pain

**Hadoop** is the open-source implementation of the MapReduce paper (its early development is
associated with Yahoo and the Apache community). Hadoop bundles two things worth separating:

- **HDFS** — a distributed file system (data-in-space storage).
- **MapReduce engine** — the compute.

**Hadoop 1** exposed a real limitation: everything had to be expressed as map+reduce, the job
tracker was a scaling/HA bottleneck, and iterative or multi-stage jobs paid to write
intermediate results to disk between every stage. **Hadoop 2** re-architected around **YARN**
(Yet Another Resource Negotiator), separating resource management from the processing model.
That separation is what let *other* engines — including Spark — run on the same cluster and
data, and it is why Spark-like DAG execution became the norm rather than rigid map→reduce.

> The source framed this as "Hadoop 2 was re-engineered with Spark-like features." More
> precisely: Hadoop 2/YARN generalized the resource layer so DAG engines like Spark could run
> on it; Spark itself is a separate project, not a Hadoop feature.

## Spark — declarative, lazy, DAG-based data processing

**Apache Spark** is the workhorse. Place it with the lenses: **master–slave** (a *driver*
plans, *executors* run tasks), **data-parallel**, **DAG-based**, can run **batch or
streaming**, and it is **declarative** — you describe transformations, Spark builds and
optimizes the execution plan.

### Transformations vs actions, and laziness

Spark splits operations into two kinds:

- **Transformations** (`map`, `filter`, `select`, `join`, `groupBy`) — return a new dataset
  and are **lazy**: they only *record* what to do, building up the DAG.
- **Actions** (`count`, `collect`, `write`, `show`) — force execution. Only when you call an
  action does Spark schedule the recorded DAG.

Laziness is what enables optimization: because Spark sees the whole DAG before running it, it
can fuse steps, push filters down, and pick join strategies (this optimizer is **Catalyst**
for DataFrames/Datasets). Lineage — the recorded DAG — is also how Spark recovers: a lost
partition is *recomputed* from its inputs rather than replicated.

### RDD, DataFrame, Dataset

- **RDD** — Resilient Distributed Dataset: the low-level, typed, functional API (partitions +
  transformations). Maximum control, no schema-aware optimization.
- **DataFrame** — a distributed table with a schema; untyped rows (`Row`). Catalyst can
  optimize it heavily. (In Python this is the main API.)
- **Dataset** — typed DataFrame (JVM only, Scala/Java): schema optimization *plus*
  compile-time types.

Rule of thumb: reach for DataFrame/Dataset so the optimizer can help; drop to RDD only when
you need control it cannot express.

### The mental model: Spark as a distributed, push-based LINQ / Rx

A useful analogy (treat as pedagogy, not an official claim): Spark's declarative
transform-and-action style is like Microsoft's **LINQ** made distributed, and its streaming
side is like **Rx** made distributed.

- **LINQ** is *pull-based*: `IEnumerable`/`IEnumerator` — the consumer pulls the next item.
- **Rx** is the *push-based dual*: `IObservable`/`IObserver` + a scheduler — the source pushes
  items to the observer. Same operators (`map`, `filter`, `flatMap`), opposite direction of
  control.

Spark batch feels like distributed LINQ; Spark Structured Streaming (push of micro-batches)
feels like distributed Rx. The value of the analogy is that the *composition laws* are the
same as in [03 — FP Foundations](03-functional-programming-foundations.md).
`[unverified — check]` The "distributed reverse-LINQ / distributed Rx" phrasing is a teaching
analogy from the source, not Spark's own documented positioning.

### Batch vs streaming

Spark runs batch jobs (data-in-space) and streaming jobs. Its streaming is fundamentally
**micro-batch**: Structured Streaming chops the incoming stream into small batches and runs
the same engine on each. (Project *Continuous Processing* is an experimental low-latency mode,
but micro-batch is the default and the mental model.) Contrast this with Kafka's true
event-at-a-time leader-follower model in [04](04-coordination-and-topology.md) and
[06 — Case Studies](06-case-studies.md).

### The correctness rule: referential transparency inside transformations

This is the single most important operational fact about Spark. Code inside a transformation
(the lambda you pass to `map`/`filter`/UDFs) **must be referentially transparent** — a pure
function of its inputs. Concretely, inside a transformation **do not**:

- do **IO** or call external **services/network** (DB lookups, REST calls),
- return or dereference **null**,
- throw **exceptions** as control flow,
- spawn **threads**,
- introduce **nondeterminism** (random, wall-clock, order-dependence).

Why: Spark's guarantees — recompute-on-failure via lineage, speculative execution, retries,
partition re-ordering — assume a task can be run again and yield the same result. Break
purity and you break those guarantees. In the source's words, the system **degrades from a
data-parallel engine into a mere scheduler** (a Storm-like thing) that happens to run your
side effects an unpredictable number of times. See the "six villains" and their FP remedies
in [03](03-functional-programming-foundations.md).

> Practical corollary: if you genuinely need per-record service calls, batch them at the
> partition boundary (`mapPartitions`), make them **idempotent**, and accept that you are
> stepping outside Spark's purity contract on purpose.

### PySpark and the serialization boundary

**PySpark** lets you write Spark in Python, but the engine runs on the JVM. Python UDFs
therefore cross a **serialization boundary**: rows are serialized from the JVM to a Python
worker process, transformed, and serialized back. That round-trip is a real cost. Mitigations
are documented Spark behavior: prefer **built-in/SQL functions** (they stay on the JVM),
and use **pandas UDFs / vectorized UDFs (Apache Arrow)** to move data in columnar batches
instead of row-by-row when you must run Python.

## The task-parallel cousins

Spark is data-parallel and purity-constrained. Sometimes you want the opposite trade-off.

### Storm — task/process parallelism, side effects allowed

Apache **Storm** models a computation as a **topology** of *spouts* (sources) and *bolts*
(processors). It is **task/process-parallel** and optimizes resource utilization across the
cluster. Crucially, and unlike idiomatic Spark, **you may call services and do IO inside a
bolt** — that is a normal pattern, not a violation. The cost is that you, not the framework,
own exactly-once/idempotency concerns (Storm offers acking and, via Trident, higher-level
guarantees). Reach for Storm when the work *is* per-event side-effecting logic rather than a
pure dataset transform.

### Akka — the actor model

**Akka** implements the **actor model**: independent actors hold private state and
communicate only by **asynchronous message passing** (no shared memory, no locks). It is
built for **computation/task parallelism** and event-stream processing, *not* data
parallelism — there is no notion of partitioned datasets with lineage recovery. Think of it
as the toolkit for building concurrent, resilient, message-driven *applications and systems*
(including, in fact, the internals of other distributed systems), rather than for crunching
a large static dataset.

### Samza — asynchronous stream processing

Apache **Samza** is an asynchronous stream-processing framework (originating at LinkedIn,
tightly integrated with Kafka). It is Spark-like in offering a higher-level processing model
over streams, with stateful processing and fault tolerance, but oriented around continuous
stream jobs. Group it mentally with the stream processors (alongside Flink) rather than the
batch engines.

## Quick comparison

| Framework | Parallelism | Batch/Stream | Side effects in operators? | Recovery model |
|---|---|---|---|---|
| MapReduce/Hadoop | Data | Batch | Discouraged (re-run tasks) | Re-run pure tasks |
| Spark | Data | Both (stream = micro-batch) | **No** (must be pure) | Lineage recompute |
| Storm | Task/process | Stream | **Yes** (normal) | Acking / Trident |
| Akka | Task (actors) | Stream/events | Yes (actors own state) | Supervision/restart |
| Samza | Stream | Stream | Yes | Checkpoint + Kafka log |

**Interview signal.** The distinguishing answer explains *why* Spark forbids IO in
transformations (lineage/recompute/speculation demand purity) and can name the escape
hatches (`mapPartitions`, idempotency, Storm/Akka when you genuinely need per-event side
effects). Mentioning that Spark streaming is micro-batch, that laziness enables the Catalyst
optimizer, and that PySpark pays a serialization cost mitigated by Arrow/pandas UDFs shows
production familiarity, not just tutorial knowledge.
