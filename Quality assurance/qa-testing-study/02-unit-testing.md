# 02 — Unit Testing

## What is a unit?

A unit test tests a particular, **smallest unit of code**. Simple enough — until you ask what a "unit" actually *is*. The roundtable did not fully agree, and the disagreement is worth seeing:

- **A class** — the most common answer in the OO/enterprise world.
- **A function** — the answer you hear from people who think in functions rather than objects.
- **A collection of related functions/classes** — sometimes called a **"category"** — because a single meaningful behavior may not live in exactly one class or one function.

> **Open question:** There is no universal right answer. Pick the smallest thing that has a *meaningful, testable behavior* in your codebase, and be consistent. The important discipline is naming your SUT precisely (see [module 01](01-shift-left-qa-landscape.md)), not winning the "class vs function" argument.

## Basic anatomy

- **One class can have multiple tests.** You don't write one test per class; you write one test per behavior/scenario.
- **A single test can cover positive and negative cases** — e.g. a valid input returning the right value *and* an invalid input throwing the expected exception.

### Mocking vs hard-coding dependencies

A unit test isolates the unit. Its dependencies must be controlled so that the test result depends only on the unit under test:

- **Mocking** — replace a dependency with a fake that returns canned answers (a mock database, a mock network call). This keeps the test deterministic and fast.
- **Hard-coding dependencies** — supply fixed, known dependency values directly.

Either way, the goal is the same: the unit is checked against *known* inputs and *known* expected outputs, with nothing unpredictable in the way.

## Why bother? (The benefits)

- **Catch known edge cases early**, before they reach anyone else.
- **Detect breaking changes automatically.** This matters most when a *junior developer* modifies code — the tests break loudly instead of the bug shipping silently.
- **Fits the CI pipeline.** Add the tests to continuous integration and regressions break the build automatically.
- **Validate code before integration.** You can prove a piece of code correct *before* it's wired into the rest of the system.

## The philosophical objection (and the answer)

A common pushback, quoted honestly in the roundtable:

> "The code will change anyway. Why write tests? Why not just test it manually?"

The answer is precisely *because* the code will change. The system **evolves** over time. Unit tests exist so that when you (or a teammate) modify something, you find out immediately whether your change created a **side effect** on existing functionality. Manual testing does not scale across the hundreds of times you'll come back around the loop; a saved test does.

> **Honest practitioner note:** Several engineers in this roundtable admitted their real products had **no TDD, and sometimes no unit tests at all.** This guide teaches the ideal *and* reports the reality. Knowing the ideal is what lets you argue for it.

## The Open/Closed Principle connection

The **Open/Closed Principle** (open for extension, closed for modification) applies directly to how you write and maintain unit tests:

- Write tests so that you **rarely have to modify *previous* tests.** Old tests are **closed for modification**; the suite is **open for extension** — you *add* new tests rather than rewriting old ones.
- If the class's logic changes, prefer **adding methods** over rewriting existing ones, so existing tests keep protecting existing behavior.
- **Developer discretion matters.** Sometimes you *must* change a test. When you do, change it in a way that **regression tests still catch problems** — don't neuter the safety net to make the build green.

The payoff: a test suite that grows monotonically becomes a reliable regression asset. A suite you constantly rewrite protects nothing.

---

## xUnit / test-framework mechanics

The following generalizes across **JUnit** (Java), **TestNG** (Java), **NUnit** (.NET), **xUnit** (.NET), and **XCTest** (Apple/Swift). The names differ; the machinery is the same.

### The test orchestration engine

At the heart of every framework is a **test orchestration engine**. It:

1. **loads assemblies** (compiled code),
2. **finds the tests** — via **reflection**, by looking for the markers you put on test methods,
3. **executes** them, and
4. **reports** Red (fail) / Green (pass).

The markers are called **annotations** in Java, **attributes** in .NET, and **macros** in Swift. They tell the engine "this method is a test," "this method is setup," "this method is teardown."

### Setup / Teardown (a.k.a. initialize / cleanup)

Before each test the engine runs **setup** (initialize); after each test it runs **teardown** (cleanup). Their job is to (re)initialize the SUT and its mocked/hard-coded dependencies into a **known clean state**, so that tests don't contaminate each other.

- **Per-test setup** runs once *per test* — the strictest isolation.
- **Per-group setup** runs once for a whole group/class — cheaper, used when re-initialization is expensive and safe to share.

> Choosing per-test vs per-group is a real trade-off: isolation (per-test) vs speed (per-group). Default to per-test isolation; relax to per-group only when you understand why it's safe.

### xUnit specifics: Fact vs Theory

xUnit (the .NET framework) has **two kinds of test**, and the distinction is genuinely illuminating:

- **`[Fact]`** — a **parameterless** assertion. The SUT is checked against **hard-coded** values. There are no inputs to vary.
- **`[Theory]`** — a **parameterized** test. You supply data (inline, or from a data-provider class) and the framework **iterates the one test over many data points**.

There's a clean logic analogy:

> A **Fact** ≈ a **proposition** (a single statement that is true or false).
> A **Theory** ≈ a **predicate** (a statement with variables, true or false *depending on the input*).

```csharp
using Xunit;

public class CalculatorTests
{
    // A Fact: no parameters, checked against hard-coded values.
    [Fact]
    public void Add_TwoAndTwo_ReturnsFour()
    {
        var calc = new Calculator();
        Assert.Equal(4, calc.Add(2, 2));
    }

    // A Theory: one test, many data points (inline data).
    [Theory]
    [InlineData(1, 1, 2)]
    [InlineData(2, 3, 5)]
    [InlineData(-1, 1, 0)]
    public void Add_VariousInputs_ReturnsSum(int a, int b, int expected)
    {
        var calc = new Calculator();
        Assert.Equal(expected, calc.Add(a, b));
    }
}
```

### Data injection

For a Theory you inject data two ways:

1. **Inline data** — attributes on the method itself (`[InlineData(...)]` above).
2. **A data-provider class** — you *write a class* the framework pulls data from. This is how you feed data from a file, a spreadsheet, a computed sequence, etc.

```csharp
public class AddData : TheoryData<int, int, int>
{
    public AddData()
    {
        Add(1, 1, 2);
        Add(10, 20, 30);
    }
}

public class CalculatorTheoryTests
{
    [Theory]
    [ClassData(typeof(AddData))]      // data-provider class
    public void Add_FromProvider(int a, int b, int expected)
        => Assert.Equal(expected, new Calculator().Add(a, b));
}
```

The equivalent in JUnit 5 is `@ParameterizedTest` with `@ValueSource` / `@CsvSource` / `@MethodSource`:

```java
class CalculatorTests {

    @Test                                   // JUnit's Fact-equivalent
    void addTwoAndTwo() {
        assertEquals(4, new Calculator().add(2, 2));
    }

    @ParameterizedTest                      // JUnit's Theory-equivalent
    @CsvSource({"1,1,2", "2,3,5", "-1,1,0"})
    void addVarious(int a, int b, int expected) {
        assertEquals(expected, new Calculator().add(a, b));
    }
}
```

### Historical asides (worth knowing, not memorizing)

- **Microsoft Fakes** — Microsoft's paid/licensed mock-and-stub tool. Now largely defunct/scrapped.
- **NUnit** — the old .NET standard.
- **xUnit** — now the common open-source choice for .NET.
- **XCTest** (Apple) — the very framework Apple's own dev team uses. The newer **Swift Testing** runs *on top of* XCTest.
- **Reflection style differs:** Swift uses **dynamic reflection via `Mirror`** (because of key-value compliance / KVC), rather than the static, attribute-driven reflection of the .NET/Java world. Same idea — "find and describe the tests at runtime" — different mechanism.

## Check your understanding

1. Give three different answers to "what is a unit?" and say why there's no single right one.
2. What is the difference between mocking a dependency and hard-coding it? What do both achieve?
3. Explain the Open/Closed Principle as it applies to a growing test suite. Why is rewriting old tests a warning sign?
4. Someone says "the code will change anyway, so unit tests are a waste." Answer them in two sentences.
5. What are the four jobs of a test orchestration engine?
6. Explain the Fact-vs-Theory distinction using the proposition-vs-predicate analogy. Convert one Fact into a Theory.
7. What are the two ways to inject data into a parameterized test?
8. Why might you choose per-group setup over per-test setup — and what do you give up?
