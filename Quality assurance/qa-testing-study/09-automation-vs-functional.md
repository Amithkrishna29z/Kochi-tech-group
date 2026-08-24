# 09 — Automation vs Functional (Manual) Testing

## The two words, honestly

- **Functional testing** = **"manual testing."** The word "functional" is largely a **euphemism** — used so as not to insult the people who do it.
  - Ideally done by people with **domain expertise** (they know the business and can spot wrong behavior).
  - In reality often done by people with **neither domain nor testing background** — e.g. **freshers following a spreadsheet of mechanical steps blindly.** This is exactly why **test-content development quality matters enormously**: if a fresher is going to follow your steps without understanding them, your steps had better be complete and unambiguous.
- **Automation testing** = **you test with code.** You write programs that exercise the software.

## The maturity constraint (why manual testing is unavoidable early)

A hard truth that shapes everything:

> **You cannot automate against an immature application.** Automation needs a **minimum level of maturity** — at least an **MVP running in a test/QA environment** — before there's anything stable enough to write scripts against.

The consequence:

- **Early on, functional (manual) testing is unavoidable** — the app changes too fast and isn't stable enough to script.
- **By the time automation is ready, the first product release has usually already shipped.** Automation almost always arrives *after* the first release, not before it.

## The default stance

> **Argued default:** **automation-first.** Fall back to functional (manual) testing **only when automation isn't feasible** — most importantly, when the app isn't yet mature enough to script.

## The big debate: can functional testing be shifted to automation?

> **Open question.** Can enough functional testing be **shifted into automation** that human functional testers are freed to focus on **new features** (which are, by definition, the least mature and least automatable part)?
>
> The honest answer from the roundtable: **there is no perfectly-automated software-development regime.** Automation is an **improvement over manual testing — not the ideal, complete flow.** You are always somewhere on the spectrum, never at the "fully automated" end.

## The developer-portal / codegen thread

A big reason some companies automate so much more effectively than others: **internal developer portals.**

- **T-Mobile, Netflix, Amazon, Google** all have rich internal **developer portals** — **menu-driven scaffolding, codegen, and templates** (via tooling like **Ansible**). The effect: **"coding becomes fill-in-the-blank."** When new services are generated from templates, their tests and pipelines can be generated too — automation compounds.
- **Service companies usually lack these portals.** They typically use the **parent client's portal** rather than owning one. So the automation advantage accrues to product companies.

> This correlates with company type: **developer-portal maturity tracks whether you're a product company or a pure services company.** (See the cross-cutting notes in the [README](README.md) and [debates file](discussion-debates.md).)

## The economics of automation (a real constraint)

Automation is not free, and maintaining it can be brutally expensive:

- A **single template change can cascade to 30–100 microservices.** Now your "cheap" codegen is a migration project.
- **Microsoft can amortize migration-tool cost across worldwide users** — building a migration tool is worth it when millions benefit.
- **A single company often can't** justify that cost, and must therefore **co-exist with legacy** — keep old and new side by side rather than migrating everything.

> **The strategic point for a would-be tech lead:** deciding *how much* to automate is an **economics** decision, not just an engineering one. The right amount of automation for Amazon is the wrong amount for a 40-person shop, because Amazon amortizes the tooling cost across a scale you don't have.

## Putting it together

1. **New / immature code** → functional (manual) testing, because it can't be automated yet.
2. **As code matures (MVP in QA)** → shift stable, repeated checks into automation.
3. **Free the humans** to focus on the newest, least-automatable features — but accept you'll **never reach fully-automated**, and that maintaining automation is an ongoing cost you must justify against your own scale.

## Check your understanding

1. Why is "functional testing" called a euphemism? Who actually performs it, and why does that make test-content quality critical?
2. State the maturity constraint. Why does automation almost always arrive *after* the first release?
3. What is the argued default stance, and what is the one main reason to fall back to manual?
4. Summarize the "can functional testing be shifted to automation?" debate. Why is there "no perfectly-automated regime"?
5. What is a developer portal, and how does "coding becomes fill-in-the-blank" help automation compound?
6. Explain the economics: why can Microsoft justify a migration tool that a 40-person company can't? What is that company forced to do instead?
