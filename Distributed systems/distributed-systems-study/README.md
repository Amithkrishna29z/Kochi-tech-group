# Distributed Computing — A Holistic Study Pack

A self-contained study pack for **reasoning about and constructing distributed applications
and systems holistically**, rather than memorizing one tool. It is distilled from a
practitioners' conversation (enterprise computing / data engineering) and expanded, verified,
and structured for study.

**Audience.** A backend developer (Java/Spring background, broadening stack) preparing for
interviews and real production work — solid programming knowledge, limited hands-on
distributed-systems experience.

**Core thesis.** Three activities get conflated — *distributed application programming*,
*distributed data processing*, and *distributed systems engineering*. Learn the **lenses**
(execution model, parallelism type, DAG, coordination topology, processing strategy, client
model) and any new tool becomes readable at a glance.

## Files

| # | File | What's inside |
|---|---|---|
| — | [README.md](README.md) | This index, learning path, prerequisites, verification flags |
| 01 | [01-taxonomy.md](01-taxonomy.md) | The mental model: three activities + six lenses for any system |
| 02 | [02-data-processing-frameworks.md](02-data-processing-frameworks.md) | MapReduce, Hadoop/YARN, Spark, Storm, Akka, Samza |
| 03 | [03-functional-programming-foundations.md](03-functional-programming-foundations.md) | Referential transparency, the six villains, Monoid→Functor→Monad→Applicative→Effects |
| 04 | [04-coordination-and-topology.md](04-coordination-and-topology.md) | Master-slave, leader-follower, multi-master; heartbeats, witnesses, election |
| 05 | [05-databases-and-wire-protocols.md](05-databases-and-wire-protocols.md) | CLI vs wire protocol, JDBC/ODBC, ClickHouse case study |
| 06 | [06-case-studies.md](06-case-studies.md) | Four anecdotes reframed as engineering lessons |
| 07 | [07-glossary.md](07-glossary.md) | Every term, concisely defined |
| 08 | [08-flashcards.md](08-flashcards.md) | Atomic Q/A pairs for spaced repetition |
| 09 | [09-interview-questions.md](09-interview-questions.md) | Practice questions + model answers + "what they're testing" |
| 10 | [10-reading-list.md](10-reading-list.md) | Books, courses, papers with why-to-read notes |

## How to use this pack

- **First pass (understanding):** read 01 → 02 → 03 → 04 → 05 in order. They build on each
  other; each ends with an **"Interview signal"** note telling you what a strong answer sounds
  like.
- **Grounding:** [06 — Case Studies](06-case-studies.md) turns the concepts into concrete
  decisions. Read it after 04.
- **Retention:** drill [08 — Flashcards](08-flashcards.md) daily; keep
  [07 — Glossary](07-glossary.md) open as a quick reference.
- **Interview prep:** rehearse [09 — Interview Questions](09-interview-questions.md) out loud,
  leading with the lens/mechanism before naming any product.
- **Going deeper:** follow the suggested order in [10 — Reading List](10-reading-list.md).

Every file stands alone but cross-links to the others, so you can also enter from a flashcard
or glossary term and follow links to the full treatment.

## Prerequisite checklist

You'll get the most from this pack if you're comfortable with:

- [ ] Writing and reasoning about functions, lambdas/closures, and generics.
- [ ] Basic concurrency: threads, blocking vs non-blocking, race conditions.
- [ ] Networking basics: TCP connections, ports, request/response, latency vs throughput.
- [ ] SQL: `SELECT`, `GROUP BY`, aggregation, joins (used as an analogy in 03 and 05).
- [ ] Relational databases and a driver (JDBC or equivalent) at a using-it level.
- [ ] Comfort reading small Scala/pseudocode snippets (used in 03).

If several are unchecked, skim the early chapters of *Designing Data-Intensive Applications*
(see [10](10-reading-list.md)) alongside file 01.

## Conventions

- **Concepts vs anecdotes.** Durable concepts are taught as general truth. One team's specific
  experiences appear only in [06 — Case Studies](06-case-studies.md) (and a marked passage in
  [05](05-databases-and-wire-protocols.md)), framed as illustrative lessons, not universal
  recommendations.
- **`[unverified — check]`** marks a claim that is a source characterization or teaching
  analogy rather than a documented fact — verify before repeating it as fact.
- **No hallucinated APIs.** Framework/database features described are grounded in documented
  behavior; where a specific configuration/offering could drift over time, the text says so.

## Verification & flagged items

Per the pack's requirements, source names/attributions were cross-checked. Results:

**Corrected (were approximate in the source, now fixed):**

- *Designing Data-Intensive Applications* — author is **Martin Kleppmann** (source said
  "Klubman/Klubmann"). ✅ corrected in [10](10-reading-list.md).
- *Database Internals* — correct title (source said "Database Systems"), by **Alex Petrov**.
  ✅ corrected in [10](10-reading-list.md).

**Verified (as stated):**

- *Patterns of Distributed Systems* — **Unmesh Joshi**, Addison-Wesley Signature Series
  (Fowler), 2023; grew from martinfowler.com articles. ✅
- **MIT 6.824** Distributed Systems — **Prof. Robert Morris**; paper-driven; labs in Go. ✅
- **NPTEL Distributed Systems** — **Prof. Rajiv Misra, IIT Patna** (PhD from IIT Kharagpur,
  which likely explains the "Roorkee/Kharagpur/Patna" uncertainty in the talk). Instructor and
  institution confirmed. ✅
- **Kafka coordination** — historically **ZooKeeper**; **KRaft** (KIP-500) removes the
  ZooKeeper dependency; **ClickHouse Keeper** is a ZooKeeper-compatible alternative. ✅

**Left marked `[unverified — check]` in the text (double-check before repeating as fact):**

- The **"Spark = distributed reverse-LINQ / distributed Rx"** framing
  ([02](02-data-processing-frameworks.md)) — a useful teaching analogy, not Spark's official
  positioning.
- The **ClickHouse-proxy-replaces-Spark/Hadoop** architecture
  ([05](05-databases-and-wire-protocols.md)) — one team's real solution; fit depends on
  workload shape and the storage-compute coupling trade-off, not a universal claim.
- All four **[06 — Case Studies](06-case-studies.md)** — the *situations* are one team's
  anecdotes; the *lessons* are general.
- The exact **NPTEL offering/course number** — instructor/institution are confirmed, but NPTEL
  re-runs courses under different codes; confirm the specific run if you enroll.

**Not independently re-verified here (well-established, stated from domain knowledge):** the
MapReduce/GFS/RDD/Raft papers and their authors, and the wire-protocol facts (SQL\*Net for
Oracle, TDS for SQL Server, Kafka's proprietary format, ClickHouse native + HTTP ports). These
are standard and documented; consult the primary docs in [10](10-reading-list.md) if you need
to cite them precisely.
