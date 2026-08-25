# 04 — Coordination & Topology

> Part of the [Distributed Computing Study Pack](README.md). Builds on the coordination lens
> in [01 — Taxonomy](01-taxonomy.md); the anecdotes it references are worked out in
> [06 — Case Studies](06-case-studies.md).

## The problem this file solves

Once work is spread across machines that fail independently, someone has to answer: who
decides what runs where? who is allowed to accept a write? and — the hard one — what happens
when the node in charge dies? The answer is a *coordination topology*. There are three
archetypes. Pick the wrong one for your availability requirement and you will build either a
fragile system or an over-complicated one.

## Failure detection comes first: heartbeats and polling

Before any topology can react to a dead node, it must *detect* the death. Networks do not send
you a "node X died" event — a dead node and a slow/partitioned node look identical from
outside. So distributed systems detect failure by **heartbeats**: each node periodically
signals "I'm alive," and a peer/coordinator that misses N heartbeats within a timeout declares
the node dead.

Note the direction of information flow. Detection is fundamentally **pull/poll-based**: you
*ask* (or wait for a periodic signal and check the clock), rather than being *pushed* a
notification — because the failure mode you care about is exactly the one where the node can
no longer push anything. Kafka's approach to noticing a dead broker/leader is this kind of
periodic heartbeat/poll ("data polling"), not a push-to-listener callback. This distinction —
poll to detect failure vs push to deliver data — is a recurring theme; see the reverse-proxy
keep-alive case in [06](06-case-studies.md).

The unavoidable tension: **short timeout** = fast failover but false positives (you evict a
node that was merely slow); **long timeout** = fewer false positives but slower recovery.

## Topology 1 — Master–slave

One node (the **master**) holds the plan and assigns work; the others (**slaves/workers**)
execute and report back.

- **Example:** Spark — the **driver** builds the DAG, schedules tasks, and tracks progress;
  **executors** run tasks on partitions. (See [02](02-data-processing-frameworks.md).)
- **Strengths:** simple to reason about; the master has a global view, so scheduling and
  optimization are easy.
- **Weakness:** the master is a *special* node. If it dies mid-job, the job typically dies
  (Spark restarts the application; long-running clusters add master HA). Master-slave is great
  for *bounded jobs* where a restart is acceptable, and for systems where a control plane can
  be made HA separately.

Master–slave is a **role asymmetry baked into the software**: the master runs different code
than the workers.

## Topology 2 — Leader–follower

Here the nodes run the **same software** and are peers, but for each unit of work one peer is
**elected leader** and the others are followers. In Kafka the unit is a **partition**: each
partition has one leader broker that takes reads/writes and followers that replicate it.

### Why you need a witness / consensus service

Election cannot be done by the peers alone without risking **split-brain** (two nodes both
believing they are leader after a partition). So leader–follower needs an external
**witness** — a consensus/coordination service that provides a single source of truth for
"who is leader" and stores cluster metadata:

- Kafka **historically used ZooKeeper** for this (controller election, broker membership,
  metadata).
- Kafka is moving to **KRaft** (KIP-500): a self-managed metadata quorum using the Raft
  consensus protocol, **removing the ZooKeeper dependency**. Modern Kafka can run
  ZooKeeper-less.
- **ClickHouse** uses ZooKeeper *or* its drop-in replacement **ClickHouse Keeper** (a
  ZooKeeper-compatible service) to coordinate replicated tables.

Election itself rides on the heartbeat/timeout machinery above: when followers stop hearing
from the leader, they trigger an election through the witness, which awards leadership to one
candidate (with a fencing term/epoch so the old leader can't keep acting).

### Why this gives near-automatic zero-downtime

Because followers are already replicating the leader's log, a leader failure is handled by
**promoting an existing in-sync follower** — no data reload, minimal interruption. This is the
key insight behind the zero-downtime case study in [06](06-case-studies.md): Kafka's
leader–follower structure gives you failover *by construction*, whereas snapshot-of-state
systems (Spark/Flink/Kinesis-style) recover by restoring a checkpoint, which is a heavier,
slower operation. Choose leader–follower when continuous availability of a *stateful stream*
is the requirement.

## Topology 3 — Multi-master

Every node is a peer that **accepts writes**; nodes reconcile with each other. There is no
single writer.

- **Example:** ClickHouse replication — replicas are equal; you can write to any replica and
  it propagates. Inter-node coordination often uses **gRPC** (protobuf over HTTP/2) for
  efficient peer-to-peer RPC.
- **Strengths:** highest write availability and locality (write to the nearest/any node); no
  leader to fail.
- **Weakness:** the hardest **consistency** story — concurrent writes to the same key on
  different masters must be reconciled (conflict resolution, last-write-wins, CRDTs, or
  quorum reads/writes). You trade coordination cost now (leader election) for reconciliation
  cost later.

## Choosing a topology by requirement

| Requirement | Best fit | Why |
|---|---|---|
| Bounded batch/ETL job, restart OK | Master–slave | Global scheduling, simple; restart on master loss acceptable |
| Continuous, stateful stream, zero-downtime | Leader–follower | Failover = promote in-sync follower; no state reload |
| Write-anywhere, max availability, geo-local writes | Multi-master | No leader to fail; reconcile conflicts later |

A subtle but important point from [01](01-taxonomy.md): topology and **client model**
interact. Leader–follower and multi-master systems tend to ship **smart clients** that know
which node currently owns a partition/shard and route accordingly (e.g., write-to-leader,
read-from-follower). A **dumb client** hitting a central proxy re-introduces a single hop and
a chokepoint, which is why it produces the slowest infrastructure even on top of a good
topology.

## Reliability lives above the topology

None of these topologies, by themselves, guarantee that a message arrived intact or from a
legitimate sender. Those concerns are layered *on top*: **CRC/checksums** to detect
corruption on the wire, and **auth codes / signatures** to authenticate the sender. Smart
clients often carry this logic, which is another reason distributed systems favor them — the
integrity check happens at the endpoint that can best retry or re-route.

**Interview signal.** A strong answer ties topology to a *requirement*, not a preference:
"zero-downtime stateful streaming → leader–follower, because failover is promotion of an
in-sync replica, not a checkpoint restore." It explains why leader election needs a witness
(split-brain), knows Kafka went ZooKeeper → KRaft and that ClickHouse Keeper is a ZooKeeper
alternative, and correctly frames failure detection as heartbeat/poll-based (you can't be
*pushed* news of a node that can no longer push).
