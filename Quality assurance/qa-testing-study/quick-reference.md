# Quick Reference / Cheat Sheet

Decision tables and one-liners. For depth, follow the links back to the numbered modules.

## The testing taxonomy at a glance

| Stage | Question it answers | Module |
|-------|--------------------|--------|
| Unit | Does this smallest piece work in isolation? | [02](02-unit-testing.md) |
| API | Does the endpoint/SDK/contract behave? | [05](05-api-testing.md) |
| Integration | Do modules/services cooperate correctly? | [06](06-integration-testing.md) |
| E2E integration | Do multiple *systems* work end to end? | [06](06-integration-testing.md) |
| UI/UX | Does it behave through the interface? | [07](07-ui-ux-testing.md) |
| Static analysis | Is the code clean/secure without running it? | [08](08-static-analysis-instrumentation.md) |
| UAT | Does the stakeholder accept it? | [10](10-user-acceptance-testing.md) |
| Smoke | Any obvious breakage on this release/change? | [11](11-release-post-release-testing.md) |
| Regression | Did we break anything that worked before? | [11](11-release-post-release-testing.md) |
| Performance family | Does it hold up under load/stress/time? | [11](11-release-post-release-testing.md) |

## Shift-left vs shift-right

| | Shift-left | Shift-right |
|--|-----------|-------------|
| Quality/security handled | Early, by design | Late, bolted on |
| Nature | A deliberate decision | The accidental default of feature-driven work |
| Driven by | Learning that neglect hurts profitability | MVP-first, "bells and whistles later" |

## TDD vs BDD

| | TDD | BDD |
|--|-----|-----|
| What it is | A dev philosophy (test-first) | Behavior specs against a black box |
| Author knows internals? | Yes | No — describes intent only |
| Language | Plain test code | Gherkin (Given/When/Then) |
| Weight | Light / non-intrusive | Heavier (spec + step classes) |
| Best for | Developers driving their own code | Non-developers/QA specifying behavior; living docs |
| Cycle | Red → Green → Refactor (RGB) | Given → When → Then |

## Fact vs Theory (xUnit)

| | Fact | Theory |
|--|------|--------|
| Parameters | None | Yes (inline data or data-provider class) |
| Checks against | Hard-coded values | Many data points |
| Logic analogy | Proposition | Predicate |

## API testing: which thing are you testing?

| Approach | You call | Example |
|----------|----------|---------|
| Web API | The HTTP endpoint directly | Postman/SoapUI/xUnit hitting an endpoint |
| SDK | A generated client library | A generated `.dll` / `.jar` |
| Hybrid | A call-level interface wrapping REST | Informatica MDM Hub's generated `.jar` |

**API vs SPI:** API = what the **consumer** calls; SPI = what the **provider** implements. The divide is real in **OEM/product** companies (one API call → many implementations) — that's where extensive API testing matters most.

## Integration testing: pick your definition deliberately

1. Name your **SUT**: modules *within* a system, or your system *against another*?
2. **Mock a boundary** → deterministic, automatable integration test.
3. **True multi-system E2E** → non-deterministic → often manual.
4. Invest where **your** bugs cluster (leaf/units vs interior/integration points).

## UI automation: how findable is your element?

| | Developer-aware | Hostile-automatable |
|--|-----------------|---------------------|
| How elements are found | You provide IDs / accessibility labels | OS introspection or image processing |
| Reliability | High | Fragile |
| Examples | XCUITest labels, Selenium `By.id`, VoiceOver labels | Windows/X-Windows native introspection; Ranorex image processing |

**Best thing a developer can do:** add stable IDs / accessibility labels. Bonus: it also enables assistive tech.

## The performance family

| Test | Answers | Watch for |
|------|---------|-----------|
| Average Load | Meet SLA under normal load, always? | Load has a *shape* (peaks) |
| Stress | Behavior beyond normal? | ≠ Chaos (that's failure, not load) |
| Spike | Handle sudden bursts? | Write scaling policy **up-front** |
| Breakpoint | Where do we break / capacity? | Root-cause the number; plan capacity |
| Soak | Leaks / degradation over time? | Needs baselines + long runs |

**Scaling:** out = easy; in = hard (drain requests, free machines, non-disruptively) → **SRE** at the load balancer.

## A/B testing in one box

- **Treated** (new) vs **control** (original) pages — RCT language.
- Stage traffic: **5% → 10% → 25%**, then phase the tool out.
- Not inherently about new features — it's about **comparing variants.**
- Can run at **render time** (SPA) or **after SSL termination** (content/user routing). **Careful near payments.**
- Tools: **Weblab** (Amazon home-grown), **Firebase Remote Config** (config/variants/percentage/rules), **Optimizely.**

## Coverage: don't fool yourself

- **Code coverage** = lines executed. **Gameable** (touch a line, assert nothing).
- **100% code coverage ≠ correctness.** Sufficiency of **test cases** is what matters.
- Test the **public contract top-down** → covers internals *and* stays refactorable.
- **Don't** make privates public to hit a number.
- Healthy practice: **document cases at planning time → verify at review → gate-keep.** No gate = tooling is useless.
- **Mutation testing** (Stryker) checks whether tests would *notice* a bug.

## "Which test do I write?" decision flow

1. Smallest piece of logic in isolation? → **Unit** ([02](02-unit-testing.md))
2. An endpoint / SDK / contract? → **API** ([05](05-api-testing.md))
3. Modules/services cooperating? → **Integration** (mock a boundary to make it deterministic) ([06](06-integration-testing.md))
4. Behavior only visible on screen? → **UI/UX** (Selenium/XCUITest) ([07](07-ui-ux-testing.md))
5. Code cleanliness/security without running? → **Static analysis** ([08](08-static-analysis-instrumentation.md))
6. App too immature to script? → **Functional/manual** for now ([09](09-automation-vs-functional.md))
7. Stakeholder acceptance? → **UAT** ([10](10-user-acceptance-testing.md))
8. About to release? → **Smoke → Regression** ([11](11-release-post-release-testing.md))
9. Worried about load/capacity/leaks? → **Performance family** ([11](11-release-post-release-testing.md))
10. Do the tests themselves work? → **Mutation testing** ([11](11-release-post-release-testing.md)), and mind **coverage** ([12](12-coverage.md))

## Recurring truths to remember

- **Automate any mastered, repeated manual process** — beyond testing.
- **Name your SUT/AUT/DUT** before writing any test.
- **Notice bias** — definitions are colored by platform (server vs mobile vs load/QA).
- **Where power sits** shapes what test management is even possible.
- **A gate is essential** — tooling without gate-keeping is useless.
- **There is no perfectly-automated regime** — automation is an improvement, not the ideal.
