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
- No skipped tests (`XCTSkip`, `xit`, commented-out blocks) — if a test is skipped, document the latent gap and create a follow-up
- No `Thread.sleep` or fixed delays — use structured async / expectations

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

**The Fragile Tests Problem:** business rules verified only by driving the volatile UI through XCUITest break in bulk on any UI or navigation change, so the suite gets disabled rather than maintained. The fix is to verify rules through a stable behavioral seam (a view-model API, a use-case method, a pure function) and reserve XCUITest for genuine end-to-end journeys.

**Bugs congregate.** When a test reveals a bug, widen test coverage of the surrounding area before moving on. The same conditions that produced one defect tend to produce others nearby.

**Look for patterns in failures.** When multiple tests fail, find what they share — similar input ranges, the same code path, a specific boundary value — before debugging any individual failure. The pattern names the root cause faster than examining failures in isolation.

**A suite that has never been broken is not trusted.** Occasionally break a branch or flip a condition in production code and confirm the right test goes red. A suite that stays green through any change proves nothing.
