# 03 — Test-Driven Development (TDD)

## TDD is a philosophy, not a tool

The single most important thing to understand: **TDD is a development *philosophy*, not a testing tool.** You can do TDD with xUnit, JUnit, XCTest, or anything else — the framework is incidental. What makes it TDD is the *order* in which you work: **the test comes first.**

## Red — Green — Refactor

The model is a three-step cycle:

1. **Red** — write the test case *first*. Run it. It **fails** (there's no code yet, or not enough).
2. **Green** — write the **minimal** code needed to make the test pass. Nothing speculative — just enough.
3. **Refactor** — clean up the code you just wrote, with the passing test guarding you.

Then repeat for the next behavior.

Some practitioners call this **RGB**, where **Blue = Refactor** — a nod to Joshua Kerievsky's *Refactoring to Patterns* (refactoring toward known design patterns as the "blue" step).

```csharp
// 1. RED — write the test first; Add() doesn't exist yet, so this fails to compile/pass.
[Fact]
public void Add_TwoAndThree_ReturnsFive()
    => Assert.Equal(5, new Calculator().Add(2, 3));

// 2. GREEN — minimal code to pass. No overflow handling, no extra features.
public class Calculator
{
    public int Add(int a, int b) => a + b;
}

// 3. BLUE/REFACTOR — improve structure now that the test protects you.
//    (Here there's nothing to refactor yet — and that's fine.)
```

Each passing test then becomes an **asset for regression testing**: it will keep proving, forever, that this behavior still works as the system evolves.

## Why TDD is "non-intrusive"

TDD is presented as a **non-intrusive** approach — lighter weight than a full BDD framework (see [module 04](04-behavior-driven-development.md)). You don't need a special language, a spec parser, or step-definition classes. You write ordinary tests in your ordinary test framework; the only discipline is the *order*. That lightness is a genuine advantage when you're trying to shift-left without adding process overhead.

## An honest look: TDD is a reasoning tool, and it doesn't click for everyone

Here the roundtable was refreshingly candid.

TDD is fundamentally a **reverse-mapping / self-reasoning tool.** Writing the test first forces you to state, precisely, what "correct" means *before* you write the code — you reason backwards from the desired outcome to the implementation.

But:

> **Not everyone reasons logically by default.** For some developers the "write the failing test, then satisfy it" mental model clicks immediately; for others it never quite does. TDD is a **habit / discipline** — you have to build the *procedural knowledge* for the mental model to become natural. An enterprise that adopts TDD as a shared habit benefits from it. But **TDD does not work out everywhere**, and pretending otherwise is dishonest.

And the most honest note of all:

> **Several practitioners admitted their actual products had no TDD, and sometimes no unit tests at all.** This is not an endorsement of skipping tests — it's a reminder that the gap between best practice and real practice is wide, and that adopting TDD is a *cultural* project as much as a technical one.

## Where TDD sits relative to BDD

- **TDD** — the test author knows the internals. It's a developer's tool for driving *their own* code. Lightweight, non-intrusive. See this module.
- **BDD** — the test author describes *intent* against a black box using a fixed vocabulary; good for letting non-developers specify behavior, and heavier as a result. See [module 04](04-behavior-driven-development.md).

> **Open/contested:** Whether BDD is "too heavy" for shift-left, or whether it "doesn't bother the developer much," is genuinely debated. That debate lives in module 04 and in the [debates file](discussion-debates.md). TDD's lightness is the baseline against which that argument is had.

## Check your understanding

1. Why is it accurate to call TDD a *philosophy* rather than a *tool*?
2. Walk through Red–Green–Refactor for a tiny feature of your choice. What does "minimal code" mean in the Green step, and why does it matter?
3. What does the "Blue" in RGB refer to, and where does the name come from?
4. Explain the claim that "TDD is a reverse-mapping / self-reasoning tool."
5. Why does the guide say TDD "doesn't work out everywhere"? Is that an argument against ever using it?
6. In one sentence each, contrast who writes the test in TDD vs BDD, and what they know about the code.
