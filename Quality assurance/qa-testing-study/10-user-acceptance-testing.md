# 10 — User Acceptance Testing (UAT)

## What UAT actually is

The most common misunderstanding: people think UAT is the *final round of system validation.* It isn't.

> **UAT is a sign-off strategy, not primarily system validation.** The **stakeholder / sponsor / beneficiary** confirms that they are **happy** with what was built. It's about *acceptance* — someone with authority saying "yes, this is what we wanted" — not about finding bugs (those should already be caught by the layers before it).

Two consequences:

- UAT **should not be done by the development team.** The people who built it can't independently accept it.
- **Ideally not only by domain experts either** — you want the people who will *live with* the software, not just those who understand the domain in the abstract.

## Who performs UAT

| Who | Notes |
|-----|-------|
| The **customer's own testing team** | They represent the beneficiary directly |
| **Business analysts on the customer side** | Know the requirements, sit with the customer |
| An **outsourced vendor's representative** | Often **preferable** — a vendor rep signing off gives a clean, accountable acceptance |
| A **third party** | Independent acceptance |

> **Noted preference:** a **vendor representative signing off is often preferable** — it's a clear, accountable handshake between the party that built (or brokered) the software and the party accepting it.

## The modern SaaS / agile reframing

Many agile/SaaS teams argue they **don't need formal UAT** at all:

- The **Product Owner** gets a **demo** each release — and that demo is **part of the Definition of Done.**
- Everything else is **automated**; after PR checks pass, the build is **practically production-ready.**
- So "acceptance" collapses into "the PO watched the demo and the pipeline is green."

> This is a legitimate reframing, not a corner cut — *provided* the automated layers underneath (unit/integration/manual) are genuinely doing their job. The demo is the sign-off; the pipeline is the validation.

## The B2C nuance (you can't just demo)

For **B2C** products (e.g. **Amazon.com**), the demo model breaks down — there is no single stakeholder to demo to; there are millions of anonymous users. Here **UAT shifts meaning:**

- UAT increasingly means **checking UX** — not functional correctness.
- **Functional correctness is *assumed* already ensured** by the unit / integration / manual layers below.
- UAT strictly checks **how the change *affects users*** — that **UX hasn't degraded** and that the **design matches intent.**

> **For B2C, UX can mean "business and death."** A functionally-correct change that quietly hurts the user experience can cost real money at scale. So B2C UAT polices the experience, having delegated correctness to the earlier layers.

## The multi-vendor org reality

In large enterprises the work is often split across **different vendors**, which shapes who accepts what:

- **Development** by one vendor (e.g. Wipro),
- **DevOps** by another (e.g. Cognizant / TCS),
- **QA** by another (e.g. UST).

And there are dedicated **crowd-testing** companies: they field **random people who test the product as a black box** — closer to real, unbiased user behavior than any internal team can simulate.

> This fragmentation is another instance of the guide's theme: *who signs off* and *who tests* is as much an organizational/contractual question as a technical one.

## Where UAT sits in the loop

UAT comes **late** — after the automated and manual layers, near release (see the [cycle](01-shift-left-qa-landscape.md#7-the-cycle-its-a-loop)). It is the human acceptance gate on top of everything the machines and manual testers already verified. In a healthy pipeline it should be *boring* — a confirmation, not a discovery.

## Check your understanding

1. Why is UAT described as a "sign-off strategy" rather than "system validation"? What does that imply about *who* should do it?
2. List the candidate performers of UAT. Why is a vendor representative often preferred?
3. How do SaaS/agile teams reframe UAT? What must be true underneath for that reframing to be safe?
4. Why does the demo model break down for B2C? What does UAT check instead, and why is UX "business and death" there?
5. In a multi-vendor org, name three functions and three different vendors. What is crowd-testing and what does it add?
6. Why should UAT be "boring" in a healthy pipeline?
