# Glossary

Concise definitions of every term used in this module. Where a term is contested, that's noted — see the [debates file](discussion-debates.md) for the full argument.

## Strategy & lifecycle

- **Shift-left development** — Emphasizing non-functional requirements (quality, testing, security) *early* in the development lifecycle.
- **Shift-right development** — The accidental default where quality concerns are handled *late*; a side effect of feature-driven work.
- **Functional requirements** — What the software does (features).
- **Non-functional requirements** — How well it does it: quality, performance, security, reliability, maintainability, usability.
- **MVP (Minimum Viable Product)** — The smallest releasable product; the starting point of feature-driven development.
- **Automation mindset** — The habit of automating any repeated manual process once you've mastered it, to free your attention for the next problem.
- **Test Management** — The integrated strategy for how testing happens; heavily shaped by organizational politics (where power sits).
- **Test Content Development** — Deciding *what* to test; for automation, also writing the code that runs the tests.
- **Test Execution Strategy** — Running tests and reporting results.
- **SUT / AUT / DUT** — System / Application / Device Under Test. The precisely-named thing every test case and suite is designed around.

## Development approaches

- **TDD (Test-Driven Development)** — A development *philosophy*: write the test first, then the code. Non-intrusive/lightweight.
- **Red–Green–Refactor** — The TDD cycle: failing test (Red) → minimal passing code (Green) → clean up (Refactor).
- **RGB** — Alternate name for Red–Green–Refactor, where **Blue = Refactor** (from *Refactoring to Patterns*).
- **BDD (Behavior-Driven Development)** — Driving development from plain-English behavior specs written against a black box; heavier than TDD.
- **Gherkin** — The English-like language for BDD specs (used by Cucumber).
- **Given / When / Then** — BDD spec structure: assumptions / action / expected consequent.
- **Step definitions** — Annotated methods (`@Given`, `@When`, `@Then`) that make Gherkin executable.
- **Open/Closed Principle** — Open for extension, closed for modification; applied to tests: add new tests rather than rewriting old ones.

## Test frameworks & mechanics

- **Test orchestration engine** — Loads assemblies, finds tests via reflection, executes them, reports Red/Green.
- **Reflection** — Runtime inspection used to discover tests. Static (attributes, .NET/Java) vs dynamic (`Mirror` in Swift, due to KVC).
- **Annotation / Attribute / Macro** — Java / .NET / Swift markers for tests, setup, teardown.
- **Setup / Teardown (Initialize / Cleanup)** — Code run before/after tests to (re)establish a known clean state. Per-test (isolation) vs per-group (speed).
- **Fact (xUnit)** — A parameterless test checking the SUT against hard-coded values. Analogy: a **proposition**.
- **Theory (xUnit)** — A parameterized test iterating over many data points. Analogy: a **predicate**.
- **Inline data / Data-provider class** — Two ways to feed a Theory: attributes on the method, or a class the framework pulls data from.
- **Mocking** — Replacing a dependency with a controlled fake for deterministic tests.
- **Hard-coding dependencies** — Supplying fixed known dependency values directly.
- **JUnit / TestNG / NUnit / xUnit / XCTest** — Test frameworks for Java / Java / .NET / .NET / Apple-Swift.
- **Swift Testing** — Newer Apple framework running on top of XCTest.
- **Microsoft Fakes** — Microsoft's old paid mock-and-stub tool; largely defunct.

## API testing

- **REST / SOAP / gRPC** — API styles. gRPC uses **Google Protocol Buffers** (binary, contract-first).
- **SoapUI / Postman / Swagger UI** — API testing/exploration tools.
- **Web API testing** — Hitting the endpoint directly.
- **SDK testing** — Testing a generated client library (`.dll` / `.jar`).
- **Hybrid (API) testing** — A call-level interface that hits REST underneath (e.g. Informatica MDM Hub's generated `.jar`).
- **API (Application Programming Interface)** — The interface the *consumer* calls.
- **SPI (Service Provider Interface)** — The interface the *provider* implements/plugs into.
- **API–SPI divide** — The gap between what consumers need and what providers implement; real in OEM/product companies, where one API call maps to many implementations.
- **Mediator (pattern)** — An intermediary (e.g. between controller and service) that keeps code modular and testable.
- **ASP.NET MVC** — Framework whose URI→controller indirection made test mapping possible (unlike TDD-unfriendly Web Forms).
- **OLE DB Consumer / Provider** — Historical API/SPI example; the consumer side later became ADO.NET.

## Integration & UI

- **Integration testing** — Integrating components/modules/services with real data flow, or mocking one side deterministically. *Contested definition — see debates.*
- **End-to-end (E2E) integration** — Integrating two or more independent *systems*, across different form factors; typically non-deterministic.
- **Deterministic integration test** — One made repeatable by mocking the network/DB boundary.
- **Selenium WebDriver** — Web UI automation via a browser agent injected into the DOM; abstracts the driver so one script targets web/Android/iOS.
- **Ranorex** — Desktop UI automation using image processing to locate elements.
- **XCUITest** — Apple's native iOS UI automation framework.
- **Introspection / accessibility hierarchy** — A queryable tree of UI elements (VoiceOver-style) that makes elements findable without image processing.
- **Developer-aware automation** — The developer provides IDs/accessibility labels so tools find elements reliably.
- **Hostile-automatable** — Automatable from outside without app cooperation (e.g. Windows/X-Windows native introspection).
- **Device farm** — Cloud fleet of real devices for mobile UI testing.
- **Xamarin** — `.iOS`/`.Android` bindings; `Xamarin.Forms` sits on top, behaves WPF-like. Testing left open.

## Static analysis & instrumentation

- **Static analysis** — Inspecting code without running it.
- **SonarQube** — Widely-used static analysis platform ("an agent with other tools inside"), for quality and security.
- **Linter** — Per-language static style/quality checker.
- **FxCop / StyleCop** — Old Microsoft .NET analyzers, later folded into VS Code Analysis, now largely unused.
- **Checkmarx / Veracode** — Security static analysis tools; Veracode offered as a service.
- **Static / trace-based / hybrid** — The three approaches to security analysis.
- **Instrumentation / dynamic testing** — Observing a program while it runs (profilers, memory-leak detectors like BoundsChecker; Debug builds; Reflector-style tools).
- **Git hook** — A script triggered by check-in (e.g. pre-push) that runs the testing infrastructure.
- **Gate / gate-keeping** — A rule that blocks merge on failure; without it, tooling alone is useless.

## Execution strategy

- **Functional testing** — A euphemism for **manual testing**; ideally done by domain experts, often by freshers following a spreadsheet.
- **Automation testing** — Testing with code. Requires a minimum app maturity (at least an MVP in QA).
- **Developer portal** — Internal menu-driven scaffolding/codegen/templates (T-Mobile, Netflix, Amazon, Google) making "coding fill-in-the-blank."

## Acceptance & release

- **UAT (User Acceptance Testing)** — A stakeholder *sign-off* strategy, not system validation; not done by the dev team.
- **Definition of Done** — The agreed checklist for "complete"; in SaaS, a PO demo can be part of it.
- **Crowd-testing** — Random people testing a product as a black box.
- **Smoke testing** — A prima-facie/sanity subset run on release or after a change; often git-hook gated.
- **A/B testing** — Comparing **treated** vs **control** variants via staged traffic (RCT language); not inherently about new features.
- **Weblab / Firebase Remote Config / Optimizely** — A/B or A/B-capable tools (Weblab is Amazon's home-grown).
- **Regression testing** — "Smoke test plus-plus"; re-run functional/automation + smoke before release.

## Performance family

- **Average Load Test** — Do we meet SLA under normal (shaped) load, always?
- **Stress Test** — Behavior beyond normal load. (Distinct from **Chaos testing / Chaos Monkey** = resilience to failure.)
- **Spike Test** — Handling sudden bursts; write scaling policy up-front.
- **Breakpoint Test** — Increase load until it breaks, to find capacity; doubles as capacity planning.
- **Soak Test** — Long-duration run to find leaks / gradual degradation.
- **SLA (Service Level Agreement)** — The performance/availability target the system must meet.
- **Scaling out / in** — Adding machines (easy) / removing them by draining and freeing (hard).
- **SRE (Site Reliability Engineering)** — Scripts + infra config that scale non-disruptively, configured at the load balancer.
- **Goroutine** — Go's lightweight virtual thread; cause of Go's characteristic CPU spikes under load.

## Coverage & test quality

- **Code coverage** — How many lines/branches were executed. Gameable; ≠ correctness.
- **Logic / behavior coverage** — How many intended behaviors were actually verified.
- **Top-down testing** — Testing the public interface/contract; internals covered through it, keeping code refactorable.
- **Mutation testing** — Introducing bugs (mutants) to verify the tests catch them; checks the tests themselves.
- **Stryker (`Stryker.NET`, StrykerJS)** — Mutation testing tools for C# / JS-TS.
