# 06 — Case Studies (Anecdotes → Engineering Lessons)

> Part of the [Distributed Computing Study Pack](README.md). These are *illustrative*
> stories from one team's experience, reframed as general lessons. The durable concepts they
> point to live in [04 — Coordination & Topology](04-coordination-and-topology.md) and
> [01 — Taxonomy](01-taxonomy.md). Treat the specifics as `[unverified — check]` anecdote;
> treat the lessons as transferable.

Each case follows the same shape: **Situation → What actually happened → Lesson.**

## Case 1 — "Can we create topics dynamically?" (Kafka vs Pulsar)

**Situation.** A lunch-table debate: a design wanted to create a **new Kafka topic per
entity** (per user, per device, per session) on the fly, and someone argued Apache Pulsar
handles dynamic topic creation better.

**What actually happened.** The real question is not "which broker allows dynamic topics" but
"what is a topic *for*." A topic is a **named, partitioned, durable log** with associated
metadata, partitions, replicas, and retention. Every topic costs cluster metadata, open file
handles, and per-partition overhead. Creating topics keyed on unbounded identifiers
(`orders-{userId}`) means **unbounded growth** of that overhead — a metadata and file-handle
explosion, slow rebalances, and eventually an unstable cluster. Kafka *can* auto-create
topics; that does not make it wise. Pulsar's architecture tolerates *many* topics better, but
"tolerates more" is not "unbounded is fine."

**Lesson.** Model streams by **bounded topic + key**, not by unbounded topic name. Put the
entity id in the **message key** (which drives partitioning and per-key ordering), not in the
topic name. Reserve distinct topics for distinct *schemas/semantics*, not for distinct
*instances*. When you evaluate a messaging system, ask about the **cardinality limits of the
unit you plan to multiply** (topics, partitions, subscriptions) before you design around it.
See publish/subscribe semantics in [07 — Glossary](07-glossary.md).

## Case 2 — Zero-downtime architecture: leader-follower vs snapshot-of-state

**Situation.** A requirement for a stateful streaming system: survive a node failure with
**zero downtime** and no data loss. Candidate engines: Kafka, Spark Streaming, Flink, AWS
Kinesis.

**What actually happened.** The team realized the answer follows directly from *topology*, not
from feature checklists. Kafka is **leader–follower per partition** (see
[04](04-coordination-and-topology.md)): followers continuously replicate the leader's log, so
a leader failure is handled by **promoting an already-in-sync follower** — recovery is
effectively immediate and lossless because the replica is *already there and current*. The
snapshot-based engines (Spark Structured Streaming, Flink, Kinesis-style consumers) achieve
fault tolerance by periodically **checkpointing state** and, on failure, **restoring the last
snapshot and replaying** from it. That works, but restore-and-replay is a heavier, slower
operation, and you can lose the window since the last checkpoint unless carefully tuned.

**Lesson.** For continuous-availability *stateful* streams, prefer a topology whose recovery
is **promotion of a live replica** over one whose recovery is **restore a snapshot**. More
generally: **derive the engine from the topology the requirement implies**, not from a feature
matrix. "Zero-downtime stateful streaming" is practically a description of leader–follower
replication. (This is the same reasoning as the topology-by-requirement table in
[04](04-coordination-and-topology.md).)

> Nuance to keep it honest: Flink and Spark can reach very high availability with hot
> standbys and frequent checkpoints, and Kafka failover still has a short leader-election gap.
> The lesson is about *matching recovery mechanism to requirement*, not that one product is
> universally superior.

## Case 3 — How Kafka notices a dead node: heartbeats, not push

**Situation.** The team assumed a failed broker would somehow *notify* the cluster ("push to a
listener") when it went down.

**What actually happened.** It cannot — a node that has failed or been partitioned is, by
definition, unable to push anything. Kafka (like essentially all distributed systems) detects
failure by **heartbeats / periodic polling**: brokers and the controller exchange periodic
liveness signals, and a node that misses its heartbeats within the session timeout is declared
dead, triggering leader election through the coordination service (ZooKeeper/KRaft — see
[04](04-coordination-and-topology.md)).

**Lesson.** **Failure detection is pull/poll-based; data delivery is push-based.** Do not
design a health system that relies on the dying component to announce its own death. Set
heartbeat/timeout values as an explicit trade-off: short timeouts fail over fast but produce
false positives on slow nodes; long timeouts are stable but recover slowly. Whenever someone
says "the system will tell us when X dies," ask "*how*, if X is the thing that's dead?"

## Case 4 — The self-healing distributed reverse-proxy

**Situation.** An application-level proxy needed to route requests to backend nodes that could
come and go. Initially it was treated as ordinary application code.

**What actually happened.** To route correctly, the proxy had to know which backends were
alive — so it grew an embedded **keep-alive / heartbeat** to each backend and updated its
routing table as backends appeared and disappeared, effectively becoming a **self-healing
reverse-proxy**. In doing so, application code re-implemented the *same* failure-detection and
membership machinery that a coordination service provides — the distributed-systems nature had
"leaked" up into the app layer.

**Lesson.** **Distributed-systems concerns don't stay in the infrastructure tier; they leak
into application code the moment your app spans machines.** When you see app code growing
heartbeats, membership tracking, retry/failover, and routing tables, recognize that you are
building a distributed system — and consider whether an existing coordination primitive
(service discovery, a service mesh, a coordination service like ZooKeeper/etcd/Keeper, or a
load balancer with health checks) should own that responsibility instead of bespoke code. Not
every leak is bad (sometimes a smart client *should* own routing — see the client-model
discussion in [01](01-taxonomy.md)); the point is to make it a *decision*, not an accident.

## Cross-cutting lessons

- **Requirements select topology; topology selects the tool.** (Cases 2, 4)
- **Detect failure by polling; deliver data by pushing.** (Cases 3, 4)
- **Beware unbounded cardinality in whatever unit you multiply.** (Case 1)
- **When infra concepts appear in app code, name them and decide who should own them.** (Case 4)

**Interview signal.** Strong candidates turn a war story into a principle. Instead of "we used
Kafka and it was reliable," they say "the requirement was zero-downtime stateful streaming,
which *is* leader–follower replication, so failover became replica promotion rather than
snapshot restore." They reflexively question any design that depends on a dead component
announcing its own death, and they can spot when application code has quietly become
distributed-systems code.
