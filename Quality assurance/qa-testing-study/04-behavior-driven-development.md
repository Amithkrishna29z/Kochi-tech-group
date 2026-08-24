# 04 — Behavior-Driven Development (BDD)

## The idea

Where TDD drives development from tests written by someone who knows the internals, **BDD drives development from a description of *behavior*** written in near-plain English against a **black box**. The person writing the spec says *what should happen*, not *how it's implemented*.

## Tools and languages

- **SpecFlow** — BDD for .NET.
- **Cucumber** — the Java-world standard; uses the **Gherkin** language.
- **Cypress** — a JS/TS end-to-end tool, usable with a cucumber-style BDD plugin.
- Various other JS/TS-based tools.

The shared language across much of this is **Gherkin** — a small, structured, English-like syntax.

## Structure: Given — When — Then

A BDD specification is written as **Given / When / Then**:

- **Given** — a set of **assumptions** / antecedent setup. The starting state.
- **When** — the **action**. The thing that happens.
- **Then** — the **expected consequent**. What must be true afterward.

```gherkin
Feature: Shopping cart totals

  Scenario: Adding two items sums their prices
    Given the cart is empty
    And a "Pen" costs 30
    And a "Notebook" costs 50
    When I add a "Pen" and a "Notebook" to the cart
    Then the cart total should be 80
```

The plain-English Gherkin is backed by **annotated step-definition classes** — methods marked `@Given`, `@When`, `@Then` (Cucumber/Java shown) that the **orchestration engine parses and executes**:

```java
public class CartSteps {

    private Cart cart;
    private final Map<String, Integer> prices = new HashMap<>();

    @Given("the cart is empty")
    public void theCartIsEmpty() { cart = new Cart(); }

    @Given("a {string} costs {int}")
    public void aProductCosts(String name, int price) { prices.put(name, price); }

    @When("I add a {string} and a {string} to the cart")
    public void iAddTwoItems(String a, String b) {
        cart.add(prices.get(a));
        cart.add(prices.get(b));
    }

    @Then("the cart total should be {int}")
    public void theTotalShouldBe(int expected) {
        assertEquals(expected, cart.total());
    }
}
```

### Example / data tables

You can attach **example tables** to a scenario so it runs over many rows — even sourced from **spreadsheets / CSV columns**:

```gherkin
  Scenario Outline: Totals for various carts
    Given the cart is empty
    When I add items worth <a> and <b>
    Then the cart total should be <total>

    Examples:
      | a  | b  | total |
      | 30 | 50 | 80    |
      | 10 | 10 | 20    |
      |  0 | 99 | 99    |
```

## The key property: the framework doesn't care what layer you test

This is the most important conceptual point about BDD:

> **The framework doesn't care** whether you use it for unit, module, API, integration, security, or functional testing. **You** decide — by what the *step classes call.* The Gherkin is just text; the step definitions can call a function, hit an API, drive a database, or **trigger Selenium** for a full end-to-end browser run. The same `Given/When/Then` vocabulary drives any layer.

That flexibility is why BDD shows up everywhere from unit-level specs to full E2E suites.

## BDD as living documentation

Because the spec is written in readable English and is *executable*, the `Given/When/Then` files double as **living documentation**: they describe what the system is supposed to do, and they can't silently rot, because if the behavior changes and the spec doesn't, the test fails.

## The bias-removal angle

A subtle but powerful benefit:

- In **TDD**, the test author **knows the internals** — and can therefore write *any* test, including ones biased by how they happened to implement things.
- In **BDD**, the author describes **intent** against a **black box**, using a **fixed set of allowed phrases** (the defined steps). They **can't write arbitrary tests** — only ones expressible in the agreed vocabulary.

This makes BDD good for having **non-developers / QA specialists specify behavior**: they say what the system should do, in business language, without needing (or being able) to reach into the implementation.

## Real example: BDD for security testing

From the transcript: one engineer used **BDD for security testing.** They took an **open-source BDD framework and modified it into a product definition** — the `Given/When/Then` steps described security scenarios, and **inside the steps it triggered Selenium** to actually drive the application. This is a concrete instance of the "framework doesn't care what layer" point: the same BDD machinery, pointed at security.

## The debate: is BDD "heavy" or not?

> **Open / contested question.** Two honest views were expressed:
>
> - **"BDD is heavy" (against shift-left):** the extra language, the step-definition classes, the parsing engine — it's ceremony that slows the developer down compared to plain TDD. See [module 03](03-test-driven-development.md), which frames TDD as the lightweight, *non-intrusive* baseline.
> - **"BDD doesn't bother the developer much":** once the step library exists, writing new scenarios is cheap, and the documentation + bias-removal benefits outweigh the setup cost.
>
> There is no settled answer. Which side is right depends on your team, your need for non-developer-authored specs, and how much you value living documentation. This debate is revisited in the [debates file](discussion-debates.md).

## Check your understanding

1. What does each of Given, When, and Then represent?
2. How does plain-English Gherkin actually get executed? What role do the annotated step classes play?
3. Explain "the framework doesn't care what layer you test." Who decides the layer, and how?
4. Why is BDD good for letting a non-developer specify behavior? Relate this to the "fixed set of allowed phrases."
5. In what sense is a BDD spec "living documentation"?
6. Summarize both sides of the "is BDD too heavy?" debate. Which side would you take for a small team with no dedicated QA, and why?
