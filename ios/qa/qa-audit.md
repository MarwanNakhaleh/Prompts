# iOS QA / Test-Coverage Audit Prompt for Claude Code

> Acting as a principal QA engineer specializing in iOS, perform a comprehensive test-coverage audit of this codebase. **Do not implement any fixes or tests** — document gaps only. The goal is to find the bugs that *will* ship to prod next, not the ones already caught by the existing suite. Reason about the platform's real failure modes: decode-shape drift at the network boundary, persistence and date coercion, concurrency races, and SwiftUI state-lifecycle bugs — not just line coverage.

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
- Check whether code coverage is collected in the scheme and actually enforced as a gate in CI (not just displayed)
- Identify untested critical paths: screens or flows that ship with NO coverage at any tier

## 2. Decode-Boundary & Domain Contract Tests

- For every `Codable` model, look for a test that decodes a representative real backend payload (a captured fixture), not a round-trip of an encoder the test itself wrote. Round-trip tests prove the model agrees with itself, not with the server
- Audit optionality and key drift: tests that catch a previously-required field going missing/null, a renamed key, or a `keyDecodingStrategy` mismatch (snake_case vs camelCase). A non-optional property meeting `null` throws and can blank an entire screen — flag models with no such test as P0
- For every enum decoded from the wire, look for **unknown-case** coverage: does the app handle a server value it's never seen (`@unknown default`, an `.unknown` case, or a custom decoder) instead of throwing or crashing? Missing unknown-case handling is the iOS analog of allowlist drift and is P0
- Check `Date` decoding strategy tests (ISO8601 vs epoch vs custom) and that timezone handling is asserted, not assumed
- For any third-party SDK with a typed response, look for a contract test pinning its shape so an SDK upgrade fails CI rather than surfacing in crash logs

## 3. Persistence-Coupled Code

- Find every type that reads/writes Core Data, SwiftData, Realm, SQLite, Keychain, or UserDefaults. For each, check whether tests exercise a **real store** (in-memory Core Data / SwiftData container, real Keychain test, real SQLite) or only a hand-rolled mock that can't reproduce coercion and fetch behavior
- Flag tests that fabricate model instances in memory and never round-trip them through the store — these mask migration, type-coercion, and constraint bugs
- Audit migrations explicitly: Core Data lightweight/heavyweight migration tests, SwiftData schema-version migration tests. An unmigrated store on upgrade is a launch-crash class — flag missing migration tests as P0
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

## 11. Test Hygiene & Anti-Patterns

- Flag XCUITest flakiness: `sleep()` / fixed `Thread.sleep` instead of `XCTestExpectation`/`waitForExistence`, and any wait without a sensible timeout
- Look for tests that simulate failure in a way the real boundary never produces (e.g., throwing where the SDK actually returns a typed error value, or vice versa)
- Audit force-unwraps (`!`) and `try!` inside tests — these convert a meaningful assertion failure into an opaque crash and can mask the real nil/throw
- Check for tests depending on `Date()` / real clock without injecting a clock or controlling time — these go red on DST days and at year boundaries
- Find skipped/disabled tests (`XCTSkip`, disabled in the scheme, commented-out) and `// TODO: fix` markers — quantify the latent gap they represent
- Flag weak assertions (`XCTAssertNotNil`, `XCTAssertTrue(x != nil)`) where an exact `XCTAssertEqual` is possible
- Audit async test correctness: missing `await` on expectations, expectations that can pass by timing out, `@MainActor` isolation gaps in tests

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

Findings that enable any of these to recur are P0 by default.
