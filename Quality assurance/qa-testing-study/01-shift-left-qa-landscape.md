# 01 — Shift-Left Development & the QA Landscape

This is the umbrella module. It names every piece of the landscape that the rest of the guide expands. Read it first.

## 1. Functional vs non-functional requirements

Every system has two kinds of requirement:

- **Functional requirements** — *what the software does.* "The user can raise a purchase order." "The report totals correctly." Features.
- **Non-functional requirements** — *how well it does it, and everything around doing it.* Quality, performance, security, reliability, maintainability, usability.

Historically, software design **over-emphasized functional requirements**. You built the features; quality and security were someone else's problem, handled later.

## 2. Shift-right (the accidental default)

Agile and Scrum pushed a **feature-driven** style of work: build a **Minimum Viable Product (MVP)** first, ship it, then add the "bells and whistles" release by release. This is good — it gets value to users fast and gets feedback early.

But it has a side effect. When the pressure is always "ship the next feature," the non-functional work — the testing, the security hardening, the performance tuning — keeps getting pushed to the *end* of the lifecycle, or to "later." That is **shift-right development**: quality concerns handled late.

Shift-right is rarely a decision anyone makes on purpose. It is what happens by default when features have all the power and quality has none.

## 3. Shift-left (the correction)

Companies eventually learn — usually the expensive way, through lost profitability, outages, breaches, or unmaintainable code — that neglecting non-functional requirements costs more than it saves. So they push the emphasis on quality, testing, and security back toward the **start** of the lifecycle. That is **shift-left development.**

> **Established practice:** Quality Assurance and security are among the most important elements of shift-left. The whole point of this guide is to teach the QA half of it.

Shift-left does not mean "test more at the end." It means the *design* accounts for quality from the first day: you write tests as you build, you think about security while designing the API, you plan for load before you have load.

### The automation mindset (the meta-theme)

Underneath shift-left is a habit of mind worth stating on its own:

> **Whenever there is a repeated manual process, and you have become competent enough at it to know it cold — automate that stage.**

Automating a stage you have mastered frees your brain to move to the next problem. This is *why* test automation matters, but it is bigger than testing: it is how strong developers operate in general. Keep this in mind every time this guide discusses "automating" something.

## 4. The three pillars of QA

Whenever you set up quality assurance in an organization, you are really doing three things:

1. **Test Management** — the integrated strategy for how testing happens.
2. **Test Content Development** — deciding *what* to test (the cases, the suites); for automation you must also *write the code* that runs them.
3. **Test Execution Strategy** — actually running the tests and **reporting** results.

> **One team's opinion / organizational reality:** Test management is shaped by **politics** — specifically, where **power** sits. In SaaS/product companies, developers tend to hold the power (they file the PRs, they own the pipeline). In services companies, testers and support staff often hold more power. An *integrated* test-management strategy is only possible where the party that should own it actually has the authority to enforce it. This is not a technical fact; it is a lived reality that shapes what QA you can actually implement.

## 5. The System / Application / Device Under Test

A single idea underpins **all** test design:

- **SUT — System Under Test**
- **AUT — Application Under Test**
- **DUT — Device Under Test**

Whatever you are testing, name it precisely. Every **test case** and every **test suite** is designed *around* the thing under test — its boundaries, its inputs, its dependencies, its expected outputs. This sounds trivial, but it is fundamental: most confusion in testing (including the great "what is integration testing?" argument in [module 06](06-integration-testing.md)) comes from people silently drawing the boundary of the "thing under test" in different places.

> **Rule of thumb:** Before you write a test, be able to say out loud exactly what your SUT/AUT/DUT is and where its edges are. If you can't, you're not ready to write the test.

## 6. The full testing taxonomy

Here is the map. The rest of the guide is a walk through it. Nothing here is meant to be memorized yet — this is the table of contents for the discipline.

| Stage | What it tests | Guide module |
|-------|---------------|--------------|
| **Unit testing** | The smallest unit of code in isolation | [02](02-unit-testing.md) |
| **TDD / BDD** | *How* you drive development with tests | [03](03-test-driven-development.md), [04](04-behavior-driven-development.md) |
| **API testing** | Endpoints / SDKs / call-level interfaces | [05](05-api-testing.md) |
| **Integration testing** | Modules / services / systems working together | [06](06-integration-testing.md) |
| **UI / UX testing** | Behavior surfaced through the interface | [07](07-ui-ux-testing.md) |
| **Static analysis / instrumentation** | Code quality & security without (static) or with (dynamic) running it | [08](08-static-analysis-instrumentation.md) |
| **Automation vs functional (manual)** | The overall execution strategy | [09](09-automation-vs-functional.md) |
| **User Acceptance Testing (UAT)** | Stakeholder sign-off | [10](10-user-acceptance-testing.md) |
| **Release & post-release** | Smoke, A/B, regression, load, stress, spike, breakpoint, soak | [11](11-release-post-release-testing.md) |
| **Coverage** | How much of the above is *really* enough | [12](12-coverage.md) |

## 7. The cycle (it's a loop)

Do not read the taxonomy as a straight line. It is a **loop**, because the system *evolves*:

```
units built (TDD/BDD) → API tests → integration tests (+ UI/UX via Selenium etc.)
   → static analysis → to production → automation / UAT / release / post-release
   → (features & fixes change the system) → back to units …
```

The whole reason unit tests exist, for example, is that you will come back around this loop again and again — and each time you do, the tests you wrote last time protect you from breaking what already worked.

## 8. Watch for bias

A theme repeated throughout the roundtable, and worth planting here at the start:

> **Each engineer's definitions are colored by their platform.** A server-side .NET/Java developer, a mobile Swift developer, and a load/QA specialist will define the *same words* differently — and each is "right" from where they stand. As you learn this material, practice noticing these biases: in the authors of this guide, and in your own colleagues, and in yourself.

The clearest example is coming in [module 06](06-integration-testing.md). Keep the idea of bias in your pocket until then.

## Check your understanding

1. In your own words, what is the difference between shift-left and shift-right? Which one is a *decision* and which one is a *default*?
2. Give an example of a non-functional requirement for an app you've worked on. Was it handled early or late?
3. What are the three pillars of QA, and which one is most affected by organizational politics?
4. Why is it a mistake to start writing a test before you can name your SUT/AUT/DUT?
5. State the "automation mindset" rule in one sentence, and give one example *outside* of testing where it applies.
6. Why is the testing taxonomy described as a loop rather than a line?
