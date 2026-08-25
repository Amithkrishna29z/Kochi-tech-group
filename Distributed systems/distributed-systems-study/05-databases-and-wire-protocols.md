# 05 — Databases, Drivers & Wire Protocols

> Part of the [Distributed Computing Study Pack](README.md). The ClickHouse case study here
> is the concrete payoff of the aggregation-as-monoid idea in
> [03 — FP Foundations](03-functional-programming-foundations.md) and the multi-master
> topology in [04 — Coordination & Topology](04-coordination-and-topology.md).

## The problem this file solves

When your app "talks to a database," several distinct layers are doing work, and engineers
routinely blur them. Knowing the layers lets you reason about performance (where is the cost?),
portability (what actually changes when I switch databases?), and — the punchline of this file
— when a database can quietly replace a whole data-processing stack. We go from the function
call in your code down to the bytes on the wire, then use ClickHouse as a worked example.

## The layers, top to bottom

### Call-Level Interface (CLI) — the function-call API

A **Call-Level Interface** is the *programming API* your code calls: open a connection,
prepare a statement, execute, iterate results. These are CLIs (the term predates the
command-line meaning; here it means "call-level interface"):

- **JDBC** (Java), **ODBC** (C/cross-language, Microsoft-originated), **ADO.NET** (.NET),
  **Python DB-API** (e.g. `psycopg` for Postgres).

A CLI's central job is a **type mapping**: your language's data types ↔ the driver's data
types ↔ the database's data types. `java.sql.Timestamp` ↔ driver timestamp ↔ Postgres
`timestamptz`. Bugs and performance surprises often live in this mapping (timezones,
`BigDecimal` precision, `NULL` handling — see the null discussion in
[03](03-functional-programming-foundations.md)).

### Driver Manager — picking the driver from the connection string

The **Driver Manager** is the indirection layer that, given a **connection string / URL**,
selects the concrete driver to use. `jdbc:postgresql://host:5432/db` tells the JDBC
DriverManager to load the Postgres driver; `jdbc:clickhouse://host:8123/db` selects the
ClickHouse driver. Your code depends on the *CLI interface*, not the concrete driver — which
is exactly the substitutability that lets a proxy pretend to be a database (below).

### Wire-level protocol — the bytes on the connection

Once a connection is established, the **wire protocol** defines how logical requests/results
are **encoded, framed, compressed, and sent** over that connection. This is where latency and
throughput are actually won or lost. Different databases speak different wire protocols:

| Database / system | Wire protocol |
|---|---|
| Oracle | **SQL\*Net** (a.k.a. Oracle Net) |
| Microsoft SQL Server | **TDS** (Tabular Data Stream) |
| ClickHouse | its **native TCP protocol** *and* an **HTTP protocol** on a separate port |
| Kafka | its own **proprietary binary protocol** (not gRPC/Thrift) |

For contrast, general-purpose RPC/serialization frameworks that other systems build wire
formats on include **gRPC** (protobuf over HTTP/2), **Apache Thrift**, and **Apache Avro**.
Kafka deliberately uses its *own* framing rather than one of these, tuned for its log-append
workload. ClickHouse exposes **two** listening ports for two protocols: a fast native TCP
protocol (default 9000) and an HTTP interface (default 8123) that its JDBC/ODBC drivers and
`curl` can use.

The takeaway: **the CLI stays the same across databases; the wire protocol is what actually
differs.** Portability at the CLI layer hides large differences at the wire layer.

## Executable specifications: config vs action

A pipeline definition is really two kinds of **executable configuration**:

- **Executable config for *configuration*** — the *endpoints*: where data comes from and goes
  to (source and sink connection strings, credentials, formats). These describe *place*.
- **Executable config for *action*** — the *transformation graph*: the operators and how they
  chain, each operator feeding the next. This describes *behavior* (the DAG from
  [01](01-taxonomy.md)).

Both can be **static** (fixed at deploy time) or **dynamic** (endpoints/graph determined at
runtime). Seeing a pipeline as "config endpoints + action graph, each static or dynamic" is a
clean way to design and to read tools like Kafka Connect, Airflow, or Spark job definitions:
operators chain to the next operator, and each end of the graph plugs into a source or sink.

## Case study — ClickHouse as a latent distributed-computation engine

**What ClickHouse is (documented facts):**

- A **columnar** database optimized for **OLAP** (analytical) queries — it stores each column
  contiguously, so aggregations scan only the columns they need.
- **Distributed and replicated**: data is split into **shards** (horizontal partitions) and
  each shard can have replicas. It uses a **shared-nothing** architecture — each node owns its
  own storage and CPU; there is no shared disk.
- **Multi-master replication** for replicas (write to any replica), coordinated via ZooKeeper
  or **ClickHouse Keeper** (see [04](04-coordination-and-topology.md)).
- Sharding is *shards, not merely nodes*: a shard is a slice of the data set, and a
  `Distributed` table transparently fans a query out to all shards and merges the results.

**The insight the practitioner team hit.** When you push a SQL aggregation to a distributed
ClickHouse cluster, each shard computes a **partial aggregate** locally (this is the *monoid*
from [03](03-functional-programming-foundations.md) — associative combine) and the initiator
node merges the partials. That is *exactly* what a distributed data-processing engine does:
map over partitions, reduce across the cluster. In other words, **a full distributed-compute
infrastructure is latent inside the database** — you do not have to bolt Spark on top to get
distributed aggregation; the database already does it, fast, at the storage layer.

**What they built (presented as a case study, not a universal recommendation).** They wrote a
**proxy that presents itself as a database** — it speaks JDBC/ODBC/ADO.NET to BI tools and
applications (thanks to the CLI-substitutability above) but **delegates the actual computation
to ClickHouse**, orchestrating queries across shards. The result was a data-product /
reporting infrastructure with **no Hadoop and no Spark**. `[unverified — check]` This is one
team's architecture and its fit depends heavily on workload; treat it as an illustrative
lesson, not a blanket "replace Spark with ClickHouse" claim.

### The design axis this exposes: storage-compute coupling

- **Coupled storage + compute** (ClickHouse, classic MPP databases): compute runs *next to*
  the data on each shard. Aggregations are extremely fast because there is no data movement to
  a separate compute tier. You scale by adding shards; compute and storage grow together.
  Favors **vertical scaling** for raw compute speed (fat nodes crunch fast) plus **horizontal**
  sharding for volume.
- **Decoupled storage + compute** (Spark/Databricks over object storage; Snowflake;
  Microsoft Fabric/Synapse): storage (S3/ADLS) and compute (elastic clusters) scale
  independently. Favors **horizontal scaling** and elasticity — spin compute up/down without
  moving data — at the cost of moving data to compute at query time.

**Vendor context (documented):** *Databricks* is the company founded by Spark's creators and
bundles Spark over cloud object storage; *Snowflake* and *Microsoft Fabric/Synapse* offer
managed decoupled analytics, and several bundle Spark as an engine. The team's finding was
that for many reporting/aggregation workloads, **a scheduler over a coupled engine like
ClickHouse can replace the decoupled Spark stack** — cheaper and faster — *for those cases*.
It is a "right tool for the shape of the workload" argument, not "ClickHouse beats Spark."

### Why this connects back to the whole pack

The ClickHouse story is the taxonomy from [01](01-taxonomy.md) made concrete: it is a
**multi-master**, **data-parallel**, **data-in-space (then queried)** system whose SQL engine
is a distributed DAG of monoid aggregations, fronted (in the case study) by a **smart proxy**
so that dumb BI clients still get distributed compute. Every lens applies.

**Interview signal.** A strong answer separates CLI (portable function API), driver
manager (selection by connection string), and wire protocol (where cost/differences actually
live), and can name at least SQL\*Net/TDS and that Kafka uses a proprietary wire format. On
ClickHouse, the standout point is recognizing that *distributed aggregation is a monoid the
database already computes across shards*, so a columnar MPP database can subsume much of a
Spark/Hadoop reporting stack — while being honest that this depends on workload shape and on
the storage-compute coupling trade-off.
