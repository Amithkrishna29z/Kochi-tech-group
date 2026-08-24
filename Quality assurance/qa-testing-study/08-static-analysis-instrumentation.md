# 08 — Static Analysis & Code Instrumentation

This module covers testing that inspects the *code itself* — either **without running it** (static analysis) or **while running it** (instrumentation / dynamic testing).

## Static analysis for quality

Static analysis examines source (or compiled) code for problems without executing it. For **quality** (not security):

- **SonarQube** — the widely-used workhorse. It acts like an **agent with other tools inside it** — a coordinating platform that runs many analyzers and aggregates results.
- **Linters** — per-language style/quality checkers (ESLint, Pylint, etc.).
- **FxCop** and **StyleCop** — Microsoft's older .NET analyzers (in common use around **2008**). Later **folded into Visual Studio's built-in "Code Analysis"** (around **2018**), and **now largely unused** as standalone tools.

> **Aside (contested / cultural):** the **mono-repo debate** and Microsoft's tooling history come up here. Big companies with mono-repos can standardize analysis across everything; smaller shops with many repos struggle to. There's no settled "right" answer — it's an organizational choice tangled up with tooling maturity.

## Static analysis for security

Security static analysis is a related but distinct discipline, with its own tools:

- **Checkmarx** — static application security testing (SAST).
- **Veracode** — offered **as a service** (you send code/binaries, they analyze).
- **SonarQube** also plays a **big role** on the security side, not just quality.

Three broad **approaches** to security analysis:

| Approach | How it works |
|----------|--------------|
| **Static** | Analyze code/binary without running it |
| **Trace-based** | Follow data/execution traces to find vulnerable flows |
| **Hybrid** | Combine static + dynamic/trace signals |

> **Established but nuanced:** security testing **"is not a no-brainer anymore."** It depends **heavily on your chosen tool-chain** — the tools you pick largely determine what you can and can't catch. There's no single button that makes software "secure."

## Git hooks & the pipeline

Static analysis earns its keep when it's **automated into the pipeline**:

- A **check-in (commit/PR)** triggers the **testing infrastructure** — including static analysis — via **git hooks.**
- The pipeline reports **test coverage**, supports **root-cause analysis**, and assigns **severity / priority** to findings.
- Without a **gate** (a rule that blocks merge on failure), the tooling alone accomplishes little — a theme that returns forcefully in [module 12](12-coverage.md).

```bash
# .git/hooks/pre-push (illustrative)
#!/bin/sh
echo "Running static analysis before push…"
sonar-scanner || { echo "SonarQube quality gate failed — push blocked."; exit 1; }
```

## Code instrumentation / dynamic testing

**Instrumentation** means observing the program *while it runs*. Examples:

- **Profilers** — measure where time and memory go.
- **Memory-leak detectors** — older tools like **BoundsChecker.**
- **Running in Debug** is itself **a form of instrumentation** — the debug build adds hooks, checks, and symbols the release build omits.
- **Reflector-style tools** — decompile/inspect assemblies to see what's really there.

> **Honest note:** most of this is **older tooling**, and the roundtable admitted **low current awareness** of it. It's included for completeness and because the *concepts* (profiling, leak detection, debug-as-instrumentation) are evergreen even when the specific tools have aged out.

## Where this fits in the loop

Static analysis and instrumentation sit **after** the unit/API/integration/UI layers and **before** production in the [main cycle](01-shift-left-qa-landscape.md#7-the-cycle-its-a-loop). One camp in [module 06](06-integration-testing.md) specifically argued that **static analysis at the pure-code level is *less* useful than testing at integration points** — a claim worth holding in mind: static analysis is valuable, but it is not a substitute for exercising the code where modules meet.

## Check your understanding

1. What distinguishes static analysis from instrumentation/dynamic testing?
2. What is SonarQube, and why is it described as "an agent with other tools inside"?
3. Name two security-focused static analysis tools and the three approaches (static, trace-based, hybrid).
4. Why does the guide say security testing "is not a no-brainer anymore"?
5. How do git hooks connect static analysis to the pipeline, and why is a *gate* essential?
6. In what sense is "running in Debug" a form of instrumentation?
7. Recall the module-06 claim about static analysis vs integration points. Do you agree? Why?
