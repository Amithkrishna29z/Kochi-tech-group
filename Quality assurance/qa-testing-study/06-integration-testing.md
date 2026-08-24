# 06 — Integration Testing (and the big definitional debate)

> **This is the module's most important debate.** The engineers genuinely disagreed on what "integration testing" *means*, and the disagreement traces directly to their working context — their **bias**. Read this not to find "the answer," but to learn to see *why* smart people define the same term differently.

## The core problem

"Integration testing" is one of those phrases everyone uses and few define the same way. The reason is [module 01's](01-shift-left-qa-landscape.md) point about the **System/Application Under Test**: people silently draw the boundary of "the thing under test" in different places, and *then* argue.

Here are the framings that collided.

### View A — Server / enterprise / .NET-Java (Sreejith's framing)

Integration testing = verifying the **interaction between modules**.

- *Example:* an **Accounting** module + an **Inventory** module, wired together behind a **Purchase Order** automation. When a PO is raised, does data flow correctly into *both* modules?
- Also framed as: your multiple **DLLs / packaged code** working together. You wire components up *from an API* and test that they cooperate.
- The purpose: **detect side effects between units**, and decide **what actually works** before user acceptance.

### View B — Enterprise-integration

Integration = the interaction between **two independent systems**.

- *Examples:* an **SAP** configuration, **Salesforce**, external service providers, a message **bus**.
- This view draws a sharp line:
  - **"Integration testing"** = modules *within a system*.
  - **"Testing of an integration"** = *two systems* talking to each other.
- Same words, opposite scope — hence half the confusion.

### View C — Mobile-first (the Swift/iOS developer)

From the mobile app's point of view, **the app calling a server *is* integration testing.**

- The server-side engineers called that **end-to-end (E2E) testing**, not integration.
- Neither was "wrong." The resolution the group reached:

> **It depends on whose eyes you view the world through.** If you *expose* an API, then the consumer is integrating *to* you — and from your seat you only care about **your** implementation and how **your** modules come together. From the consumer's seat, calling you is the integration. Same wire, two names, depending on where you stand.

## Deterministic vs non-deterministic integration tests

This distinction is what lets you actually *automate* integration testing.

- You can write an integration test **deterministically** by **mocking the network / DB.**
  - *Mobile example:* mock the API call, then test the **calculation-logic class + the API-call class together.** Both real classes, one mocked boundary → a **deterministic integration test.** It runs the same way every time.
- When you go **true end-to-end** across multiple **real** systems, it becomes **non-deterministic** — networks flake, external systems change state, timing varies.

> **This is exactly why teams generally don't write full multi-system E2E *automated* tests.** The non-determinism makes them flaky and expensive to maintain. They mock a boundary to make integration tests deterministic, and reserve true E2E for manual or lightly-automated checks.

## The group's final compromise (teach these)

After the argument, the group settled on **working definitions** worth teaching as the practical baseline:

- **Integration testing** = integrating **components / modules / services** — getting **real data flow** between them, *or* **mocking one side deterministically.**
- **End-to-end integration** = integrating **two or more *systems***, where **different form factors** are also in play — desktop, web, mobile.

| Term | Scope | Determinism | Typical execution |
|------|-------|-------------|-------------------|
| Integration testing | Components / modules / services within a system | Deterministic (mock one side) or real data flow | Automatable |
| End-to-end integration | Two+ independent systems, multiple form factors | Non-deterministic | Often manual / lightly automated |

## The cost / coverage debate (an open question)

Even with definitions agreed, the engineers disagreed on **how much** integration testing to do. Present this as genuinely unresolved.

**Camp 1 — "unit tests are primary; keep integration minimal."**
- Test integration only at a **contract level** / one level of consumers; cover everything else with unit tests.
- Rationale: unit tests are **faster and cheaper to execute.**
- Slogan: *"if the atoms are correct, the molecules assemble correctly."*

**Camp 2 — "integration is where the costly errors live."**
- Errors at **interior nodes** (the integration points) are the expensive ones.
- Integration testing is therefore **more** important — *especially* because **static analysis at the pure-code level is less useful than testing at integration points.**
- Where do more errors actually occur? A unit "either works or it doesn't" — easy to see. **Integration is where failures hide** and you can't easily spot them.

**The "leaf vs interior node" argument** is used by *both* sides:
- Camp 1: leaves (units) are numerous and cheap — nail them and the tree is mostly sound.
- Camp 2: interior nodes (integration points) are where the tree actually breaks, and a broken interior node is costlier than a broken leaf.

> **Practical reality noted by the group:** integration testing is often done **manually** today; **automating it is an emerging idea**, not yet a solved default. So even the people arguing for "more integration testing" acknowledged it's the harder thing to automate.

## How to actually decide (for your context)

Since the definitions and the cost trade-off are context-dependent, decide deliberately:

1. **Name your SUT.** Are you testing modules *within* your system, or your system *against another system*? That alone resolves most "is this integration or E2E?" arguments.
2. **Mock to make it deterministic** wherever you can — that's what makes an integration test automatable and worth keeping in the pipeline.
3. **Reserve true multi-system E2E** for the few critical journeys, and accept it may be manual.
4. **Weigh leaf vs interior cost for *your* system.** If your bugs cluster at integration points, invest there even though it's harder.

## Check your understanding

1. State the three framings of "integration testing" (server/enterprise, enterprise-integration, mobile-first) and the working context behind each.
2. What is the difference between "integration testing" and "testing of an integration" in View B?
3. Why does "it depends on whose eyes you view the world through" resolve the mobile-vs-server disagreement?
4. What makes an integration test *deterministic*? Give the mobile calculation-logic + API-call example.
5. Why do teams generally avoid fully-automated multi-system E2E tests?
6. Summarize both camps of the cost/coverage debate, and show how "leaf vs interior node" is used by each.
7. For a system you know, decide: where do your bugs cluster — leaves or interior nodes? What does that imply for where to invest?
