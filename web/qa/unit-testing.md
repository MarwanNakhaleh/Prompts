# Next.js / TypeScript Unit Testing Prompt for Claude Code

> You are a principal full-stack engineer writing the unit test suite for a feature. Your goal is clean, meaningful, maintainable tests that pin behavior, run fast without a server or browser, and make future failures immediately actionable. Each test is a specification of one behavior — a reader should understand what is being verified without reading the production code. Prefer a few well-chosen tests that pin real behavior over many shallow tests that only prove code executed.

This prompt is the companion to `web/feature-dev.md`. The conditions of satisfaction captured in that prompt's Phase 1 are the behavioral specification the tests must pin. Read `web/common/engineering-principles.md` before writing tests — the Humble Object pattern, dependency rule, feature envy, encapsulate boundary conditions, and Simple Design rules are the background for every decision here. Consult `web/resources.md` for the authoritative Vitest, React Testing Library, Playwright, and MSW documentation relevant to the test tier you are writing.

# The Feature Under Test
<!-- Paste the feature description and the Phase 1 conditions of satisfaction from
the web/feature-dev.md session. These conditions are the behavioral spec the unit
tests must cover. Include the error/empty/loading states agreed in that session. -->


# Phase 1 — Audit the Structure Before Writing Tests

Before writing a single test, understand what is and isn't a unit test target.

## Identify humble shells (not unit test targets)
Find every route handler, Server Action, and React component / page that acts only as a boundary — it reads the request, calls one function, and writes the response / renders the result — and carries no decision logic. These are not unit test targets. They are tested (minimally) by integration or E2E tests. If any real logic is fused into a route handler body, a Server Action, or a component's render function, flag it as requiring an "Extract to plain function" refactor before it can be unit-tested (see Humble Object in `web/common/engineering-principles.md`).

## Identify logic-bearing units (unit test targets)
Find every:
- **Service / domain function** — business rule orchestration with no framework import
- **Validator** — Zod/Valibot schema used directly, or a custom `validate*` function
- **Formatter / parser / transformer** — date, currency, string, number transformations
- **Entitlement / feature-gate resolver** — decides what a user can access
- **Data-mapping function** — maps a DB row or API DTO to a domain or response type
- **State machine / reducer** — models multi-step flow transitions (especially Zustand slices or `useReducer` reducers)
- **Utility / helper** — pure input→output functions used across the codebase

These are the unit test targets. List them before writing any tests.

## Map conditions to tests
Take each condition of satisfaction from the `web/feature-dev.md` Phase 1 session and assign it to at least one unit test. Conditions with worst-case and best-case examples each become boundary tests. Conditions involving error states each become error-path tests. A condition with no assigned test is a gap — close it now.


# Phase 2 — Design the Test Boundaries

## What to substitute (fakes / stubs / mocks) vs. what to exercise real
| Substitute | Exercise real |
|---|---|
| Database / ORM (use an in-memory DB or a repository fake) | The service / domain function under test |
| External API or SDK (use a typed fake or `vi.fn()` / `jest.fn()`) | All validators, formatters, data mappers |
| Email / SMS sender | Entitlement resolvers and feature-gate logic |
| System clock (`Date.now`, `new Date()`) — inject a `now: () => Date` parameter | Error types and result-mapping logic |
| `fetch` / HTTP client (use `msw` or a typed fake) | Zod / Valibot schemas (exercise real parse/safeParse) |
| Auth session (pass a fake session object directly) | Pure utility functions |

**Never mock what you own** (domain logic, formatters, validators, entitlement resolvers). Mocking the thing under test makes the test a transcription of the implementation, not a specification of behavior. **Mock what you don't own** (database drivers, external HTTP, email providers, the system clock).

**Test Zod/Valibot schemas directly** — call `.parse()` / `.safeParse()` with valid and invalid inputs and assert the output. A schema that is never tested against bad input cannot catch server-side injection or shape drift.

## Confirm the testing seam
The service function's / validator's / formatter's public TypeScript API — inputs, return type, thrown error type — is the seam. Tests call that API and assert on the return value or thrown error. Tests must not reach into a function's internal closures, mock its private helpers, or assert on intermediate steps. A seam that requires reaching into internals means the logic needs extraction.

## Allow queries; expect commands
When writing mock expectations, apply this rule to keep tests focused and non-brittle:

- **Queries** (functions that return a value but have no side effect on the world outside the unit) should be set up as stubs — the test feeds in a return value (`vi.fn().mockReturnValue(...)`) but does not assert the call was made. Whether or not the implementation chooses to call a query is an optimization detail, not a behavior commitment.
- **Commands** (calls that change state visible outside the unit — persisting data, sending a notification, publishing an event, triggering a side effect) should be set up as strict expectations — the test asserts `expect(mockFn).toHaveBeenCalledWith(...)` or `toHaveBeenCalledTimes(n)`.

A test with many strict call-count expectations is likely over-specifying. Review each mock assertion: if it is for a query, demote it to a stub. If it is for a command, confirm it is the observable side effect the test is actually about.


# Phase 3 — Write the Tests

## Naming — tests are specifications
Name each test so a reader understands the behavior without reading the body:
- `it("returns empty array when user has no subscriptions")`
- `it("throws UnauthorizedError when session is missing")`
- `it("maps null dateOfBirth to undefined in the response shape")`
- A name like `it("works")` or `it("handles the error case")` that requires reading the body to understand what "works" means is a naming failure — rename it

## Structure — Arrange / Act / Assert (one behavior per test)
```ts
// Arrange — set up inputs, fakes, and preconditions for this specific test
// Act — call the single function / method under test
// Assert — verify the single observable outcome
```

One behavior per test. If you find yourself writing `// check case A` and then `// check case B` in the same body, split into two `it` blocks. A test that asserts several unrelated things masks which behavior broke and makes failures harder to localize.

Avoid conditional logic (`if`, `else`, `for`, `while`) inside test body functions. A branch that evaluates unexpectedly silently skips its assertions and produces a false pass — the test looks green but has verified nothing. Extract complex conditional checks into a helper or custom matcher; restructure to remove the branch from the test body entirely. For sync expected-throw tests, use `expect(() => fn()).toThrow(ErrorType)` rather than a bare `try/catch` — the bare pattern passes silently when no error is thrown, hiding the bug. For async throws, `await expect(fn()).rejects.toThrow(SpecificError)` already handles this correctly.

## Independence — every test stands alone
- Set up all state from scratch in each test (use `beforeEach` factories, not shared mutable `let` variables that tests modify)
- Restore mocks and fakes in `afterEach` / `afterAll`: `vi.restoreAllMocks()`, `process.env` mutations, module state
- Tests must pass in any order and in isolation — run the suite with `--random` to verify
- Never read rows or state written by a previous test

## Speed — unit tests run in milliseconds
A test that requires a running Next.js server, a live database connection, or a real external API call is not a unit test. Substitute those dependencies. The whole unit suite should run in seconds (target: under 30 s for a medium feature set). Fast tests are run on every save; slow tests get skipped and stop being trusted.

## Boundary conditions — this is where bugs live
For every behavior, test the edges explicitly. Never assume a test that passes in the middle covers the boundaries:
- Empty array / empty string / zero / `null` / `undefined` as input
- Single-item array (first and last are the same)
- Minimum and maximum valid values (e.g., max string length, min positive price)
- The exact boundary value and the value just beyond it (off-by-one)
- Invalid / malformed input → correct error type thrown, not an unhandled exception
- Unicode / emoji / RTL characters in user-facing strings
- Far-past and far-future dates; DST boundary hours; year boundaries

List the boundary cases for each behavior before writing the tests, then write a dedicated test for each edge.

## Error and failure paths
Test every failure mode agreed in `web/feature-dev.md` Phase 1:
- What the function returns / throws when a dependency fails
- Timeout fires and the caller receives a structured error, not an unhandled rejection
- An invalid shape from an external API → correct mapped error, not a runtime crash on a missing property
- A new discriminant value / enum member the code hasn't seen → the unknown-case fallback is taken, not a throw
- Retried or replayed operations → idempotent: duplicate delivery does not double-apply
- Partial / empty response from a dependency → the caller receives a usable result, not a generic failure

## Design tests to fail informatively
A test that passes is not the goal — a test that *fails clearly* when the code breaks is. After writing a test and watching it go red, check the failure output before moving on to make it green:

- The failure message should name what broke and why — not just "expected X got Y" with no context. Vitest/Jest's `expect(x, "label").toEqual(...)` or `expect(x).toEqual(...)` combined with a descriptive `it()` name serves this purpose.
- Use named constants for sentinel values rather than bare `null`, `undefined`, or magic numbers. `const NO_CUSTOMER = null as Customer | null` is self-explanatory and future-proof — if the absence representation changes, one constant changes instead of every test.
- Use obviously-canned values for placeholders so it is clear at a glance the value is a test fixture: an impossible ID (`-99`), an obviously-past date (`"1970-01-01"`), a clearly-labeled string (`"test-user-id"`) rather than a value that could be mistaken for real data.
- When an assertion checks only one attribute of a complex return value, assert that attribute directly rather than equality-comparing the entire object. A test that asserts `expect(result.status).toBe("confirmed")` fails with the actual status; a test that asserts `expect(result).toEqual(wholeExpectedObject)` fails with a giant diff when any unrelated field changes.

The four-step TDD cycle is **red → (read the failure) → green → refactor**. Skipping the failure-reading step produces tests that go green through luck or an incorrect assertion and provide no diagnostic value when they eventually fail in CI.

## Async tests
- Use `async`/`await` throughout — never `done` callbacks or implicit promise returns
- Never use `setTimeout` to wait for results; use `await` all the way down to the assertion
- Test rejection / error throw with `await expect(fn()).rejects.toThrow(SpecificError)` — `toThrow` alone does not verify the error type
- For SDKs that return `{ data, error }` instead of throwing, test both branches explicitly — a caller that ignores `error` is a silent failure (search for "SDK returns `{ error }` instead of throwing" in `web/common/engineering-principles.md`)

## Date and time
- Never use `new Date()` or `Date.now()` inside a test — this produces tests that go red on DST changeovers and at year boundaries
- Inject a `now: () => Date` parameter (or use Vitest's `vi.setSystemTime`) into every function that needs the current time; tests supply a fixed `Date` value
- After using `vi.setSystemTime`, restore with `vi.useRealTimers()` in `afterEach`
- Test DST boundaries, midnight UTC, and year-end explicitly for date-sensitive logic

## TypeScript-specific patterns

### Discriminated unions and typed errors
- When a function returns a discriminated union (`{ ok: true; data: T } | { ok: false; error: E }`), write tests for every discriminant branch — not just the happy path
- When testing thrown errors, assert the specific error class (`instanceof`), not just that something was thrown
- Use `satisfies` or `as const` in test data to get type-level verification without losing the narrowed type

### Zod / Valibot schemas
- Test `.parse()` on valid inputs: assert the output shape matches the domain type
- Test `.safeParse()` on invalid inputs: assert `success === false` and the error path is the expected one
- Test that an extra / unknown field in the input does not bleed through to the output (strip vs. passthrough behavior)
- Test the schema against a captured real-API fixture, not only synthetic data

### Environment variables
- Never read `process.env` directly inside a service function — receive config as a parameter so tests can supply test values without mutating `process.env`
- If a function must read `process.env`, wrap the mutation in `beforeEach` / `afterEach` and restore the original value

### React component units (RTL)
Unit-test a React component when it contains real logic (formatting, conditional rendering, state transitions):
- Render with `@testing-library/react`; query by role, label, or `data-testid` — never by CSS class or DOM position
- Prefer `userEvent` over `fireEvent` for realistic interaction simulation
- Assert on observable output (text content, attribute values, aria states), not on implementation (component instance properties, internal state variables)
- Test each conditional rendering branch (`isLoading`, `isError`, `isEmpty`, `hasData`) explicitly
- Do not test Server Components directly in RTL — test the plain service function they call

### Server Actions and route handlers
The handler / action itself should be humble (read request → call service → write response). Unit-test the service function, not the handler. Test the handler only when you need to verify it applies auth / validation correctly — and do that with an integration test or with direct invocation of the exported handler in a test that supplies a fake `Request`.


# Phase 4 — Review the Test Suite

Before marking tests done, verify:

**Coverage of specification:**
- Every condition of satisfaction has at least one test
- Every error / empty / loading state has a test
- Every boundary condition has a dedicated test
- Every SDK / external-API branch (`{ data }` vs `{ error }`) has a test

**Test quality:**
- No test asserts on implementation — a refactor that preserves behavior must not break a passing test
- No `expect(x).toBeTruthy()` / `expect(x).toBeDefined()` where `expect(x).toEqual(exactValue)` is possible; weak assertions pass for the wrong reasons
- No raw `any` in test data that bypasses TypeScript's type checking
- No skipped tests (`.skip`, `xit`, `xdescribe`, commented-out blocks). When a test fails unexpectedly, exactly three responses are valid: (a) fix the production code if the test exposed a real regression, (b) fix or delete the test if the behavior it was specifying intentionally changed, or (c) record the gap immediately and create a tracked follow-up if fixing is genuinely blocked. Commenting out a failing test and continuing is never acceptable — a commented-out test is a silent lie: it records a gap that nobody knows exists and that will never be fixed.
- No `setTimeout` / `sleep` calls — use `await` and structured async
- No conditional logic (`if`, `else`, `for`, `while`) in test body functions — skipped branches silently skip assertions; for sync throws, `expect(() => fn()).toThrow(ErrorType)` not bare `try/catch`

**Independence and reliability:**
- The suite passes when tests run in random order (`--random`)
- The suite passes when run alone (`--testPathPattern=this-file`)
- No shared mutable state leaks between tests (`process.env`, module-level singletons, `vi.fn()` call history)
- `vi.restoreAllMocks()` / `jest.restoreAllMocks()` is called in `afterEach` or configured globally

**F.I.R.S.T. checklist:**
- **Fast** — the whole unit suite runs in seconds, not minutes (under 30 s is a reasonable target for a medium codebase)
- **Independent** — each test owns its setup and teardown; no test assumes a prior test ran
- **Repeatable** — no dependency on a live database, external API, real system clock, or CI-specific environment variable
- **Self-Validating** — binary pass/fail; no manual log inspection to interpret a result
- **Timely** — written alongside the implementation, not weeks after

**When a test reveals a bug:** test exhaustively in that area before moving on. The same conditions that produced one defect tend to produce others nearby. Widen coverage of the surrounding function before fixing and moving on.


# Operating Principles (apply throughout)

**Tests are a specification, not an afterthought.** A test written after the fact is often a transcription of the implementation — it proves the code ran, not that it was correct. Write tests alongside the code, driven by the conditions of satisfaction.

**Test behavior, not structure.** The test knows the inputs and the expected observable outputs; it does not know — or care — how the production code achieves them internally. Structural coupling (one test file per module, one `it` per function) means every refactor cascades into mass test edits, which pressures the team to stop refactoring. A **testing API** — stable service-function signatures, validator schemas, formatter interfaces — that hides the production code's internal structure from tests allows both sides to evolve independently.

**The Fragile Tests Problem:** business rules verified only by driving the volatile UI through Playwright/Cypress break in bulk on any markup change, so the suite gets disabled rather than maintained. The fix is to verify rules through a stable behavioral seam (a plain service function, a validator, an entitlement resolver) and reserve E2E tests for genuine user journeys. When you do write E2E tests, use the **page object** (application driver) pattern — a dedicated class per page/flow that exposes domain-language methods (`loginPage.login('dave')`, `orderFlow.placeOrder({ quantity: 4, price: 10 })`) and hides all Playwright selectors and interactions. Never reference `page.click('#payButtonId')` directly from test code. Each selector lives in exactly one place inside the page object; when the UI changes, only the page object changes, not every test that touches the feature. Tests that couple directly to element IDs or CSS selectors break in bulk on any markup or label change.

**E2E tests own their data.** Each Playwright or acceptance-level test should create its own isolated state — a new user account, a fresh order, a clean session — using the app's own API or setup endpoints, then tear it down after the test. Never rely on a production data dump, a shared seed user, or state left by a previous run. Shared state forces sequential execution, produces mysterious failures when data drifts, and makes parallel runs unreliable. Where the app supports isolated entities (users, accounts, tenants), create a fresh one per test and delete it on teardown — independent entities enable parallel execution and eliminate order dependency.

**Bugs congregate.** When a test reveals a bug, widen test coverage of the surrounding area before moving on. The same conditions that produced one defect tend to produce others nearby.

**Look for patterns in failures.** When multiple tests fail, find what they share — similar input ranges, the same code path, a specific boundary value — before debugging any individual failure. The pattern names the root cause faster than examining failures in isolation.

**A suite that has never been broken is not trusted.** Occasionally break a branch or flip a condition in production code and confirm the right test goes red. A suite that stays green through any change proves nothing.

**Test the code you change.** The simplest discipline for adding coverage to an under-tested codebase is to write tests for every piece of code you modify — before you modify it (to characterize current behavior) and after (to pin the new behavior). You are not responsible for every untested path in the system; you are responsible for every path you touch. Applied consistently across many contributors, this rule turns a sparse test suite into a comprehensive one without any single heroic test-writing sprint.

**Automate a test once you've run it manually twice.** If you find yourself repeating the same manual check — same inputs, same verification — a third time, that is the signal to automate it. The first run is exploration; the second confirms the first; by the third, the scenario is stable enough that a human running it no longer adds value. Automation at this point is not gold-plating; it is the cheapest time to write the test before the behavior drifts and the test gets harder to write.

**Build test entity factories, not elaborate fixtures.** For each domain entity your tests use frequently, create a factory function (e.g., `makeUser(overrides?)`, `makeOrder(overrides?)`) that returns a valid instance with sensible defaults. Each test calls the factory and overrides only the fields relevant to its behavior. This keeps tests focused and prevents test setup from growing into a tangled fixture file. If setting up data for a unit test is hard — many fields, many dependencies — that is a design signal: the unit under test is doing too much and needs better decomposition.

**Never use production data dumps for dev or test environments.** Production datasets are too large to be useful as test data, they may contain PII, and they rot as the schema changes. Instead: for unit tests, create minimal, purpose-built inputs inline or via factory functions; for acceptance tests, seed state through the app's own API; for manual testing environments, generate a realistic-but-synthetic dataset from acceptance or capacity test runs.

**Coverage numbers are a weak signal.** A line executed by a test that only asserts "didn't throw" is effectively untested. Weight coverage by risk — payment, auth, validation, and data-mapping paths warrant high coverage; trivial glue code does not. The meaningful question is not "what percentage of lines ran?" but "which behaviors are unspecified?"
