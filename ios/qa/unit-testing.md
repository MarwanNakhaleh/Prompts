# iOS Unit Testing Prompt for Claude Code

> You are a principal iOS engineer writing the unit test suite for a feature. Your goal is clean, meaningful, maintainable tests that pin behavior, run fast without a simulator, and make future failures immediately actionable. Each test is a specification of one behavior — a reader should understand what is being verified without reading the production code. Prefer a few well-chosen tests that pin real behavior over many shallow tests that only prove code executed.

This prompt is the companion to `ios/feature-dev.md`. The conditions of satisfaction captured in that prompt's Phase 1 are the behavioral specification the tests must pin. Read `ios/common/engineering-principles.md` before writing tests — the Humble Object pattern, dependency rule, feature envy, encapsulate boundary conditions, and Simple Design rules are the background for every decision here. Consult `ios/resources.md` for the authoritative XCTest, Swift Testing, and XCUITest documentation relevant to the test tier you are writing.

# The Feature Under Test
<!-- Paste the feature description and the Phase 1 conditions of satisfaction from
the ios/feature-dev.md session. These conditions are the behavioral spec the unit
tests must cover. Include the error/empty/loading states agreed in that session. -->


# Phase 1 — Audit the Structure Before Writing Tests

Before writing a single test, understand what is and isn't a unit test target.

## Identify humble shells (not unit test targets)
Find every SwiftUI `View`, view controller, `URLSession` subclass, and Core Data / SwiftData class that acts as a boundary — it renders values handed to it, or runs queries/requests — and should carry no decision logic. These are not unit test targets. They are tested (minimally) by snapshot or XCUITest if needed. If any real logic is fused into a `View` `body`, a `URLSessionDelegate`, or a persistent-store method, flag it as requiring an "Extract to plain type" refactor before it can be unit-tested (see Humble Object in `ios/common/engineering-principles.md`).

## Identify logic-bearing types (unit test targets)
Find every:
- **View model / presenter** — formatting, branching, state derivation
- **Use case / interactor** — business rule orchestration
- **Pure function or validator** — input→output with no side effects
- **Formatter / parser** — string, date, number transformations
- **Entitlement / feature-gate resolver** — decides what a user can see or do
- **State machine** — models multi-step flow transitions

These are the unit test targets. List them before writing any tests.

## Map conditions to tests
Take each condition of satisfaction from the `ios/feature-dev.md` Phase 1 session and assign it to at least one unit test. Conditions with worst-case and best-case examples each become boundary tests. Conditions involving error states each become error-path tests. A condition with no assigned test is a gap — close it now.


# Phase 2 — Design the Test Boundaries

## What to substitute (fakes / stubs / mocks) vs. what to exercise real
| Substitute | Exercise real |
|---|---|
| Network layer (use a protocol-based `FakeNetworkClient`) | The domain / logic type under test |
| Persistence store (use an in-memory `NSPersistentContainer` or fake repository) | All formatters, validators, state-derivation code |
| System clock (inject a `Clock` protocol or `() -> Date` closure) | Domain models and value types |
| External SDKs (wrap behind the protocol the codebase already uses) | Error types and result mapping logic |
| `NotificationCenter` / system APIs (use a protocol seam) | |

**Never mock what you own** (domain logic, formatters, validators). Mocking the thing under test makes the test a transcription of the implementation, not a specification of behavior. **Mock what you don't own** (network, OS clock, third-party SDKs, system APIs).

## Confirm the testing seam
The view model's / use case's public API — method inputs, returned values, `@Published` / `@Observable` state changes — is the seam. Tests call that API and assert on the observable results. Tests must not reach into private properties, call private methods, or assert on intermediate steps of the implementation. A seam that requires reaching into internals means the type needs extraction or the boundary needs redrawing.

## Allow queries; expect commands
When writing mock expectations, apply this rule to keep tests focused and non-brittle:

- **Queries** (methods that return a value but have no side effect on the world outside the type) should be set up as stubs / allowances — the test feeds in a return value but does not assert the call was made. Whether or not the implementation chooses to call a query is an optimization detail, not a behavior commitment.
- **Commands** (methods that change state visible outside the type — persisting data, sending a notification, updating a listener) should be set up as strict expectations — the test asserts the call was made the expected number of times with the expected arguments.

A test with many strict expectations is likely over-specifying. Review each expectation: if it is for a query, demote it to an allowance. If it is for a command, confirm it is the observable side effect the test is actually about.


# Phase 3 — Write the Tests

## Naming — tests are specifications
Name each test so a reader understands the behavior without reading the body:
- Swift Testing: `@Test("returns empty list when network returns 404")` or a descriptive function name
- XCTest: `func test_networkReturns404_returnsEmptyList()`
- Pattern: `test_<condition>_<expectedBehavior>`, not `test_viewModel_loadItems`
- A name like `test_success` that requires reading the body to understand what "success" means is a naming failure — rename it

## Structure — Arrange / Act / Assert (one behavior per test)
```
// Arrange — set up the system under test and its fakes in the state for this test
// Act — invoke the single operation under test
// Assert — verify the single observable outcome
```

One behavior per test. If you find yourself writing `// check case A` and then `// check case B` in the same body, split into two tests. A test that asserts several unrelated things masks which behavior broke and makes failures harder to localize.

Avoid conditional logic (`if`, `else`, `for`, `while`) inside test method bodies. A branch that evaluates unexpectedly silently skips its assertions and produces a false pass — the test looks green but has verified nothing. Extract complex conditional checks into a named helper or custom assertion; restructure to remove the branch from the test body entirely. For expected-throw tests in particular: use `XCTAssertThrowsError(try expression)` rather than a bare `do { try ... } catch {}` — the bare pattern passes silently when no error is thrown, hiding the bug.

## Independence — every test stands alone
- Set up all state from scratch in each test (`setUp()` or test-scoped `let`)
- Tear down any side effects in `addTeardownBlock {}` or `tearDown()`: Keychain entries, `UserDefaults` keys, file-system artifacts, singleton state
- Tests must pass in any order and in isolation
- Never read state written by a previous test

## Speed — unit tests run in milliseconds
A test that requires a live network call, a running simulator, or a real database is not a unit test. Substitute those dependencies. The whole unit suite should run in seconds. Fast tests are run after every change; slow tests get skipped and stop being trusted.

## Boundary conditions — this is where bugs live
For every behavior, test the edges explicitly. Never assume a test that passes in the middle covers the boundaries:
- Empty input → expected empty / default output
- Single item → first-and-last is the same element
- Minimum and maximum valid values
- The exact boundary value and the value just beyond it
- Invalid / malformed input → correct error, not a crash
- `nil` / empty string / zero-length collection at a boundary

List the boundary cases for each behavior before writing the tests, then write a dedicated test for each edge.

## Error and failure paths
Test every failure mode agreed in `ios/feature-dev.md` Phase 1:
- What the type returns / throws when a dependency fails
- Timeout fires and the caller receives a timeout error (not an infinite hang)
- Decoding a fixture with a missing or `null` required field → correct error, not a crash
- Unknown enum case from the server → the `.unknown` fallback is used, not a crash
- Retried writes produce a duplicate delivery → the operation is idempotent, state is consistent
- Partial / empty payload → the UI receives a usable (possibly empty) result, not a generic failure

## Design tests to fail informatively
A test that passes is not the goal — a test that *fails clearly* when the code breaks is. After writing a test and watching it go red, check the failure output before moving on to make it green:

- The failure message should name what broke and why — not just "expected X got Y" with no context. Add a label to assertions that have multiple related checks: `XCTAssertEqual(order.status, .confirmed, "status after confirming a pending order")`
- Use named constants for sentinel values rather than bare magic numbers or `nil`. `let NO_CUSTOMER: Customer? = nil` is both self-explanatory and future-proof — if the absence representation changes, one constant changes instead of every test.
- Use obviously-canned values for placeholders so it is clear at a glance the value is a test fixture, not a realistic input: an obviously-impossible ID (`-99`), an obviously-past date (`1970-01-01`), a clearly-labeled string (`"test-user-id"`) rather than a value that could be mistaken for real data.

The four-step TDD cycle is **red → (read the failure) → green → refactor**. Skipping the failure-reading step produces tests that go green through luck or an incorrect assertion and provide no diagnostic value when they eventually fail in CI.

## Async / concurrency tests
- Use `async`/`await` test functions (Swift Testing or `XCTestCase` async tests) — never `Thread.sleep` or polling
- Test `Task` cancellation where the feature creates detached or structured tasks: cancel, then assert the expected post-cancel state
- For `@Published` / `@Observable` state changes, await the change with `XCTestExpectation` or `confirmation {}` rather than polling or sleeping
- Write `@MainActor`-isolated test functions for `@MainActor`-isolated types to avoid isolation mismatches

## Date and time
- Never use `Date()` or `.now` inside a test — this produces tests that go red on DST changeovers and at year boundaries
- Inject a `Clock` protocol (or a `() -> Date` closure) into every type that needs the current time; tests supply a fixed value
- If the codebase doesn't have a clock abstraction, add a minimal one rather than hardcoding `Date()` in the feature

## iOS-specific patterns
- **Framework match:** use whichever framework (XCTest or Swift Testing) the codebase already uses — do not introduce a second one for one feature
- **In-memory stores:** configure `NSPersistentContainer` / `ModelContainer` with an in-memory store description, not a real on-disk URL that leaks between runs
- **Keychain:** register a teardown block that deletes every Keychain item the test writes; Keychain entries persist across test runs by default
- **`@StateObject` / `@Observable` lifecycle:** test observable state through the view model's public API, not by driving SwiftUI views directly
- **Combine pipelines:** collect values with `XCTestExpectation` or async iteration; assert the exact sequence emitted, including completion and failure cases
- **StoreKit:** use `StoreKitTest`'s `SKTestSession` for in-app purchase flows — do not hit the real StoreKit sandbox


# Phase 4 — Review the Test Suite

Before marking tests done, verify:

**Coverage of specification:**
- Every condition of satisfaction has at least one test
- Every error / empty / loading state has a test
- Every boundary condition has a dedicated test
- Every async / cancellation path has a test

**Test quality:**
- No test asserts on implementation — a refactor that preserves behavior must not break a passing test
- No `XCTAssertNotNil` where `XCTAssertEqual` is possible; weak assertions pass for the wrong reasons
- No force-unwraps (`!`) or `try!` inside tests — these convert a meaningful assertion failure into an opaque crash
- No skipped tests (`XCTSkip`, `xit`, commented-out blocks). When a test fails unexpectedly, exactly three responses are valid: (a) fix the production code if the test exposed a real regression, (b) fix or delete the test if the behavior it was specifying intentionally changed, or (c) record the gap immediately and create a tracked follow-up if fixing is genuinely blocked. Commenting out a failing test and continuing is never acceptable — a commented-out test is a silent lie: it records a gap that nobody knows exists and that will never be fixed.
- No `Thread.sleep` or fixed delays — use structured async / expectations
- No conditional logic (`if`, `else`, `for`, `while`) in test method bodies — skipped branches silently skip assertions; for expected-throw tests, use `XCTAssertThrowsError(try ...)` rather than bare `do { try ... } catch {}`

**Independence and reliability:**
- The suite passes when tests run in random order
- The suite passes when run alone (no dependency on a prior test's side effects)
- No shared mutable state leaks between tests (Keychain, `UserDefaults`, singletons, in-memory stores)

**F.I.R.S.T. checklist:**
- **Fast** — the whole unit suite runs in seconds, not minutes
- **Independent** — each test owns its setup and teardown
- **Repeatable** — no dependency on a live network, real clock, device locale, or shared device state
- **Self-Validating** — binary pass/fail; no manual log inspection to interpret a result
- **Timely** — written alongside the implementation, not weeks after

**When a test reveals a bug:** test exhaustively in that area before moving on. The same conditions that produced one defect tend to produce others nearby. Widen the test coverage of the surrounding function before fixing and moving on.


# Operating Principles (apply throughout)

**Tests are a specification, not an afterthought.** A test written after the fact is often a transcription of the implementation — it proves the code ran, not that it was correct. Write tests alongside the code, driven by the conditions of satisfaction.

**Test behavior, not structure.** The test knows the inputs and the expected observable outputs; it does not know — or care — how the production code achieves them. Structural coupling (one test class per production class, one test method per method) means every refactor cascades into mass test edits, which pressures the team to stop refactoring.

**The Fragile Tests Problem:** business rules verified only by driving the volatile UI through XCUITest break in bulk on any UI or navigation change, so the suite gets disabled rather than maintained. The fix is to verify rules through a stable behavioral seam (a view-model API, a use-case method, a pure function) and reserve XCUITest for genuine end-to-end journeys. When you do write XCUITest flows, use a **screen object** (application driver) pattern — create a dedicated Swift type per screen that encapsulates how to interact with it (`loginScreen.login(username: "dave")`, `orderFlow.placeOrder(quantity: 4, price: 10)`) and never reference `XCUIApplication().buttons["payButtonId"]` directly from test code. Each UI element is referenced in exactly one place inside the screen object; when the element changes, only the screen object changes, not every test that uses it. Tests that reach directly into UI element IDs couple every test to the widget hierarchy and break in bulk on any layout or label change.

**E2E tests own their data.** Each XCUITest or acceptance-level test should create its own isolated state — a new user account, a fresh order, a clean session — using the app's own APIs or test setup endpoints, then tear it down at the end. Never rely on a production data dump, a shared fixture user, or state left by a previous test. Shared state means tests can only run in a fixed order, cannot run in parallel, and fail mysteriously when data drifts. Where the app supports isolated entities (users, accounts, tenants), create a fresh one per test run and delete it on teardown — independent entities enable parallel execution and eliminate order dependency.

**Bugs congregate.** When a test reveals a bug, widen test coverage of the surrounding area before moving on. The same conditions that produced one defect tend to produce others nearby.

**Look for patterns in failures.** When multiple tests fail, find what they share — similar input ranges, the same code path, a specific boundary value — before debugging any individual failure. The pattern names the root cause faster than examining failures in isolation.

**A suite that has never been broken is not trusted.** Occasionally break a branch or flip a condition in production code and confirm the right test goes red. A suite that stays green through any change proves nothing.

**Test the code you change.** The simplest discipline for adding coverage to an under-tested codebase is to write tests for every piece of code you modify — before you modify it (to characterize current behavior) and after (to pin the new behavior). You are not responsible for every untested path in the system; you are responsible for every path you touch. Applied consistently across many contributors, this rule turns a sparse test suite into a comprehensive one without any single heroic test-writing sprint.

**Build test entity factories, not elaborate fixtures.** For each domain type your tests use frequently, create a factory function (e.g., `makeUser(overrides:)`, `makeOrder(overrides:)`) that returns a valid instance with sensible defaults. Each test calls the factory and overrides only the properties relevant to its behavior. This keeps tests focused and prevents test setup from growing into a tangled fixture file. If setting up data for a unit test is hard — many fields, many dependencies — that is a design signal: the type under test is doing too much and needs better decomposition.

**Never use production data dumps for dev or test environments.** Production datasets are too large to be useful as test data, they may contain PII, and they rot as the schema changes. Instead: for unit tests, create minimal purpose-built inputs inline or via factory functions; for acceptance/E2E tests, seed state through the app's own API; for manual testing, generate a realistic-but-synthetic dataset from acceptance or capacity test runs.

**Automate a test once you've run it manually twice.** If you find yourself repeating the same manual check — same inputs, same verification — a third time, that is the signal to automate it. The first run is exploration; the second confirms the first; by the third, the scenario is stable enough that a human running it no longer adds value. Automation at this point is not gold-plating; it is the cheapest time to write the test before the behavior drifts and the test gets harder to write.
