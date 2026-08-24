# 12 — Coverage: What It Really Means

This was one of the richest arguments in the roundtable. Read it carefully — "coverage" is the single most misused number in testing.

## Code coverage vs logic / behavior coverage

Two different things wear the same word:

- **Code coverage** — *how many lines / branches were executed* by the tests.
- **Logic / behavior coverage** — *how many of the intended behaviors were actually verified.*

They are **not** the same, and confusing them is where teams fool themselves.

## The 100% coverage story (and why it's gameable)

The team in the discussion **enforced 100% code coverage** — and they could, because they worked in **TypeScript**, where **everything is mockable** (unlike Java, which has parts that simply can't be mocked). So 100% was technically attainable.

But here's the catch:

> **High code coverage is gameable.** You can **"touch" a line** — call the function so the coverage tool marks it executed — **without asserting that it did the right thing.** You get the coverage number **without real testing.**

Therefore:

> **100% code coverage ≠ correctness.** What determines *real* coverage is the **sufficiency of the test cases** — and **coverage tools do not guarantee that.** A coverage tool counts execution; it cannot count *meaning*.

```typescript
// This test achieves 100% line coverage of `discount`…
function discount(price: number, isMember: boolean): number {
  return isMember ? price * 0.9 : price;
}

test("covers the line", () => {
  discount(100, true);   // the function ran — coverage tool is satisfied…
  // …but there is NO assertion. It could return anything and this test still "passes".
});
```

The line was covered. Nothing was tested. That gap is the whole problem.

## Private / hidden functions: the coverage trap

A structural reason you may *not* hit 100% honestly:

- A **public interface `A`** calls **`B`**, which uses an **inner function `C`.**
- If **`C` has nested branches** (a `switch` with a `default`, say) whose **side effects aren't emitted through the interface**, you **can't simulate them from outside** — so you **won't hit those branches from the public surface.**

**The bad fix:**

> **Making private functions public just to test them.** **Don't do this.** You'd be corrupting your design to satisfy a metric — exposing internals that should stay hidden, purely to make a number go up.

**The better philosophy** (Sarath's, agreed and enhanced by Sreejith & Lal):

> **Test the public interface / contract, top-down.** If a hidden function's behaviors are **documented in the contract**, then the **top-level test cases must cover them** — you reach `C`'s branches by driving `A` with inputs that *should* exercise them. Test **intent / behavior at the interface**, and **program defensively from the top.**

And an honest limit:

> **Legacy / third-party static-library code that you merely call may be impossible to fully cover.** Acknowledge it rather than gaming around it.

## Why top-down (the refactoring argument)

The strongest argument for testing top-down isn't coverage at all — it's **change**:

> **If you don't unit-test top-down, refactoring becomes very hard.** When your tests are bolted onto internal functions, every refactor breaks the tests even when behavior is unchanged. When your tests assert the **public contract**, you can **freely rearrange the internals** and the tests still hold.

So: **focus first on public methods / outward contracts.** Once you have **full coverage of the contract, you needn't unit-test deeper** — the internals are covered *through* the contract, and they stay refactorable.

## The governance reality: coverage gates create politics

Here the discussion got refreshingly blunt.

> **Enforcing "no PR merge below 100% coverage" creates politics.** People write **throwaway tests to pass the gate** — the "touch the line, assert nothing" trick above. The tension is **feature-shipping vs. real quality**, and under deadline pressure, feature-shipping wins.

The honest observation:

> *"This looks like gaming a criterion to ship the feature — non-technical people do this."* A coverage gate, naively enforced, doesn't produce quality; it produces **clever ways to appear to have quality.**

## The healthier practice

What actually works, per the group:

1. **Identify and document all test cases at planning time** — at the **functional level, not the unit level** — **before coding starts.** Decide what "correct" means before you write it.
2. **Verify at review time** — it's the **reviewer's duty** to check that the documented cases are genuinely tested, not just that a number is green.
3. **Bring design up-front** too — don't discover the test cases after the fact.
4. **Enforce via gate-keeping.** *A gate is essential* — **without a gate, the tooling alone is useless** (the same point made about static analysis in [module 08](08-static-analysis-instrumentation.md)). But the gate must check **documented cases and review sign-off**, not just a coverage percentage.

> **The synthesis:** coverage *tools* are necessary but not sufficient. A **percentage** can be gamed; **documented cases + human review + a real gate** is what turns coverage from a vanity number into a quality signal. Pair this with **mutation testing** ([module 11](11-release-post-release-testing.md)) to check whether the tests would actually *notice* a bug.

## Check your understanding

1. Distinguish code coverage from logic/behavior coverage. Which one does a coverage tool measure?
2. Show, in words or code, how you can get 100% line coverage while testing nothing. Why is that possible?
3. Why can a public-only test suite fail to reach a hidden function's branches? What's the *bad* fix, and why is it bad?
4. State the top-down testing philosophy. How does documenting a hidden function's behavior in the contract help?
5. What is the refactoring argument for testing top-down?
6. Explain how a naive 100%-coverage merge gate creates politics and throwaway tests.
7. Describe the healthier practice in four steps. Why is a gate still essential, and what should it actually check?
