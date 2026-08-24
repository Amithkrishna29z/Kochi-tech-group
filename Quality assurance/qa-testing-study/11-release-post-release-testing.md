# 11 — Release & Post-Release Testing

This is where the roundtable spent the most time — the practical, ground-experience end of testing. It groups the checks that run **around a release** and the ones that run **after** it in production.

## Smoke testing

A **smoke test** is a **prima-facie / sanity subset** — a quick "does it catch fire when we turn it on?" check. Run it:

- when a **new release** comes, or
- when a developer **modifies an existing system**, to confirm **no side effects** before pushing / raising a PR.

Mechanics and debates:

- Often **gated by a git hook** — if the smoke test fails, the **pipeline fails** and the push is blocked.
- **Granularity varies:** some run smoke tests against the **whole application**; some against **just the modified module.**
  - > **Debate:** whole-app confidence (slower, catches cross-module surprises) vs. module-level confidence (fast, but may miss side effects elsewhere). No single right answer — it's a speed-vs-coverage trade-off.
- **Composition varies:** a smoke test can be a **subset of integration tests**, or just the **critical-functionality unit tests.** The point is *breadth over depth* — touch the important paths quickly, don't exhaustively verify anything.

## A/B testing

A/B testing borrows the language of the **randomized controlled trial (RCT)** — the same framing as medical drug testing:

- New pages = **treated pages** (the "drug").
- Original pages = **control pages** (the "placebo").

You **stage the rollout of traffic**: route ~**5%** of traffic to the treated pages, then **10%**, then **25%**; if no issues surface, keep going and eventually **phase the A/B tool out of the pipeline.**

**Tools:**

- **Amazon Weblab** — Amazon's **home-grown** A/B system. Many companies **build their own** A/B infrastructure.
- **Firebase Remote Config** and **Optimizely.**
  - > **Nuance:** Firebase isn't an "A/B tool" *per se* — but its **config / variant / percentage-rollout / rule-based-engine** capabilities let you do A/B. Think of it as a **backend service that hands out config toggles and variants**, with percentage rollout and rule-based routing.

**Two clarifications that matter:**

1. **A/B testing has nothing inherently to do with "new feature release."** It's about **comparing variants** — which could be two designs, two prices, two copy variations. New-feature rollout is just one *use* of it.
2. **Where it operates:** A/B routing can happen at **render time** (SPA / browser side), *or* **after SSL termination** via **content-based / user-based routing** at the edge/load balancer. **Be cautious near payment flows** — you don't want to A/B a checkout in a way that risks transactions.

## Regression testing

**Regression testing** runs **before release** and is described as **"smoke test plus-plus"**:

- Both **functional/automation** tests **and** the smoke test are **re-run** — more than smoke, aiming to prove nothing that worked before is now broken.
- > **Established payoff:** if your **up-front test cases are well covered**, **regression bugs drop.** Regression testing is only as good as the case coverage you built earlier — it re-runs what you already have.

## The performance family

Five related-but-distinct kinds of performance test. Keep them straight — the differences are the whole point.

### Average Load Test

Does the system meet its **SLA under normal expected load, all the time?**

- Relate to **time-triggered systems**: e.g. an **attendance system** that peaks **morning and evening** and is quiet in between. "Normal load" isn't flat — it has a known shape.

### Stress Test

Push **beyond normal** load to see how the system behaves under strain — where it slows, degrades, or errors.

- > **Distinguish from Chaos testing.** **Chaos testing / "Chaos Monkey"** (Netflix pulling cables, killing instances at random) is about **resilience to failure**, not load. Aside from the roundtable: Chaos Monkey **once seemed silly** and **later became a big deal.**

### Spike Test

Test **sudden bursts** — find the **spiking points** and how to **scale for them.**

- > **Safety note:** it is **safest to write the scaling policy up-front**, because **unrecognized spike patterns can crash the whole system** before autoscaling reacts.

### Breakpoint Test

**Increase load until the system breaks** — to find its **limits / capacity.** This doubles as a **capacity-planning** exercise.

#### Detailed case study (Sarath's Go service)

A concrete breakpoint test, worth studying end to end:

- **SUT:** a **Go-based identity / workflow service** on a **single heavy node** (deliberately **no replicas**, to measure one node's true capacity).
- **Method:** a **declarative load-testing tool** with **virtual concurrent users**, hitting a set of **2–3 APIs that together form a transaction.**
- **Result:** found the breakpoint at **~a few hundred requests/sec** — **lower than hoped.**
- **Root-cause analysis:**
  1. **Go's garbage collector (GC) was jerky.**
  2. A **1 GB array was allocated in `main()` at startup** — memory **spiked before any requests even arrived.** After fixing that, **memory was smooth** at the start.
  3. Then, under **sustained load, memory grew and the service broke** — a second, load-dependent problem.
  4. **Go's CPU always spikes** due to **virtual threads / goroutines** — expected behavior to account for.
- **Doubling as capacity planning:** find **req/sec per node** → derive **scaling factors** → meet an expected **SLA of ~3000 total across ~10 nodes.**

> The lesson: a breakpoint test isn't just "when does it fall over?" — it's a *diagnostic and planning* tool. Finding the number is step one; **root-causing why the number is what it is** (startup allocation, GC behavior) is where the value is.

### Soak Test

A **long-duration run** to find **memory leaks, resource leaks, and gradual performance degradation** that only appear over time.

- Keep **dashboards and metrics open in parallel** and **compare against past metrics** — the signal is the *slow drift*, which you can only see over hours/days and against a baseline.

### Summary table

| Test | Question it answers | Watch out for |
|------|--------------------|---------------|
| **Average Load** | Do we meet SLA under normal load, always? | Load has a *shape* (peaks) |
| **Stress** | How do we behave beyond normal? | Don't confuse with Chaos (failure, not load) |
| **Spike** | Can we handle sudden bursts? | Write scaling policy *up-front* |
| **Breakpoint** | Where do we break / what's our capacity? | Root-cause the number; plan capacity |
| **Soak** | Do we leak / degrade over time? | Needs baselines + long runs |

## Scaling in vs out (and where SRE comes in)

Performance testing feeds directly into **scaling policy**:

- **Scaling out** (adding machines) is **easy.**
- **Scaling in / shrinking** is **harder** — you must **route traffic off a machine, drain its current requests, then free it** — non-disruptively. **Someone has to trigger this.**
- This is where **SRE (Site Reliability Engineering)** comes in: **scripts + correct infrastructure config**, done **non-disruptively**, and **configured at the load balancer.**

> **Takeaway:** growing is trivial; *shrinking safely* is the real engineering — and it's an operational (SRE) discipline, configured at the load balancer, not an afterthought.

## Mutation testing

A different kind of check: **who tests the tests?**

- **Method:** **introduce bugs (mutants) into the code** — e.g. **comment out or alter a line** — then **run the test suite** and verify the tests **catch the mutant** (a test should now fail). If no test fails, your tests weren't really checking that code.
- **Tools:** **Stryker** — **`Stryker.NET`** (C#) and **StrykerJS** (TS/JS).
- **Motivating quote:** *"a bug of an automation cannot be caught by another automation."* Mutation testing addresses exactly this — it's the tool that checks whether your *automated tests* actually work.
- **The reasoning behind heavy automation:** *"we can't fully trust humans; machines do a better job."* Mutation testing turns that logic on the tests themselves — machines checking the machine-checks.

> Mutation testing is the natural companion to the coverage discussion in [module 12](12-coverage.md): coverage tells you a line was *executed*; mutation testing tells you whether a test would *notice* if that line were wrong. They answer different questions.

## Check your understanding

1. When do you run a smoke test, and what is it a subset *of*? Summarize the whole-app vs module-level debate.
2. Explain A/B testing using the RCT vocabulary (treated/control) and the staged 5→10→25% rollout. Why is Firebase "not an A/B tool per se" yet usable for A/B?
3. Why is A/B testing *not* inherently about new-feature release? Where can A/B routing physically happen, and why be careful near payments?
4. Why is regression testing called "smoke test plus-plus," and what determines how well it works?
5. Define each of: average load, stress, spike, breakpoint, soak. For each, give the one thing to watch out for.
6. Walk through the Go breakpoint case study. What were the two distinct memory problems, and how did the test double as capacity planning?
7. Why is scaling *in* harder than scaling *out*, and what does SRE contribute?
8. What does mutation testing check that code coverage cannot? Explain "a bug of an automation cannot be caught by another automation."
