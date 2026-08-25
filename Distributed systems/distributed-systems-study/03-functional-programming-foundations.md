# 03 — Functional-Programming Foundations

> Part of the [Distributed Computing Study Pack](README.md). This file explains *why* the
> correctness rules in [02 — Data-Processing Frameworks](02-data-processing-frameworks.md)
> exist. Analogy back to databases connects to
> [05 — Databases & Wire Protocols](05-databases-and-wire-protocols.md).

## The problem this file solves

[02](02-data-processing-frameworks.md) told you Spark transformations must be *pure* — no IO,
no service calls, no null, no exceptions, no threads, no nondeterminism. That looks like an
arbitrary list of prohibitions. It is not. Each prohibition is there because distributed
engines rely on functional-programming properties to run your code safely on many machines
and re-run it after failures. This file connects the six "villains" to the FP machinery that
replaces them, and to the algebraic hierarchy (Monoid → Functor → Monad → Applicative →
Effects) that explains what different stream operators can and cannot do.

## Referential transparency: the one property that matters

An expression is **referentially transparent** if you can replace it with its value without
changing the program. Equivalent phrasing: **same input → same output, always, with no
observable side effects.** A referentially transparent function can be:

- **retried** safely (Spark task failure → re-run),
- **run speculatively** (two copies race; either result is fine),
- **reordered / parallelized** across partitions and machines,
- **cached / recomputed** from lineage.

Every one of those is a distributed-execution feature. So referential transparency is not
FP dogma here — it is the *contract* that makes distributed execution correct.

## The six villains and their FP remedies

| Villain | Why it breaks distribution | FP remedy |
|---|---|---|
| **IO** | Not repeatable; re-run doubles the effect | Push effects to the edges; keep the core pure |
| **Service/network calls** | Same as IO + partial failure + latency | Do lookups before/after; broadcast static data; idempotency |
| **null** | Hidden absence → NPE on some partitions only | **Option / Maybe** monad, or Null Object pattern |
| **Exceptions** (as control flow) | A thrown exception on one partition diverges from others | **Either / Try** — make errors *data* |
| **Threads** | Non-deterministic interleaving; the engine already owns parallelism | Let the framework parallelize; keep operators single-threaded |
| **Nondeterminism** (random, clock, order) | Re-run yields a different answer; lineage recovery corrupts | Pass seeds/timestamps *in* as data; sort explicitly |

The three data-shaped remedies deserve a closer look, because they are the gateway to the
hierarchy:

- **Absence** — instead of `null`, return `Option[A]` (`Some(a)` / `None`). The absence is
  now a value you must handle, checked by the type system, identical on every partition.
- **Errors** — instead of `throw`, return `Either[Error, A]` (`Left(err)` / `Right(a)`) or
  `Try[A]` (`Success` / `Failure`). Errors become ordinary data that flow through the
  pipeline like any other record.
- **Effects** — genuine mutation/IO is *allowed*, but only in a place explicitly marked as
  effectful (an `IO`/effect type, or the collecting **action** in Spark), never smuggled
  inside a transformation.

## The algebraic hierarchy

These abstractions form a ladder. Each rung adds one capability. You do not need category
theory; you need to know *what each rung lets you do to a stream/collection*.

### Monoid — "aggregatable"

A **Monoid** is a type with:
1. an **identity** element (`empty`), and
2. an **associative** binary operation (`combine`), such that `combine(a, empty) == a` and
   `combine(combine(a,b),c) == combine(a,combine(b,c))`.

Examples: integers under `+` (identity `0`), strings under concatenation (identity `""`),
lists under append. **Associativity is what makes distributed aggregation legal** — you can
combine partial results per machine in any grouping and get the same total. Identity is what
lets an empty partition contribute nothing. This is exactly `reduce` in MapReduce.

```scala
trait Monoid[A] {
  def empty: A
  def combine(x: A, y: A): A
}
val intSum = new Monoid[Int] { def empty = 0; def combine(x: Int, y: Int) = x + y }
// distributable: partitions reduce locally, then results combine — any order is fine
```

### Functor — Monoid's world + `map`

A **Functor** is anything you can `map` over with a function, preserving structure:
`map(f): F[A] => F[B]`. Lists, `Option`, `Future`, Spark RDD/DataFrame are functors. `map`
transforms each element **without changing the shape** of the container and without letting
elements interact. This is data-parallelism in one word.

```scala
List(1,2,3).map(_ * 2)        // List(2,4,6)
Option(5).map(_ + 1)          // Some(6)
Option.empty[Int].map(_ + 1)  // None  — absence flows through safely
```

### Monad — Functor + `flatMap` (sequential composition)

A **Monad** adds `flatMap` (a.k.a. `bind`): `flatMap(f): F[A] => (A => F[B]) => F[B]`. It
lets each element produce a *new sub-computation* and then flattens the nesting. Two powers
come with it:

- **sequential composition** — step 2 depends on the *result* of step 1 (the essence of a
  pipeline, and of `for`-comprehensions / async chains).
- a **Cartesian-product** flavor — `flatMap` over collections expands combinations
  (every `a` paired with every `b`).

```scala
for {
  user  <- findUser(id)        // Option[User]
  email <- user.primaryEmail   // Option[Email]  — short-circuits on None automatically
} yield email                  // Option[Email]
```

`Option`, `Either`, `Try`, `Future`, `List`, and Rx `Observable` are all monads — which is
why the *same* `flatMap` code composes absence, errors, async, nondeterminism, and streams.

### Applicative — combining independent computations

An **Applicative** sits between Functor and Monad in power but enables something Monad's
sequential `flatMap` does not emphasize: **combining multiple independent computations** with
`apply`/`ap` (or `mapN`). Where `flatMap` is "do B *after and depending on* A," applicative
is "do A and B independently, then merge." This is exactly what you need to **merge multiple
streams** or validate several fields at once and collect *all* errors instead of stopping at
the first.

```scala
// Independent: both run, results combined. With Validated, errors accumulate.
(validateName(n), validateAge(a)).mapN(User.apply)
```

> Ordering note: mathematically Applicative is *weaker* than Monad (every Monad is an
> Applicative). The source presented the ladder as Monad → Applicative by *capability added
> for stream work* (merging multiple streams). Both framings are fine as long as you know
> `flatMap` = sequential/dependent, `ap`/`mapN` = independent/parallel-merge.

### Effects — mutation, allowed and contained

At the top, **Effects** re-admit mutation and IO — but wrapped in a type (`IO`, `Task`, a
Spark **action**) that keeps them *described* and *sequenced* rather than scattered. Effects
are where the pure core finally touches the outside world, at the edge of the program.

## The database analogy (bridge to file 05)

The same ladder appears in SQL, which is why it feels familiar:

| SQL operation | Rung |
|---|---|
| `SELECT sum(x), max(x)` — plain column aggregation | **Monoid** (associative + identity) |
| `SELECT sum(x*2 + y)` — aggregation over an expression | **Functor** (map, then aggregate) |
| Joining/merging several relations to produce rows | **Applicative** (combine independent sources) |
| `UPDATE` / `INSERT` — mutating the data | **Effects** |

If you have written SQL, you have already used every rung; the FP names just make the
distributability of each explicit. See [05](05-databases-and-wire-protocols.md) for how a
columnar database (ClickHouse) turns this aggregation-as-monoid idea into a distributed
compute engine.

## Why Rx composes event streams when plain streams cannot

A plain `Stream`/`Iterator` is a *pull* Functor+Monad: you can `map`/`flatMap` it, but you
cannot cleanly **merge two live sources**, apply **backpressure**, or handle **errors and
completion as first-class signals**. Rx's `Observable` is an *applicative, effect-aware*
stream: `merge`, `combineLatest`, `zip`, `retry`, and error/`onComplete` channels exist
precisely because the abstraction is higher on the ladder. That is also why **Rx's error
messages reveal structure** — a stack trace through `flatMap`/`merge`/`combineLatest` tells
you exactly how the streams were wired. This is the composition backbone behind the
"distributed Rx" framing of Spark streaming in [02](02-data-processing-frameworks.md).

## Bringing it back to Spark

- `map`/`select` → **Functor** — parallel, no interaction between rows.
- `flatMap`/dependent joins → **Monad** — sequential/dependent composition.
- multi-source joins, merging streams → **Applicative**.
- `reduce`/`aggregate` → **Monoid** — associative combine across partitions.
- `write`/`collect`/`foreach` → **Effects** — the action at the edge.

Keep the transformation core on the Functor/Monad/Monoid rungs (pure), and confine Effects to
actions. Do that and Spark's fault tolerance holds; violate it and you get the "degrades to a
scheduler" failure from [02](02-data-processing-frameworks.md).

**Interview signal.** A strong candidate can state referential transparency in one sentence,
explain *why each* of the six prohibitions follows from it (retry/speculation/lineage), and
map at least Monoid (associative aggregation) and Monad (`flatMap` = dependent composition)
onto both a stream API and SQL. Bonus: explaining that Applicative merges independent
computations (accumulate all validation errors; merge multiple streams) whereas Monad chains
dependent ones.
