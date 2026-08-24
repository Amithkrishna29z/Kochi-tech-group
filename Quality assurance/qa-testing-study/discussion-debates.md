# Discussion & Debates

This roundtable was a *real* conversation between working engineers who disagreed. Those disagreements are pedagogically valuable — they show that testing is not a settled set of rules but a set of trade-offs shaped by context. Each debate below is presented **fairly, from both sides**, with the working context (the **bias**) that produced each view.

> **How to read this file:** don't look for who "won." Look for *why* each person believed what they did — usually it traces to the platform they work on or the kind of company they work in.

---

## Debate 1 — What is "integration testing"? (the big one)

**The disagreement:** three engineers used "integration testing" to mean three different things.

| View | Held by | Integration testing means… |
|------|---------|----------------------------|
| Server / enterprise / .NET-Java | Sreejith | Interaction **between modules** (e.g. Accounting + Inventory behind a Purchase Order); your DLLs working together. |
| Enterprise-integration | — | Interaction between **two independent systems** (SAP, Salesforce, a bus). Distinguishes "integration testing" (modules) from "testing of an integration" (systems). |
| Mobile-first | the Swift/iOS dev | The **app calling a server** *is* integration testing — which the server folks called **E2E**. |

**The resolution:** *"It depends on whose eyes you view the world through."* If you expose an API, the consumer integrates *to* you; you only care about your own modules coming together. The group's compromise: **integration testing = components/modules/services**; **E2E integration = two+ systems across form factors.**

**The bias:** each definition is exactly what you'd expect from that person's platform. This is the guide's headline example of platform bias. Full treatment in [module 06](06-integration-testing.md).

---

## Debate 2 — How much integration testing? (leaf vs interior node)

**Camp 1 — unit tests primary, integration minimal.** Test integration only at a contract level / one level of consumers; cover the rest with unit tests, because they're *faster and cheaper.* *"If the atoms are correct, the molecules assemble correctly."*

**Camp 2 — integration is where the costly errors live.** Errors at **interior nodes** (integration points) are the expensive ones; a unit "either works or not," but integration is where failures **hide.** Static analysis at the pure-code level is *less* useful than testing at integration points.

Both sides invoke **leaf vs interior node**: Camp 1 says leaves are cheap and numerous, nail them; Camp 2 says the tree breaks at interior nodes, and those breaks cost more.

**Status: open question.** Practical note the group agreed on: integration testing is often still **manual**, and automating it is an **emerging** idea. Decide based on where *your* bugs cluster.

---

## Debate 3 — Is BDD too heavy for shift-left?

**"BDD is heavy" (against shift-left):** the extra language (Gherkin), step-definition classes, and parsing engine are ceremony that slows the developer versus plain, non-intrusive TDD.

**"BDD doesn't bother the developer much":** once the step library exists, new scenarios are cheap, and you gain living documentation plus **bias removal** (non-developers specify intent against a black box using fixed phrases).

**The bias:** developers who value speed and own their own tests lean toward TDD; teams that need **non-developers/QA to specify behavior** value BDD's constraints. See [module 04](04-behavior-driven-development.md).

---

## Debate 4 — Does TDD actually work everywhere?

**For:** TDD is a disciplined reverse-mapping/self-reasoning tool; adopted as an enterprise *habit* it improves quality.

**Against (honest reality):** **not everyone reasons logically by default**; the mental model doesn't click without built procedural knowledge, so **TDD doesn't work out everywhere.** Several practitioners admitted their real products had **no TDD, sometimes no unit tests at all.**

**Takeaway:** TDD adoption is a **cultural** project, not just a technical switch. See [module 03](03-test-driven-development.md).

---

## Debate 5 — Should you consume REST services directly?

**Against (one engineer's firm view):** directly consuming REST services is **not good practice** — better options exist (SDKs, mediators, plugins). Non-modular services force consumers onto a client SDK anyway.

**For (the caveat):** direct consumption **works fine when provider and consumer are "in the same shop"** and there is **no API–SPI divide** — you control both ends and no third-party provider plugs in behind the API.

**The bias:** **OEM/product** thinking (where the API–SPI divide is real and one call maps to many implementations) pushes toward SDKs/mediators; **in-house** thinking tolerates direct consumption. See [module 05](05-api-testing.md).

---

## Debate 6 — Smoke test: whole app or just the modified module?

**Whole-app:** run the smoke suite across everything for confidence that a change didn't cause cross-module side effects. Slower.

**Module-level:** run only against the modified module — fast, keeps the pipeline snappy — but may miss side effects elsewhere.

**Status:** a genuine **speed-vs-coverage** trade-off with no universal answer. Granularity and composition (subset of integration tests vs critical unit tests) both vary by team. See [module 11](11-release-post-release-testing.md).

---

## Debate 7 — Does 100% code coverage mean anything?

**The gate view:** enforce 100% (feasible in TypeScript, where everything is mockable) and "no PR merges below 100%."

**The skeptic view:** coverage is **gameable** — you can *touch* a line without asserting behavior, getting the number without the testing. 100% code coverage **≠ correctness.** Merge gates create **politics**: people write **throwaway tests** to ship the feature.

**The synthesis the group reached:** document all test cases **at planning time** (functional level), verify at **review time** (reviewer's duty), design up-front, and **gate-keep** — but gate on documented cases + review, not a raw percentage. Pair with **mutation testing** to check the tests themselves. See [module 12](12-coverage.md).

---

## Debate 8 — Testing private/hidden functions

**The temptation:** make private functions public so you can test them and hit 100%.

**The rejection (Sarath, enhanced by Sreejith & Lal):** **don't** corrupt the design for a metric. Test the **public contract top-down**; if a hidden function's behavior is in the contract, top-level cases must reach it. Program defensively from the top. And accept that **legacy/third-party static-library code may be impossible to fully cover.**

**Why it matters:** top-down testing keeps code **refactorable** — the real prize, bigger than any coverage number. See [module 12](12-coverage.md).

---

## Debate 9 — Can functional (manual) testing be shifted to automation?

**The aspiration:** shift enough manual testing into automation that human testers focus only on new features.

**The reality check:** **there is no perfectly-automated software-development regime.** Automation is an **improvement over** manual testing, not the ideal complete flow. You can't automate against an **immature** app, so early manual testing is unavoidable and automation lands *after* the first release.

**The economics twist:** automation maintenance is costly — a template change can cascade to **30–100 microservices.** Microsoft amortizes migration-tool cost across worldwide users; a single company often can't, and must **co-exist with legacy.** See [module 09](09-automation-vs-functional.md).

---

## Debate 10 — Is formal UAT still needed?

**"We don't need formal UAT":** the Product Owner gets a **demo** each release (part of Definition of Done); everything else is automated, and post-PR the build is practically production-ready.

**"It depends — especially B2C":** for B2C (Amazon.com) you **can't just demo** — UAT shifts to checking **UX** (functional correctness assumed handled below). For B2C, UX can be **"business and death."**

**The bias:** internal/B2B SaaS tolerates the demo-as-sign-off; B2C at scale cannot. See [module 10](10-user-acceptance-testing.md).

---

## Cross-cutting theme: where power sits shapes everything

Several debates reduce to **organizational power**:

- In **SaaS/product** companies, **developers** hold power (they file PRs and stress about them, yet testers complain more). Integrated test management is possible when developers own it.
- In **services** companies, **testers/support staff** hold more power.
- **Developer-portal maturity** correlates with company type — product companies (T-Mobile "wonderful example," Netflix, Amazon, Google) have rich portals; pure services companies typically use the client's.

**Integrated test management is only a *possibility* where the party that should own it actually has the power to enforce it.** Many "technical" disagreements above are really about *who gets to decide.*

---

## Meta-lesson: notice your own bias

The single most repeated caution: **every engineer's definitions are colored by their platform** (server-side .NET/Java vs mobile Swift vs load/QA). The skill this guide most wants to teach is to **notice these biases** — in this material, in your colleagues, and in yourself — so that when you hear "integration testing" or "100% coverage" or "we don't need UAT," you ask: *from whose seat, and in what kind of company, is that true?*
