# BDD & Cucumber — 100 In-Depth Interview Questions & Answers

> A comprehensive, theory + concept-focused interview preparation document built from the BDD Study Plan. Each answer is written to demonstrate depth, not just keyword recall — suitable for SDET / QA Automation Engineer / Senior QA interviews.

---

## Section 1 — BDD Philosophy & Foundations (Q1 – Q10)

### Q1. What is BDD and why did it emerge?
**Answer:** Behavior-Driven Development (BDD) is a collaborative software development practice introduced by **Dan North in 2006** as a response to recurring confusion around Test-Driven Development (TDD). Developers using TDD struggled with *what* to test, *where* to start, and *how* to name tests. North reframed tests as **specifications of behavior** written in business language, shifting the focus from "testing code" to "describing behavior". BDD evolved into a **second-generation, outside-in, pull-based, multiple-stakeholder, high-automation agile methodology** that promotes a shared understanding between business and engineering through concrete examples.

### Q2. How does BDD differ from TDD and ATDD?
**Answer:**
| Aspect | TDD | ATDD | BDD |
|---|---|---|---|
| Focus | Code design | Acceptance criteria | Shared business behavior |
| Audience | Developers | Devs + Testers | Devs + Testers + Business |
| Language | Code / unit assertions | Acceptance tests | Ubiquitous, natural language (Gherkin) |
| Granularity | Unit-level | Feature-level | Behavior-level (any granularity) |
| Output | Passing unit tests | Acceptance test pass | Living documentation + executable specs |

BDD is **ATDD done with collaboration + ubiquitous language**, and **TDD done at the behavior level instead of the code level**.

### Q3. Explain the Discovery → Formulation → Automation cycle.
**Answer:**
1. **Discovery** — Three Amigos (Dev, QA, BA/PO) explore a story using techniques like **Example Mapping** to uncover rules, examples, and questions.
2. **Formulation** — The agreed examples are converted to structured **Gherkin scenarios** (Given/When/Then) that serve as both specification and acceptance criteria.
3. **Automation** — Step definitions and glue code are written so the scenarios become **executable specifications** that run continuously in CI.

The cycle is repeated for each story; the output is **living documentation** that always reflects the current behavior of the system.

### Q4. What is the role of the Three Amigos?
**Answer:** The Three Amigos are typically the **Business Analyst / Product Owner**, **Developer**, and **Tester**. Each brings a viewpoint:
- BA — *what problem are we solving?*
- Dev — *how will we build it?*
- QA — *how could it break?*

The session deliberately surfaces hidden assumptions **before code is written**, dramatically reducing rework. It is not a sign-off meeting; it is a **discovery conversation** that produces shared examples.

### Q5. What is "Ubiquitous Language" and why is it central to BDD?
**Answer:** Borrowed from **Domain-Driven Design (Eric Evans)**, ubiquitous language is a single vocabulary shared by business stakeholders and engineers. In BDD, Gherkin scenarios must be written **using domain terms** (e.g., "Premium Customer", "Settle Trade") rather than technical terms ("user_id = 1", "click #btn"). This prevents the "translation gap" where requirements lose fidelity as they move from business → tickets → code → tests.

### Q6. What is Living Documentation?
**Answer:** Living documentation is documentation that is **always current because it is executable**. Each Gherkin feature file describes system behavior in business language; because it is also run by CI on every build, any drift between documentation and code immediately fails the build. This solves the perennial problem of stale Word/Confluence docs.

### Q7. What is Specification by Example (SBE)?
**Answer:** SBE, coined by **Gojko Adzic**, is the practice of describing requirements through **concrete, illustrative examples** rather than abstract prose. Instead of "the system shall apply discounts based on customer tier", you write:
> Given a Gold customer with a cart total of $200, when they checkout, then a 15% discount is applied.

BDD is the most well-known implementation of SBE.

### Q8. When should you NOT use BDD?
**Answer:**
- Pure CRUD systems with no business rules — Gherkin adds ceremony with no benefit.
- Throwaway scripts / PoCs.
- Pure infrastructure / DevOps automation.
- Performance, load, or chaos testing — wrong abstraction.
- Teams unwilling to collaborate — BDD without Three Amigos degenerates into "Gherkin-flavored Selenium", the worst of both worlds.

### Q9. What is "Deliberate Discovery"?
**Answer:** A concept by Dan North stating that the **biggest project risk is ignorance** — what you don't know that you don't know. Deliberate Discovery means **intentionally investing time** to surface unknowns (via Example Mapping, spikes, conversations) instead of jumping to implementation. BDD operationalizes this by forcing examples and questions to the surface before automation.

### Q10. What is "Outside-In" development in BDD?
**Answer:** You start from the **user/business-facing behavior** (the outside) and let that drive the design of internal components (the inside). A failing acceptance scenario triggers the need for a controller; the controller triggers the need for a service; the service triggers unit-level TDD. This contrasts with "inside-out", where you build technical layers first and hope they compose into something useful.

---

## Section 2 — Gherkin Language (Q11 – Q25)

### Q11. List and explain all major Gherkin keywords.
**Answer:**
- **Feature** — top-level grouping; describes a capability.
- **Rule** (Gherkin 6+) — groups scenarios under a single business rule.
- **Background** — common Given steps shared by all scenarios in a feature.
- **Scenario / Example** — a concrete instance of behavior.
- **Scenario Outline / Scenario Template** — parameterized scenario.
- **Examples / Scenarios** — data table feeding a Scenario Outline.
- **Given / When / Then / And / But** — the steps of a scenario.
- **`"""` or ` ``` `** — DocStrings (multi-line input).
- **`|`** — DataTables.
- **`@tag`** — tags for filtering.
- **`#`** — comments.

### Q12. Difference between Scenario and Scenario Outline.
**Answer:** A **Scenario** describes one concrete example. A **Scenario Outline** is a *template* parameterized with `<placeholders>` that is expanded into multiple concrete scenarios via an `Examples:` table. Use Outline when the **same behavior** is verified with **different data**; use multiple Scenarios when each example tests a **different behavior**.

### Q13. What is Background and how is it different from a Before hook?
**Answer:**
- **Background** — Gherkin-level, **business-readable** preconditions, visible in the feature file and in living documentation. Runs before every scenario in that feature.
- **`@Before` hook** — Code-level **technical** setup (start browser, open DB connection), invisible to business readers.

Rule of thumb: if the precondition is *interesting to the business*, put it in Background; if it is *plumbing*, put it in a hook.

### Q14. What is the `Rule` keyword?
**Answer:** Introduced in Gherkin 6, `Rule` groups one or more scenarios that validate **a single business rule**, making feature files more navigable and aligning structure with Example Mapping (blue cards = Rules, green cards = Examples). Each Rule can have its own Background.

### Q15. Explain Gherkin tags and tag expressions.
**Answer:** Tags (`@smoke`, `@regression`, `@wip`) are metadata placed above Features, Rules, Scenarios, Outlines, or individual Examples. They support **boolean expressions** for filtering:
- `@smoke and @regression`
- `@smoke or @api`
- `not @wip`
- `(@smoke or @sanity) and not @manual`

Tags **inherit downward**: a Feature-level tag applies to all child scenarios.

### Q16. Imperative vs Declarative scenarios — give an example.
**Answer:**
**Imperative (bad):**
```gherkin
Given I open the browser
And I navigate to "/login"
And I type "admin" in the "username" field
And I type "pass123" in the "password" field
And I click the "Login" button
Then I see the dashboard
```
**Declarative (good):**
```gherkin
Given I am logged in as an administrator
Then I see the admin dashboard
```
Declarative scenarios describe **what** the user is doing in business terms; imperative ones leak UI implementation details, making the suite brittle and unreadable.

### Q17. What is a DataTable and what shapes can it take?
**Answer:** A DataTable is tabular data attached to a step. It can be:
- **Single-column list** — `List<String>`
- **Two-column key-value** — `Map<String, String>`
- **Multi-column table with header** — `List<Map<String, String>>`
- **Raw table** — `List<List<String>>`
- **Transposed** — flipped via `.transpose()`

In Cucumber-JVM, the conversion is automatic based on the parameter type, and can be customized via `@DataTableType`.

### Q18. What is a DocString?
**Answer:** A multi-line string delimited by `"""` or triple backticks, attached to a step. It is used for large payloads (JSON, XML, SQL) that would be unreadable inline. A **content type hint** (`"""json`) can be provided and used by `@DocStringType` transformers to auto-convert to POJOs.

### Q19. Gherkin internationalization — what is it?
**Answer:** Gherkin supports **70+ languages**. A header `# language: fr` enables French keywords (`Fonctionnalité`, `Scénario`, `Étant donné`, `Quand`, `Alors`). Useful for non-English teams writing in their domain language for true ubiquitous language.

### Q20. What does the "So That" clause add to a Feature description?
**Answer:** The classic Connextra format:
```
Feature: X
  As a <role>
  I want <capability>
  So that <business value>
```
The **So That** clause forces the author to articulate the **business value**. If you cannot complete it, the feature may not be worth building — a powerful prioritization filter.

### Q21. What are common Gherkin best practices?
**Answer:**
- One scenario = one behavior.
- Use business vocabulary, not UI selectors.
- Keep scenarios independent and idempotent.
- Avoid scenario ordering dependencies.
- Minimize data — only what is essential to the rule under test.
- Use Background sparingly; never for technical setup.
- Use Scenario Outline only when the variations test the same behavior.
- Lint feature files; enforce naming conventions.

### Q22. When should you choose multiple Scenarios over a Scenario Outline?
**Answer:** When each row would describe a **different behavior** (different Then), or when each row needs different Given/When steps. Outline is for *same behavior, different data*. If you find yourself adding `if/else` logic in step definitions based on Examples values, you should be writing separate scenarios.

### Q23. What is a "Pending" step?
**Answer:** A step definition that exists but throws `PendingException` (or similar). Cucumber reports it as *pending* (yellow), distinct from *failed* (red) and *undefined* (yellow with snippet). Useful for marking work-in-progress without breaking the suite.

### Q24. What are "ambiguous step definitions" and how do you resolve them?
**Answer:** When two step definitions match the same Gherkin step, Cucumber throws `AmbiguousStepDefinitionsException`. Causes: overlapping regex/Cucumber expressions, duplicated steps after refactoring, or imported steps from multiple glue packages. Resolve by **tightening expressions**, **renaming**, or **consolidating** the definitions.

### Q25. What is Gherkin linting and why is it useful?
**Answer:** Tools like **gherkin-lint** enforce conventions: max scenarios per feature, no trailing spaces, required tags, mandatory "So that", no duplicate scenario names, etc. Running them in **pre-commit hooks / CI** prevents stylistic drift and ensures the suite stays readable as it scales.

---

## Section 3 — Cucumber Framework Core (Q26 – Q45)

### Q26. What is Cucumber and what are its main components?
**Answer:** Cucumber is the most popular BDD automation tool. Its components:
- **Gherkin parser** — reads `.feature` files.
- **Step definition layer** — maps steps to code via annotations.
- **Runner** — orchestrates execution (JUnit / TestNG / CLI).
- **Hooks** — lifecycle callbacks.
- **Plugins / Formatters** — produce output (HTML, JSON, JUnit XML, Allure, etc.).
- **Object Factory** — instantiates step classes (with DI integration).

### Q27. What is a step definition?
**Answer:** A method in a Java class annotated with `@Given`, `@When`, `@Then`, etc. The annotation's expression (Cucumber Expression or Regex) matches Gherkin steps. At runtime, Cucumber parses each step, finds the matching definition, converts parameters, and invokes the method.

### Q28. Cucumber Expressions vs Regular Expressions.
**Answer:**
- **Cucumber Expressions** — modern, human-readable. Built-in types: `{int}`, `{float}`, `{double}`, `{long}`, `{string}`, `{word}`, `{bigdecimal}`. Supports custom types via `@ParameterType`. Easier to write and maintain.
- **Regex** — legacy, more powerful for complex matching (lookaheads, alternation), but harder to read.

Cucumber automatically detects which style you used. Prefer Cucumber Expressions unless regex power is genuinely needed.

### Q29. What is `@ParameterType` and when do you use it?
**Answer:** It defines a **custom Cucumber Expression parameter type**. Example:
```java
@ParameterType("active|inactive|pending")
public Status status(String value) {
    return Status.valueOf(value.toUpperCase());
}
```
Now `{status}` can be used in any step. Use it for enums, domain objects, date formats, money — anywhere you want type-safe, reusable conversions.

### Q30. What are `@DataTableType` and `@DocStringType`?
**Answer:**
- `@DataTableType` — converts a DataTable (or its rows) into a domain object automatically. Variants: entry-based (`Map` per row), cell-based, row-based, table-based.
- `@DocStringType` — converts a DocString (often JSON/XML) into a typed object, matched by content-type hint.

Both eliminate boilerplate conversion code in step definitions.

### Q31. List all Cucumber hooks and their execution order.
**Answer:**
- `@BeforeAll` — once, before any scenarios (static).
- `@Before` — before each scenario.
- `@BeforeStep` — before each step.
- *(step executes)*
- `@AfterStep` — after each step.
- `@After` — after each scenario.
- `@AfterAll` — once, after all scenarios (static).

Hooks support **ordering** (`@Before(order = 10)` — higher runs first for `@Before*`, reversed for `@After*`) and **tag filters** (`@Before("@db")`).

### Q32. Difference between `@Before` and `Background`?
**Answer:** Already covered in Q13 — repeat for emphasis: `Background` is **business-visible Gherkin**; `@Before` is **invisible technical glue**. They serve different audiences.

### Q33. How do tagged hooks work?
**Answer:** `@Before("@web")` runs only for scenarios tagged `@web`. Supports the same boolean expressions as Gherkin tag filters: `@Before("@api and not @smoke")`. Enables **scenario-type-specific setup** (e.g., browser launch only for UI tests).

### Q34. What is the `Scenario` object injected into hooks?
**Answer:** A handle giving access to:
- `scenario.getName()`, `getId()`, `getUri()`, `getLine()`
- `scenario.getSourceTagNames()`
- `scenario.isFailed()`, `getStatus()`
- `scenario.attach(byte[], mediaType, name)` — attach screenshots, logs, JSON to the report
- `scenario.log(String)` — emit messages into the report

Commonly used in `@AfterStep` to capture a screenshot on failure.

### Q35. What is `dryRun` and why is it useful?
**Answer:** When `dryRun = true`, Cucumber **parses features and matches steps but does not execute them**. Used to:
- Verify that every Gherkin step has a definition.
- Detect ambiguous / duplicate definitions early in CI.
- Generate snippets for missing steps quickly.

It is a fast linter for the glue layer.

### Q36. Snippet types: CAMELCASE vs UNDERSCORE.
**Answer:** When Cucumber finds an undefined step, it suggests a method snippet. `CAMELCASE` produces `iHaveCucumbers` style; `UNDERSCORE` produces `i_have_cucumbers`. Choose one per team and stick to it for consistency.

### Q37. What is the `rerun` plugin?
**Answer:** A built-in formatter (`rerun:target/rerun.txt`) that writes the **file paths and line numbers** of failed scenarios. A second runner can then read this file as its `features` source to rerun only the failures, dramatically reducing CI feedback time on flaky tests.

### Q38. List the built-in Cucumber plugins and their purposes.
**Answer:**
| Plugin | Purpose |
|---|---|
| `pretty` | Rich console output |
| `html:path` | Stand-alone HTML report |
| `json:path` | Machine-readable JSON; feeds other tools |
| `junit:path` | JUnit XML for CI integrations |
| `message:path` | Cucumber Messages ndjson protocol |
| `timeline:path` | Visualizes thread/scenario timing in parallel runs |
| `rerun:path` | List of failed scenarios for reruns |
| `progress` | Dots in console |
| `usage:path` | Step-definition usage statistics |

### Q39. What is the Cucumber Messages protocol?
**Answer:** A language-agnostic, **NDJSON-based event stream** describing everything Cucumber does (sources, pickles, test runs, step results, attachments). It enables tool interoperability — e.g., a single message file can feed Allure, Cucumber Studio, custom dashboards, all without re-parsing features.

### Q40. How do you filter scenarios by line number?
**Answer:** Append `:line` to the feature path: `src/test/resources/features/login.feature:12:25` runs only the scenarios starting on lines 12 and 25. Useful for IDE "run this scenario" actions and targeted debugging.

### Q41. How do you share state between step definition classes?
**Answer:** Through **dependency injection** (PicoContainer, Spring, Guice). Define a `ScenarioContext` / `TestContext` class containing shared state (response, user, cart) and inject it via constructor into each step class. Cucumber creates a **new instance per scenario**, ensuring isolation.

**Anti-pattern:** static fields — not thread-safe and leak state between scenarios.

### Q42. Why are step definitions instantiated per scenario?
**Answer:** To guarantee **scenario isolation** — no leakage of fields between scenarios. Combined with DI, this gives a clean "fresh world" for each scenario, which is essential for **parallel execution** and **deterministic re-runs**.

### Q43. How does Cucumber pick a glue package?
**Answer:** Via the `glue` attribute in `@CucumberOptions` (or `cucumber.glue` system property). Cucumber **scans the package** (and subpackages) for classes containing hook/step annotations. Glue paths are *Java package names*, not file paths.

### Q44. What happens if `glue` is left empty?
**Answer:** Cucumber falls back to scanning **the package of the runner class and its subpackages**. This is convenient for small projects but explicit is better — declare glue paths to avoid loading unrelated steps when the project grows.

### Q45. What is the `ObjectFactory` and when would you customize it?
**Answer:** The `ObjectFactory` is the SPI Cucumber uses to **instantiate step definition classes**. Default implementations exist for PicoContainer, Spring, Guice, CDI. Custom factories are needed when you use a non-supported DI container or have specific scoping rules. Specified via `objectFactory` in `@CucumberOptions` or `cucumber.object-factory` property.

---

## Section 4 — Cucumber + TestNG Integration (Q46 – Q55)

### Q46. Why use TestNG over JUnit for Cucumber?
**Answer:**
- Native **parallel execution** at suite/test/class/method level.
- Rich `testng.xml` for declarative suite configuration.
- Built-in **groups, dependencies, data providers, listeners**.
- Better support for **enterprise CI/CD** (parameterized suites, listener hooks).
- Re-run, retry analyzers, and annotation transformers out of the box.

### Q47. What is `AbstractTestNGCucumberTests`?
**Answer:** The base class provided by `cucumber-testng`. It exposes scenarios as a **TestNG `@DataProvider`** named `"scenarios"`. The subclass typically just adds `@CucumberOptions`. Overriding `scenarios()` with `@DataProvider(parallel = true)` enables parallel execution.

### Q48. How do you enable parallel scenario execution with TestNG?
**Answer:**
```java
@Override
@DataProvider(parallel = true)
public Object[][] scenarios() {
    return super.scenarios();
}
```
And in `testng.xml`:
```xml
<suite name="BDD" parallel="methods" data-provider-thread-count="4"/>
```
Set `dataproviderthreadcount` to control the worker pool. Each scenario then runs in its own thread.

### Q49. How do you make WebDriver thread-safe in parallel runs?
**Answer:** Use `ThreadLocal<WebDriver>`:
```java
private static final ThreadLocal<WebDriver> DRIVER = new ThreadLocal<>();
public static WebDriver get() { return DRIVER.get(); }
public static void set(WebDriver d) { DRIVER.set(d); }
public static void remove() { DRIVER.get().quit(); DRIVER.remove(); }
```
Combined with `@Before` to instantiate and `@After` to call `remove()`, each thread holds its own driver. Never share a single `WebDriver` field statically.

### Q50. What CucumberOptions attributes do you use most often?
**Answer:** `features`, `glue`, `plugin`, `tags`, `monochrome`, `dryRun`, `snippets`, `name`, `publish`, `objectFactory`. (See full table in Phase 4.3 of the Study Plan.)

### Q51. How do you run only `@smoke` scenarios from Maven?
**Answer:**
```bash
mvn test -Dcucumber.filter.tags="@smoke"
```
Or set in `@CucumberOptions(tags = "@smoke")`. The system property overrides the annotation, which is convenient for CI variants.

### Q52. How do you integrate TestNG listeners with Cucumber?
**Answer:** Annotate the runner with `@Listeners(MyListener.class)` or declare listeners in `testng.xml`. `ITestListener` hooks (`onTestStart`, `onTestFailure`, etc.) fire around each TestNG method — and since each scenario *is* a TestNG method when using `AbstractTestNGCucumberTests`, you get per-scenario callbacks.

### Q53. How does `IRetryAnalyzer` interact with Cucumber?
**Answer:** A TestNG `IRetryAnalyzer` can retry failed scenarios at the TestNG layer. However, it retries the **entire Cucumber pickle**, and hooks/state must be idempotent. Often the simpler **`rerun` plugin + a second runner** is preferred because it is Cucumber-native and integrates with CI reruns.

### Q54. Parallel by Scenario vs Parallel by Feature.
**Answer:**
- **Parallel by Scenario** — finer granularity, better CPU utilization, but state isolation must be perfect.
- **Parallel by Feature** — coarser, simpler; scenarios in the same feature still run sequentially, which helps when feature-level setup is expensive.

TestNG + Cucumber-JVM defaults to per-scenario via the data provider model.

### Q55. How do you organize multiple runners?
**Answer:** Common splits:
- **By module** — `OrderRunner`, `PaymentRunner`.
- **By suite type** — `SmokeRunner`, `RegressionRunner`, `FailedRerunRunner`.
- **By layer** — `ApiRunner`, `UiRunner`.

Each runner overrides `@CucumberOptions(tags=…, features=…)`; `testng.xml` orchestrates which runners to execute in each pipeline stage.

---

## Section 5 — Dependency Injection (Q56 – Q62)

### Q56. Why is DI essential in Cucumber?
**Answer:** Step definitions are intentionally split across multiple classes (organized by domain). They need to share state (e.g., a logged-in user, an API response). DI provides **scenario-scoped instances** of shared context, avoiding global statics that break parallelism and isolation.

### Q57. How does PicoContainer work with Cucumber?
**Answer:** Add `cucumber-picocontainer` to the classpath — no configuration required. Cucumber detects it via the ObjectFactory SPI and uses **constructor injection** to wire step classes. A shared `TestContext` class is automatically instantiated once per scenario and injected wherever requested.

### Q58. How does Spring integration differ from PicoContainer?
**Answer:** With `cucumber-spring`:
- Annotate a class with `@CucumberContextConfiguration` and Spring annotations (`@SpringBootTest`, `@ContextConfiguration`).
- Use `@Autowired` for injection.
- Use `@ScenarioScope` for beans that should be recreated per scenario.
- Gain full Spring features: profiles, property sources, test slices, transactions.

Spring is heavier but ideal when the **application under test is already Spring-based**.

### Q59. What is `@ScenarioScope` (Spring) / `@ScenarioScoped` (Guice)?
**Answer:** A custom scope that makes a bean live exactly as long as a single Cucumber scenario. Equivalent to PicoContainer's default behavior. Without it, Spring beans would be singletons and leak state between scenarios.

### Q60. Can you mix DI containers?
**Answer:** Generally **no** — Cucumber resolves a single `ObjectFactory` and one container manages step instantiation. Mixing would create competing instantiation paths. If you need both (e.g., Spring app + lightweight test scope), use Spring as the single container and configure its scopes accordingly.

### Q61. What is a `ScenarioContext` / `TestContext` class?
**Answer:** A simple POJO that holds shared per-scenario state — e.g., the current `User`, `Cart`, last `Response`, captured `screenshotPath`. Injected into every step class via the DI container. Treats Cucumber steps as **collaborators acting on a shared world model**.

### Q62. Common pitfalls with shared context.
**Answer:**
- Letting context grow into a "god object" — split by domain (e.g., `UserContext`, `OrderContext`).
- Putting framework objects (WebDriver) into business context — keep them in dedicated wrappers.
- Mutating context unpredictably from background steps — document expectations.
- Forgetting to clear context — DI handles this if scoping is correct.

---

## Section 6 — Design Patterns & Architecture (Q63 – Q72)

### Q63. Describe the layered architecture of a mature BDD framework.
**Answer:**
```
Feature files          (business language)
Step definitions       (orchestration; thin glue)
Business actions       (reusable, domain-named workflows)
Page Objects / API     (interaction layer)
Drivers / Config / Utils (technical infrastructure)
```
Each layer talks **only** to the one below. Step definitions should not contain `findElement` calls; Page Objects should not contain assertions.

### Q64. What is the Page Object Model and how does it apply to BDD?
**Answer:** POM encapsulates a page's **locators** and **actions** into a class, hiding HTML structure from tests. In BDD, step definitions call page object methods (`loginPage.loginAs(user)`), keeping Gherkin business-focused and shielding the suite from UI churn — locator changes are made in **one place**.

### Q65. What is the Screenplay Pattern?
**Answer:** An advanced alternative to POM championed by Serenity BDD. Concepts:
- **Actor** — performs tasks.
- **Ability** — what an actor can do (BrowseTheWeb, CallAnApi).
- **Task** — high-level business action.
- **Interaction** — low-level UI/API operation.
- **Question** — reads state for assertions.

It adheres better to SOLID (especially SRP and OCP) and scales better than POM in large suites, at the cost of more classes and a learning curve.

### Q66. What design patterns commonly appear in BDD frameworks?
**Answer:**
- **Page Object / Screenplay** — UI abstraction.
- **Factory** — driver creation, test-data creation.
- **Singleton** — configuration manager (carefully, thread-safe).
- **Builder** — fluent test data.
- **Strategy** — pluggable browsers / environments.
- **Decorator** — driver wrappers (logging, screenshots).
- **Dependency Injection** — scenario state.
- **Template Method** — base test/page classes.

### Q67. How do you manage configuration across environments?
**Answer:**
- Hierarchy: defaults in `config.properties` → overridden by `env.properties` (dev/qa/stg) → overridden by JVM args / env vars.
- A typed `ConfigManager` (Owner library, or a custom Singleton).
- Never commit secrets — use env vars / vault / GitHub Actions secrets.
- Switch environments via `-Denv=qa`.

### Q68. How do you generate and manage test data?
**Answer:**
- **Static** — JSON/CSV/Excel files for stable reference data.
- **Dynamic** — Faker / Datafaker library for unique values.
- **Builders** — `UserBuilder.aPremiumUser().withEmail(...).build()`.
- **APIs** — call setup endpoints to seed.
- **DB** — transactional inserts rolled back in `@After`.

Aim for **independent**, **isolated**, and **idempotent** scenarios.

### Q69. How should you keep step definitions "thin"?
**Answer:** Step methods should:
1. Convert Gherkin parameters into domain objects.
2. Delegate to an action / page object / API client.
3. Pull a result and assert.

If a step contains loops, conditionals, or `findElement`, it is doing too much. Push that logic into action classes.

### Q70. Should assertions live in step definitions or page objects?
**Answer:** **Step definitions.** Page Objects are the *interaction layer*; mixing assertions in them couples them to specific tests and breaks SRP. Step definitions own the *Then* — they should read state via page objects and assert. Some teams put assertions in dedicated `Verifier` / `Question` classes (Screenplay-style).

### Q71. How do you handle waits in a BDD framework?
**Answer:**
- Prefer **explicit waits** (`WebDriverWait` + `ExpectedConditions`) inside page objects.
- Avoid `Thread.sleep` — it is the #1 cause of flakiness.
- Centralize wait constants and conditions in a base page.
- For APIs, use polling utilities (Awaitility) with sensible timeouts and intervals.

### Q72. How do you support multi-layer testing (same Gherkin, UI + API)?
**Answer:** Write feature files in **layer-agnostic business language**. Provide **two step definition packages** (`steps.ui`, `steps.api`). Choose at runtime via `glue` and tags (e.g., `@ui` vs `@api`). The same scenarios verify behavior at both layers, encouraging shifting tests *down the pyramid* for speed.

---

## Section 7 — Reporting (Q73 – Q78)

### Q73. Compare built-in Cucumber reporters.
**Answer:** See Q38. Most teams combine `json` (for downstream tools), `html` (for quick local view), `junit` (for CI integrations), and `rerun` (for failures).

### Q74. How do you integrate Allure with Cucumber?
**Answer:**
- Add `allure-cucumber7-jvm` dependency.
- Configure `allure.properties` (results directory).
- Use plugin `io.qameta.allure.cucumber7jvm.AllureCucumber7Jvm`.
- Annotate helpers with `@Step`, `@Attachment` to enrich reports.
- Attach screenshots on failure via `scenario.attach(...)` or Allure's `Attachment`.
- Serve with `allure serve allure-results`.

### Q75. What value does Allure add over the built-in HTML report?
**Answer:**
- **Trends and history** across runs.
- **Categories** for failure classification.
- **Severity, epics, features, stories** annotations.
- **Step-level timing and attachments**.
- **Retry tracking**.
- Better UX for stakeholders.

### Q76. How do you build a custom Cucumber reporter?
**Answer:** Implement `ConcurrentEventListener` (thread-safe) and register event handlers:
```java
publisher.registerHandlerFor(TestCaseStarted.class, e -> ...);
publisher.registerHandlerFor(TestStepFinished.class, e -> ...);
publisher.registerHandlerFor(TestRunFinished.class, e -> ...);
```
Then list the plugin in `@CucumberOptions(plugin = "com.example.MyReporter")`. Use it to push metrics to Slack/Teams/Elastic, build custom dashboards, or apply org-specific reporting rules.

### Q77. How do you attach screenshots only on failure?
**Answer:**
```java
@AfterStep
public void afterStep(Scenario scenario) {
    if (scenario.isFailed()) {
        byte[] png = ((TakesScreenshot) driver).getScreenshotAs(OutputType.BYTES);
        scenario.attach(png, "image/png", scenario.getName());
    }
}
```
Place it in `@AfterStep` (not `@After`) so the screenshot reflects the **failing step**, not post-teardown state.

### Q78. What is the Cucumber Reports service?
**Answer:** A hosted service (reports.cucumber.io). Setting `publish = true` (or env var `CUCUMBER_PUBLISH_ENABLED=true`) uploads results after each run and returns a shareable URL. Useful for distributed teams; for sensitive data use a self-hosted alternative or Allure.

---

## Section 8 — API, UI, Mobile, Specialized Testing (Q79 – Q86)

### Q79. How do you structure API testing with BDD?
**Answer:**
- Use **RestAssured** (or similar) inside step definitions.
- Maintain a **request/response context** in `ScenarioContext`.
- Provide reusable generic steps for HTTP verbs and a layer of **domain-specific steps** for business actions.
- Validate via JSONPath/GPath, JSON Schema, and response time assertions.

### Q80. Generic vs domain-specific API steps — pros and cons.
**Answer:**
- **Generic** (`When I send POST to "/users" with body`) — flexible, fewer steps; but scenarios become technical and imperative.
- **Domain-specific** (`When I register a new premium customer`) — readable, aligns with BDD philosophy; more steps to maintain.

Prefer **domain-specific at scenario level**, with generic helpers underneath.

### Q81. How do you test asynchronous behavior (Kafka, queues)?
**Answer:**
- Publish in `When`.
- In `Then`, **poll** the consumer / DB / downstream API using Awaitility with timeout + interval.
- Use **TestContainers** to spin up real Kafka for high-fidelity tests.
- Tag long-running scenarios so they can be excluded from fast suites.

### Q82. How does contract testing relate to BDD?
**Answer:** Tools like **Pact** define consumer expectations as contracts; providers verify them. BDD scenarios can act as the **specification format** describing each interaction (Given a service state, When a request, Then a response). This combines collaboration (BDD) with structural verification (contracts).

### Q83. How do you integrate Selenium WebDriver with Cucumber + TestNG for parallel runs?
**Answer:** See Q49 — `ThreadLocal<WebDriver>`. Combined with:
- `@Before` to create the driver via a `DriverFactory`.
- `@After` to quit & remove.
- Configuration-driven browser selection.
- Remote execution via `RemoteWebDriver` + Grid / Selenoid / cloud providers.

### Q84. How do you do mobile testing with BDD?
**Answer:** Use **Appium**. Wrap `AppiumDriver` similarly to WebDriver with `ThreadLocal`. Manage **desired capabilities** per platform via config. Write platform-agnostic steps where possible; use tagged scenarios (`@android`, `@ios`) for platform-specific behavior.

### Q85. What is visual / accessibility testing in BDD?
**Answer:**
- **Visual** — tools like Applitools Eyes / Percy add `eyes.check()` calls in step definitions to compare screenshots against a baseline. BDD scenarios can include "Then the homepage matches the visual baseline".
- **Accessibility** — integrate axe-core (`axe.run()`) and assert no WCAG violations. Frame as "Then the page is WCAG 2.1 AA compliant".

Both are excellent additions for shifting non-functional checks left.

### Q86. How do you handle database verification in BDD?
**Answer:**
- Seed via `Given` (preferably through APIs to mirror real usage).
- Verify side effects in `Then` via repository / JDBC.
- Wrap each scenario in a **transaction rolled back in `@After`** for isolation (when feasible).
- Use **Flyway / Liquibase** to manage schema for test environments.
- Never share write-state across scenarios.

---

## Section 9 — Advanced Cucumber & CI/CD (Q87 – Q94)

### Q87. How do you rerun only failed scenarios?
**Answer:** Use the `rerun:target/rerun.txt` plugin. Create a second runner reading `features = "@target/rerun.txt"`. In CI, run the main suite first; if it fails, run the rerun runner. Combine with a retry count cap to avoid masking real failures.

### Q88. How do you achieve true parallelism with Cucumber-JVM?
**Answer:**
- **JUnit 5** — set `cucumber.execution.parallel.enabled=true` and `cucumber.execution.parallel.config.strategy=fixed`.
- **TestNG** — override `scenarios()` with `@DataProvider(parallel=true)` + suite `dataproviderthreadcount`.
- Make **state and resources thread-safe** (ThreadLocal driver, DI scoping, unique data per scenario).
- Be mindful of **shared external resources** (databases, accounts).

### Q89. How do you wire BDD tests into a Jenkins pipeline?
**Answer:**
```groovy
pipeline {
  agent any
  stages {
    stage('Test') { steps { sh 'mvn clean test -Dcucumber.filter.tags="@smoke"' } }
  }
  post {
    always {
      junit 'target/surefire-reports/*.xml'
      cucumber jsonReportDirectory: 'target', fileIncludePattern: '*.json'
      allure includeProperties: false, results: [[path: 'target/allure-results']]
    }
  }
}
```
Use the Cucumber and Allure plugins for visualization; configure email/Slack notifications in `post`.

### Q90. How do you containerize BDD execution?
**Answer:** Build a **Docker image** with JDK, Maven, browsers (or use selenium/standalone-* images). Use **docker-compose** to orchestrate app + DB + Selenium Grid. CI executes `docker compose run tests mvn test`. Benefits: reproducible builds, no "works on my machine".

### Q91. How do you debug a flaky scenario?
**Answer:**
- Reproduce locally with same tags, browser, env.
- Add Allure step logs / screenshots on every step.
- Check for **implicit waits**, **hard-coded sleeps**, **race conditions**, **shared state**, **non-deterministic data** (random IDs colliding), **time-zone** issues, **animation timing**.
- Run scenario in isolation vs in suite — narrows isolation problems.
- Quarantine with `@flaky` tag and track in a backlog; do not delete.

### Q92. How do you handle environment-specific data and secrets?
**Answer:**
- Per-env property files committed to repo (non-sensitive).
- Secrets via env vars, CI secret stores, Vault, AWS Secrets Manager.
- A typed `ConfigManager` reads from sources in priority order.
- Never hard-code credentials in Gherkin — use placeholders or tagged scenarios that resolve from config.

### Q93. How do you keep a large BDD suite fast?
**Answer:**
- Move tests **down the pyramid** (API > UI).
- Run **parallel**.
- Tag and run **smoke** subsets on every push, **regression** nightly.
- Reuse logged-in state via cookie injection / API auth bypass.
- Disable images / use headless browsers.
- Cache static fixtures.
- Profile slow scenarios with `usage` plugin and optimize.

### Q94. How do you implement "subcutaneous" tests?
**Answer:** Skip the UI and call the **application service layer** (controller / facade) directly with the same Gherkin scenarios. Benefits: faster, more stable, still covers business logic end-to-end. Particularly valuable in Spring/Spring-Boot apps where you can `@Autowired` services into step definitions.

---

## Section 10 — BDD Process, Anti-Patterns, Strategy (Q95 – Q100)

### Q95. List the most common BDD anti-patterns.
**Answer:**
- **Imperative scenarios** (UI clicks instead of behavior).
- **Incidental detail** in Gherkin.
- **Scenario coupling / ordering dependency**.
- **Hard-coded test data** in features.
- **BDD without collaboration** — "Gherkin-flavored Selenium".
- **Giant step definitions** — logic that belongs in actions/services.
- **Missing Then** — no assertion.
- **Feature-coupled steps** — not reusable.
- **Dead features** nobody reads.
- **Quarantine creep** — `@skip` everywhere.
- **All tests through UI** — slow and flaky.

### Q96. What is Example Mapping and how do you facilitate it?
**Answer:** A 25-minute structured conversation by **Matt Wynne**:
- **Yellow card** — the story.
- **Blue cards** — rules / acceptance criteria.
- **Green cards** — concrete examples for each rule.
- **Red cards** — open questions / unknowns.

The QA's role is to **ask probing questions** to surface examples and edge cases. The output is a refined backlog item with examples ready to become Gherkin.

### Q97. Where do BDD scenarios fit in the test pyramid?
**Answer:** BDD is **layer-agnostic** — the pyramid shape applies to BDD too. Most scenarios should live at the **service/API layer** (fast, stable). UI BDD scenarios should cover only **critical end-to-end user journeys**. A small number of unit-level "subcutaneous" tests can also use BDD when business clarity is valuable.

### Q98. How do you bring legacy code under BDD?
**Answer:**
- Start with **new features** — do not retrofit everything.
- Hold Three Amigos for upcoming stories; capture as Gherkin.
- Add a thin acceptance test for each bug fixed (regression-driven discovery).
- Refactor toward testable architecture (hexagonal / ports & adapters) to enable subcutaneous tests.
- Avoid the trap of rewriting legacy tests as Gherkin without collaboration — that's pure cost.

### Q99. What does BDD maturity look like at scale?
**Answer:**
| Level | Indicator |
|---|---|
| 0 | No BDD; manual / scripted automation only. |
| 1 | Cucumber used as automation tool; QA writes scenarios alone. |
| 2 | Three Amigos active; collaborative scenario authoring. |
| 3 | Living documentation consumed by the whole team; BDD drives development. |
| 4 | BDD embedded in culture; continuous discovery; metrics on collaboration outcomes. |

Most orgs claim Level 3 but operate at Level 1. The interview-worthy answer is honest about this gap and how you address it.

### Q100. Describe your end-to-end BDD framework architecture.
**Answer:** A model answer:
> *"Cucumber 7 with Java 17 and TestNG. Maven project with `src/test/resources/features` for `.feature` files, `step.definitions` for glue, and `pages`, `api.clients`, `business.actions` for reusable layers. PicoContainer provides per-scenario DI with a `ScenarioContext` holding shared state. Selenium WebDriver is wrapped in a `ThreadLocal` for parallel execution; browsers are configured per environment via a typed `ConfigManager`. RestAssured handles API steps; the same feature files run at UI or API level using tag-based glue selection. Allure produces reports with screenshots attached on failure; the `rerun` plugin generates a failure list consumed by a `FailedRerunTest` runner. Maven Surefire runs the suite in parallel with `dataproviderthreadcount=4`. GitHub Actions runs `@smoke` on every PR and `@regression` nightly, publishing Allure to GitHub Pages. Three Amigos sessions happen during refinement; Example Mapping output feeds new feature files reviewed by the BA. The suite is monitored for flakiness and step usage; dead steps are pruned monthly."*

A senior answer ties **architecture decisions back to BDD principles**: collaboration, living documentation, fast feedback, and scenarios as the single source of truth.

---

## Appendix — Quick Revision Cheat Sheet

| Topic | One-Line Recall |
|---|---|
| BDD origin | Dan North, 2006, response to TDD confusion |
| Core cycle | Discovery → Formulation → Automation |
| Three Amigos | Dev + QA + BA/PO |
| Gherkin keywords | Feature, Rule, Background, Scenario, Outline, Examples, Given/When/Then/And/But |
| Tag operators | and, or, not |
| Cucumber expressions | `{int}`, `{string}`, `{word}`, `{float}`, custom via `@ParameterType` |
| Hook order | BeforeAll → Before → BeforeStep → step → AfterStep → After → AfterAll |
| DI options | PicoContainer, Spring, Guice, CDI |
| Parallel TestNG | `@DataProvider(parallel=true)` + `dataproviderthreadcount` |
| Rerun plugin | `rerun:target/rerun.txt` + second runner |
| Reports | pretty, html, json, junit, message, timeline, allure, extent, serenity |
| Anti-patterns | Imperative steps, scenario coupling, BDD without collab, giant steps |
| Discovery technique | Example Mapping (yellow/blue/green/red cards) |
| Pyramid placement | Most BDD at API layer, few at UI |

---

*End of Document — 100 Questions covering BDD philosophy, Gherkin, Cucumber core, TestNG integration, DI, design patterns, reporting, API/UI/mobile testing, CI/CD, advanced features, and process maturity.*
