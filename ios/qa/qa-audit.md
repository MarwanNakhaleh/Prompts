# iOS QA / Test-Coverage Audit Prompt for Claude Code

> Acting as a principal QA engineer specializing in iOS, perform a comprehensive test-coverage audit of this codebase. **Do not implement any fixes or tests** — document gaps only. The goal is to find the bugs that *will* ship to prod next, not the ones already caught by the existing suite. Reason about the platform's real failure modes: decode-shape drift at the network boundary, persistence and date coercion, concurrency races, and SwiftUI state-lifecycle bugs — not just line coverage.

Read `ios/common/engineering-principles.md` — the dependency rule, humble-shell pattern, and composition root it describes are the background for why certain coverage gaps carry higher risk (logic trapped behind a hard-to-test shell, dependencies that can't be substituted in tests, behavior verified only through the volatile UI).

Consult `ios/resources.md` as needed — it lists the authoritative testing frameworks (XCTest, Swift Testing, XCUITest), Apple security and platform docs, and OWASP sources for security-relevant test gaps.

**Framing — cover all four testing quadrants, not just the automated ones.** The canonical model lives in `shared/testing-quadrants.md`; read it first. In one line: judge coverage on a two-axis map — business- vs. technology-facing × tests that *support* building vs. *critique* the finished product — spanning **Q1** unit/component, **Q2** acceptance/story tests from concrete customer examples, **Q3** exploratory/usability/UAT driven by a thinking human, and **Q4** performance/load/security/"ilities." A suite living entirely in Q1/Q2 can be all-green and still ship the bugs that matter, so note explicitly which quadrants the existing suite neglects, and apply context-driven judgment throughout — the value of any practice depends on the product's risk profile.

---

## 0. App-Specific Context (fill this in before running)

<!-- The more you give the agent, the sharper the findings. Leave blank and it
will infer from the code, but prior incidents are the highest-signal input. -->

- **UI framework:** SwiftUI / UIKit / mixed
- **Min deployment target & Swift version:** 
- **Concurrency model:** async/await + actors / Combine / GCD / mixed; Swift 6 strict concurrency on/off
- **Persistence:** SwiftData / Core Data / Realm / SQLite / GRDB / Keychain / UserDefaults
- **Networking:** URLSession / Alamofire / custom; Codable / other decoding
- **Test frameworks present:** XCTest / Swift Testing / XCUITest / snapshot (which lib) / StoreKitTest
- **CI:** Xcode Cloud / GitHub Actions / Bitrise / fastlane; coverage gating yes/no
- **Monetization:** StoreKit 2 / StoreKit 1 / third-party; server-side receipt validation yes/no
- **Known prior production incidents (most valuable):** paste a short list; findings that would re-enable these are P0 by default

---

## 1. Test Strategy & Pyramid Shape

- Inventory every test target and classify each test: pure unit (everything mocked), integration (real persistence / real decode against fixture payloads), contract (API/SDK response shape), snapshot (UI), UI automation (XCUITest)
- Flag suites that are unit-heavy with no integration tier — these can't catch the bugs that matter on iOS: JSON-decode-shape drift, Core Data/SwiftData fetch + type coercion, actor/threading behavior
- Look for "the test passes, the real call fails" risk: tests that mock `URLSession`/the repository so completely they assert nothing about real decoding or real persistence
- Check whether code coverage is collected in the scheme and actually enforced as a gate in CI (not just displayed). Look for whether the team uses **ratcheting** — gating on "metric got worse since last build" rather than only an absolute floor. A ratchet prevents gradual regression even when the absolute threshold is comfortably met; an absolute floor can be gamed by adding code that doesn't lower the number. Treat the percentage itself as a weak signal, not a quality measure: coverage proves a line *executed*, not that an assertion pinned its behavior or that the failing path was exercised. A line run by a test that only asserts "returned some value" is effectively untested. Flag a high coverage number cited as a quality claim, check whether the suite's strength is ever validated by deliberate fault injection (break a function / flip a branch and confirm a test goes red) rather than assumed, and weight coverage expectations by risk — higher for payment, auth, decode, and persistence paths than for trivial glue
- Identify untested critical paths: screens or flows that ship with NO coverage at any tier
- **Push tests to the lowest level that can hold them.** For each behavior covered only by a slow UI/integration test, ask whether it could be asserted at the unit or component level instead — lower-level tests are faster, more isolated, and pinpoint failures. Flag inverted pyramids (UI-heavy, unit-light) where a single failure can't be localized, and flag the opposite trap: pure unit suites that mock the boundary so completely they assert nothing about real decode/persistence behavior.
- **Audit for a stable testing seam below the UI.** Business rules should be verifiable through a framework-free API — view-model inputs, use-case methods, or plain Swift functions — without requiring a simulator, live server, or real store. Tests that can only reach business logic by driving the full XCUITest layer must be fragile: any navigation or layout change breaks them, and the team learns to fear UI refactoring. Audit whether each critical rule (auth logic, entitlement decisions, data transformations, error-path branching) is accessible through such a seam. If not, the missing seam — not the missing UI test — is the architectural gap. Flag it as requiring an "Extract to plain type" move that converts an XCUITest-only behavior into a fast, stable unit test. A testing seam that allows you to bypass the UI, inject test state, and exercise business rules directly is the highest-leverage architectural investment a low-coverage codebase can make.
- **Find decision logic trapped behind a hard-to-test shell.** When a behavior is only reachable through a slow XCUITest/integration test, the cause is often that real logic — branching, formatting, state derivation, entitlement decisions — is fused into a hard-to-test boundary (a SwiftUI `View`/`body`, a view controller, or a networking/persistence class) that can't be exercised without a simulator, a live server, or a real store. The structural fix that makes it unit-testable is to keep that boundary *humble* (the `View` only renders values handed to it; the store/client class only runs the query or request) and pull the decision logic into a plain, framework-free type — a view model, a presenter, or a pure function — that a test calls directly. Audit for humble shells that still carry untested work: a `View` computing formatted strings / deciding visibility in its body, a controller deriving entitlement inline, a store wrapper branching on results. That buried logic, not the missing UI test, is the real gap; the proposed fix is "extract to a plain type and unit-test it there." A unit that needs half the app stood up to test it usually signals a missing boundary (or a dependency cycle) — flag that too.
- **Automate by risk, not by reflex.** Not every check earns automation. Look-and-feel, usability, one-off validations, and behavior that realistically can never regress are often cheaper to verify once by hand than to maintain forever; the highest-ROI investment belongs in the unit/component base of the pyramid. Treat XCUITest/UI tests as the fragile, high-maintenance tip — keep them few, and where lower tiers already cover a behavior, question whether a parallel UI case still earns its keep. Flag suites that automate trivia while leaving risky paths uncovered, and conversely flag manual scripted regression re-run every release that should have been automated.
- **Fast feedback is itself a coverage property.** When the build + test run grows too long, check-ins stack up and the team stops trusting the signal. Flag a slow CI loop: profile the bottleneck, push behavior down to faster unit tests, parallelize across simulators/machines, and move genuinely costly suites (full integration, device-farm runs) to a scheduled run. A fast green build is the highest-ROI automation a team has.

## 2. Decode-Boundary & Domain Contract Tests

- For every `Codable` model, look for a test that decodes a representative real backend payload (a captured fixture), not a round-trip of an encoder the test itself wrote. Round-trip tests prove the model agrees with itself, not with the server
- Audit optionality and key drift: tests that catch a previously-required field going missing/null, a renamed key, or a `keyDecodingStrategy` mismatch (snake_case vs camelCase). A non-optional property meeting `null` throws and can blank an entire screen — flag models with no such test as P0
- For every enum decoded from the wire, look for **unknown-case** coverage: does the app handle a server value it's never seen (`@unknown default`, an `.unknown` case, or a custom decoder) instead of throwing or crashing? Missing unknown-case handling is the iOS analog of allowlist drift and is P0
- Check `Date` decoding strategy tests (ISO8601 vs epoch vs custom) and that timezone handling is asserted, not assumed
- For any third-party SDK with a typed response, look for a contract test pinning its shape so an SDK upgrade fails CI rather than surfacing in crash logs

## 3. Persistence-Coupled Code

- Find every type that reads/writes Core Data, SwiftData, Realm, SQLite, Keychain, or UserDefaults. For each, check whether tests exercise a **real store** (in-memory Core Data / SwiftData container, real Keychain test, real SQLite) or only a hand-rolled mock that can't reproduce coercion and fetch behavior
- Flag tests that fabricate model instances in memory and never round-trip them through the store — these mask migration, type-coercion, and constraint bugs
- Audit migrations explicitly: Core Data lightweight/heavyweight migration tests, SwiftData schema-version migration tests. An unmigrated store on upgrade is a launch-crash class — flag missing migration tests as P0. Test migrations against a realistically *large* store, not a handful of fabricated rows: a migration that completes instantly on dev data can take minutes on a power user's accumulated store and blow the app-launch watchdog into a launch-time termination. Missing large-store migration-timing coverage is itself a P0
- Check threading on the persistence layer: Core Data context confinement (main vs background), `@MainActor` model access, SwiftData `ModelActor` usage — and whether any test asserts cross-context/cross-actor correctness
- Look for Keychain accessibility/migration coverage: tokens surviving (or correctly not surviving) OS upgrade, device restore, and locked-device access

## 4. Concurrency & Lifecycle

- Find every detached/unstructured `Task { ... }` started from a view or controller. Look for "Task cancelled when the view disappears, work never completes" coverage — this is the iOS analog of the fire-and-forget Promise and is a top recurring class. Flag missing coverage as P0
- Audit Combine pipelines for retained `cancellables` and for subscriptions torn down before completion; look for tests asserting completion/cancellation behavior
- Check for data-race coverage under Swift 6 strict concurrency: shared mutable state touched off the expected actor. If strict concurrency is off, flag that as a latent gap
- Look for **main-thread UI mutation** coverage: a background-thread update to UI/`@Published`/observable state. These produce runtime purple warnings and intermittent corruption that snapshot/unit tests miss
- Audit retain-cycle risk in escaping closures (`self` captured strongly in network/Task callbacks) — note where a leak test or `weak self` convention is absent on long-lived objects
- Check ordering/race coverage on multi-step async flows (token refresh during in-flight requests, concurrent writes)

## 5. SwiftUI / UIKit State & Rendering

- Audit `@StateObject` vs `@ObservedObject` usage: an observable created as `@ObservedObject` is recreated on every parent re-render, losing state and re-triggering work. Look for tests/snapshots that would catch this lifecycle bug
- Check `@State`/`@Binding`/`@Environment` propagation across steps: when a value changes upstream, do dependent views and side effects observe the new value? Look for cross-step state-propagation coverage
- For any multi-step flow, look for a test that a state change in step N is visible to step N+1 (the iOS analog of "milestone date change → next cron sees it")
- Audit navigation state: deep-link routing, `NavigationStack` path restoration, state restoration after backgrounding/termination
- Check feature-gating UI tests: free vs paid visibility, disabled/enabled controls, premium-required screens
- Look for snapshot coverage across the axes that actually break layouts: Dynamic Type (largest sizes), dark mode, RTL, smallest and largest device classes, landscape

## 6. Networking, Error Surfacing & Offline

- Audit every networking error path: does the layer surface a structured, distinguishable error (status code + decoded server error body) or collapse everything into a generic "Something went wrong"? Generic catch-alls hide root cause for weeks — flag every occurrence
- Check coverage for the failure matrix: timeout, no connectivity, 4xx with error body, 5xx, decode failure, empty/partial payload. Each should be a distinct, tested branch
- Look for offline-mode and flaky-network coverage: cached-data fallback, queued writes, retry-with-backoff (and that retries are bounded)
- Check graceful-degradation coverage for the *slow* dependency, not just the failed one: when a screen fans out to several endpoints and one responds slowly or hangs, does that section degrade independently (its own timeout, a per-section error/retry affordance) or does it block the whole screen behind one spinner? A slow backend is more dangerous than a down one — it's the case happy-path tests miss. Flag screens with no per-section timeout/degradation coverage
- Audit that the UI renders the real error, not a hardcoded string that masks the server's actual message
- Check that logging on error paths includes enough structured detail (status, endpoint, decoded error) to diagnose without a repro

## 7. StoreKit / Subscriptions & Entitlements

- Find every premium-grant path: StoreKit purchase, restore purchases, server-granted entitlement, promo/offer codes. Each should have a fixture + a happy-path test (use StoreKitTest where applicable)
- Audit entitlement resolution: do all the places that decide "is this user paid" (local transaction state, server entitlement, cached flag) converge on the same answer for the same state? Divergent gating logic is a recurring class
- Check transaction-finishing coverage: an unfinished `Transaction` re-prompts or re-delivers. Look for "transaction verified → entitlement granted → transaction finished" end-to-end coverage
- Look for revocation/expiry/refund and "subscription lapsed" coverage parallel to the grant paths
- Audit receipt/transaction verification: is it validated (ideally server-side) and resistant to a tampered client, with a test asserting an invalid transaction is rejected?

## 8. Background Work & Notifications

- Inventory background execution: `BGTaskScheduler` tasks, background URLSession, background fetch, silent push handlers. Each should have a happy-path test and a degraded-mode test (downstream failure handled, task expiration handled)
- Check that background-task expiration handlers are tested — work that runs past its window is killed mid-flight
- Audit push/local notification handling: payload parsing (a decode-boundary case — malformed/unknown payloads), tap-routing into the right screen, scheduling correctness, and de-duplication on retry
- Look for "duplicate notification / duplicate work on retry" coverage where a side effect runs before the awaited completion

## 9. Auth, Session & Token Edge Cases

- Find tests for token storage and refresh: token in Keychain not UserDefaults, refresh-token rotation, and the **concurrent-refresh race** (multiple in-flight 401s triggering parallel refreshes)
- Audit the "session present but user id / profile missing" path and the "token deleted/expired while app foregrounded" path
- Check biometric-gate coverage: success, failure, fallback to passcode, and that a failed/bypassed biometric does not grant access (back the gate with a Keychain ACL, not a boolean)
- Look for "logged-out state cleared completely" coverage — stale cached PII or tokens surviving logout is a leak class

## 10. Device / OS / Locale Edge Cases

- Adversarial date/time tests: DST rollover, midnight UTC vs local, year boundaries, far-past/far-future dates, 12h/24h locale formatting, non-Gregorian calendars
- Locale and formatting: number/currency/date formatting across locales; RTL layout; pluralization
- Lifecycle: backgrounding mid-operation, termination + state restoration, low-memory warnings, app launched cold from a notification/deep link
- OS-version branches: any `if #available` / availability-gated code path should have coverage on both branches
- Permission states: denied/restricted/not-determined for camera, location, notifications, photos — and graceful behavior in each

## 11. Exploratory, Scenario & Product-Critique Coverage (the bugs scripts miss)

- The existing suite almost certainly lives in Q1/Q2 (tests that *support* the team). Audit whether anyone *critiques* the product: is there a charter-driven, time-boxed **exploratory testing** practice (session-based test management — a mission, a time box, notes that make findings reproducible), or does testing stop at scripted assertions? Exploratory testing is *simultaneous test design, execution, and learning* — not ad-hoc clicking — and it's where the most serious iOS bugs (state corruption, navigation dead-ends, gesture/lifecycle interactions) actually surface. Flag the absence as a P1 process gap.
- Check for **scenario / "soap opera" coverage** of realistic, exaggerated multi-step journeys (backgrounding mid-purchase, deep-link into a half-restored `NavigationStack`, rotate during an in-flight async load, lose connectivity between two writes) rather than isolated per-screen tests.
- **Persona coverage:** are adversarial and edge personas exercised — the user on a jailbroken/slow device, the double-tapper, the offline-then-online user, the VoiceOver-only user, the largest-Dynamic-Type user? Note personas with no representation in the test thinking.
- **Feedback loop:** confirm exploratory findings are converted into automated regression tests — a bug found by hand should become a unit/integration test so it can't silently return. A team that explores but never captures is paying for the same bug twice.
- Note where exploratory testing is the *right* tool and automation is the wrong one (usability, look-and-feel, one-off investigations) and is simply missing.
- (Search the web for "session-based test management," "exploratory testing charters," and "soap opera testing" to expand these techniques.)

## 12. Non-Functional / "ility" Coverage

- Use an explicit "ility" checklist so the team consciously decides which qualities matter and how important each is — don't let nonfunctional concerns default to "the developers will handle it." Cover at least:
  - **Performance:** launch time, scroll/interaction latency, memory under realistic data volumes. Is there a **baseline** captured so regressions are detectable, with *measurable* targets (e.g., "cold launch < 1.5s", "list scrolls at 60fps with 5k rows") rather than "should be fast"? Confirm perf is measured on a production-representative device class (and that any result extrapolated from a faster simulator/device says so explicitly), and that perf tests are re-run when features likely to move the numbers land (complex queries, large fetches, image-heavy screens) — not deferred to the end game. Watch allocations over a sustained soak for leak/retain growth, not just a single snapshot. Flag missing baselines.
  - **Reliability:** run the automated suites repeatedly / over time to surface leaks and intermittent failures (the iOS analog of a soak test); assert against stated SLAs/crash-free-session targets where they exist.
  - **Compatibility:** the supported device-class / OS-version matrix — is each `if #available` branch and each min/max device actually exercised?
  - **Install / upgrade:** is the *real upgrade path* tested (old persisted store → migration → launch), not just a clean install? Unmigrated-store-on-upgrade is a launch-crash class.
  - **Accessibility & localization** as qualities to verify, not checkboxes (VoiceOver flows, RTL, pluralization, non-Gregorian calendars).
- Performance/security/"ility" tests are listed fourth but should not be done last — flag anywhere they're deferred to the end game when redesign is no longer affordable.

## 13. Test Hygiene & Anti-Patterns

- Flag XCUITest flakiness: `sleep()` / fixed `Thread.sleep` instead of `XCTestExpectation`/`waitForExistence`, and any wait without a sensible timeout
- Look for tests that simulate failure in a way the real boundary never produces (e.g., throwing where the SDK actually returns a typed error value, or vice versa)
- Audit force-unwraps (`!`) and `try!` inside tests — these convert a meaningful assertion failure into an opaque crash and can mask the real nil/throw
- Check for tests depending on `Date()` / real clock without injecting a clock or controlling time — these go red on DST days and at year boundaries
- Find skipped/disabled tests (`XCTSkip`, disabled in the scheme, commented-out) and `// TODO: fix` markers — quantify the latent gap they represent
- Flag weak assertions (`XCTAssertNotNil`, `XCTAssertTrue(x != nil)`) where an exact `XCTAssertEqual` is possible
- Audit async test correctness: missing `await` on expectations, expectations that can pass by timing out, `@MainActor` isolation gaps in tests
- Flag inter-dependent tests that must run in a fixed order or share mutable state (a leaked Keychain entry, a populated shared store, `UserDefaults` not reset). Each test should set up and tear down its own state and pass in isolation and in any order; rely on a fresh in-memory store / reset fixtures per test, not an accreting shared one
- Flag omnibus tests asserting several unrelated behaviors at once — one condition per test means a failure pinpoints the cause instead of just saying "something broke"
- Flag **structural coupling** between the test suite and the production shape: a test class mirroring every production class one-to-one (a test method per method) is coupled to *structure*, not behavior, so any refactor cascades into mass test edits — which pressures the team to stop refactoring. The deeper form is the **Fragile Tests Problem**: business rules verified only by driving the volatile UI (XCUITest through login → navigation → screen) break in bulk on any UI change, so the suite gets disabled rather than maintained. The fix is to verify rules through a stable behavioral seam below the UI (a view model / use-case API) and reserve XCUITest for genuine end-to-end journeys. Treat tests as part of the system's design — coupled-to-structure tests make the production code rigid, which is a coverage risk in its own right.
- Note the natural divergence between tests and production code over time: tests tend to become more concrete and specific as the team learns what can go wrong; production code tends to become more abstract and general as it is refactored. Structural coupling blocks this healthy divergence — a test that calls concrete implementation types directly must be updated every time those types are refactored, even when behavior is unchanged. A **testing API** or stable behavioral seam (protocol-based, value-type view model inputs, use-case method signatures) that hides the production code's internal structure from tests is what allows both sides to evolve independently: production types can be renamed, extracted, or reorganized without touching a passing test.
- Audit XCUITest selector strategy and structure: tests bound to on-screen text or system-generated identifiers break when copy or view hierarchy shifts. Prefer stable accessibility identifiers, and a layered structure (driver → screen-object → test-data) so UI churn is absorbed in one place rather than cascading across the suite. Organize tests by the behavior's *intent*, which rarely changes, not the UI's current *implementation*, which changes constantly. (Search the web for "page object pattern" / "screen object" to expand this.)
- **Test boundary conditions explicitly.** Algorithms most commonly fail at the edges — the transition from zero to one, empty to non-empty, minimum to maximum, first to last. Don't assume a test that passes in the middle implicitly covers the boundaries. Enumerate the boundary cases for every behavior and write dedicated tests for each edge. Boundary bugs are the most common class of correctness failure and the ones most often left undetected by happy-path tests.
- **When you find a bug, test exhaustively in that area — bugs congregate.** A single defect discovered in a function is rarely alone; the same conditions that produced one tend to produce others in the same neighborhood. When a bug is confirmed, write a thorough suite of tests for the entire function and its nearest neighbors before moving on. Flag test suites that added one regression test per reported bug but never widened coverage of the surrounding code — the next bug is likely adjacent.
- **Look for patterns in test failures — they reveal root causes.** When multiple tests fail, examine what those failures share in common (similar inputs, the same code path, a specific parameter range) before debugging any single case. A cluster of failures on inputs larger than N, or on the last item in a collection, or under a specific locale — that pattern names the root cause faster than examining failures in isolation. Organize test cases so patterns in failures are visible; a sorted or grouped failure list often makes the root cause obvious.
- Apply the **F.I.R.S.T.** checklist to the suite as a whole: **Fast** — tests that are slow discourage frequent running; flag any suite where the full loop exceeds a few minutes, and identify the slowest tests to push down to a cheaper tier or run on a schedule; **Independent** — already covered above (inter-dependent tests, shared state); **Repeatable** in any environment — no dependency on a live network, a real system clock, or shared device state; **Self-Validating** — tests must produce a binary pass/fail without manual log inspection or file comparison (a test that "passes" by timing out or requires reading output to interpret is not self-validating); **Timely** — tests written alongside the code they cover, not after; a large body of production code with no accompanying test history signals code that was never designed for testability, which manifests as logic trapped behind hard-to-test shells and missing seams.
- **Validate suite strength with mutation testing**, not just coverage numbers. Mutation testing tools automatically introduce small plausible code changes — flip a `>` to `>=`, change `+` to `-`, delete a `return` statement — and check whether the test suite detects each change. A line that executes in a test proves only that it ran; a mutation that survives means no assertion would fail if that code changed, exposing tests that verify execution rather than correctness. For Swift, `Muter` provides mutation testing integration. Apply it to high-risk areas first — auth, payment flows, validation logic, decode boundaries — rather than the whole codebase; the signal is sharpest where defects are most costly. When a surviving mutation is found, add a test that catches it before fixing the mutation; this turns the audit finding directly into a regression test.

---

## Output Format

Organize all findings into a markdown report grouped by the categories above. For each finding include:

| Field | Description |
|-------|-------------|
| **Priority** | P0 / P1 / P2 / P3 (P0 = enables a known-shape production bug class) |
| **File or Surface** | Exact file path / type / view / target, with line numbers where applicable |
| **Gap** | What is not covered today |
| **Bug-class enabled** | The production failure this gap allows to ship undetected (e.g., "decode-shape drift", "unknown-enum-case crash", "detached-Task cancellation", "StateObject lifecycle reset", "main-thread UI mutation") |
| **Proposed test shape** | One sentence describing the test that closes the gap (unit / integration / contract / snapshot / UI). Do not write the test |
| **Estimated effort** | Half-day / 1 day / 2 days / >2 days |

Begin the report with an executive summary showing a count of gaps per priority level, plus a one-paragraph "what the next production bug looks like if we ship nothing" prediction grounded in the most pressing P0/P1 findings.

---

## Reference: iOS bug classes to use as a checklist

Every category above maps to at least one of these. Note explicitly when a gap would re-enable one — and prepend any incidents from the App-Specific Context block, which are P0 by default:

- **Detached-Task cancellation:** unstructured `Task` tied to a view's lifetime is cancelled on dismissal; awaited work never completes (direct analog of fire-and-forget Promise)
- **Side-effect-before-await race:** state stamped / row written / notification recorded before the awaited async call, which then fails, leaving inconsistent state never retried
- **Decode-shape drift:** backend renames a key, drops a field, or sends null for a non-optional; `Codable` throws and the screen blanks or the error is swallowed
- **Unknown enum case:** server sends an enum value the app predates; decode throws or the app misclassifies/crashes (iOS allowlist drift)
- **Date / timezone coercion:** wrong `Date` decoding strategy or timezone assumption shows the wrong day, misfires reminders, or breaks on DST
- **`@StateObject` vs `@ObservedObject` misuse:** observable recreated on parent re-render, losing state or re-running work
- **Main-thread UI mutation:** UI/observable state mutated off the main actor; intermittent corruption and runtime warnings
- **Retain cycle:** `self` captured strongly in an escaping closure on a long-lived object; leak and stale-state behavior
- **Force-unwrap crash:** `!` / `try!` on unexpectedly-nil or throwing values, often on an unhappy decode/persistence path
- **Unfinished StoreKit transaction:** transaction never finished; user re-prompted or entitlement state diverges across resolution sites
- **Keychain accessibility/migration loss:** wrong accessibility class or missing migration drops tokens on OS upgrade/restore
- **Concurrent token-refresh race:** parallel 401s trigger multiple refreshes; one wins, others 401 silently
- **Slow-dependency stall:** a request with no (or too-long) timeout hangs the UI or blocks a whole screen behind one section; a slow backend freezes the app where a failed one would surface an error

Findings that enable any of these to recur are P0 by default.
