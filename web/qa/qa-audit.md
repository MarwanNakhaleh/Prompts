# Next.js QA / Test-Coverage Audit Prompt for Claude Code

> Acting as a principal QA engineer specializing in Next.js and full-stack TypeScript, perform a comprehensive test-coverage audit of this codebase. **Do not implement any fixes or tests** — document gaps only. The goal is to find the bugs that *will* ship to prod next, not the ones already caught by the existing suite. Reason about real failure modes — schema-vs-type drift, ORM/DB type coercion, SDK response-shape changes, Server/Client component boundary bugs, and counter↔state divergence — not just line coverage.

Read `web/common/engineering-principles.md` — the dependency rule, humble-shell pattern, and composition root it describes are the background for why certain coverage gaps carry higher risk (logic trapped behind a hard-to-test boundary, dependencies that can't be substituted in tests, business rules verified only through the volatile UI or E2E layer).

**Framing — cover all four testing quadrants, not just the automated ones.** The canonical model lives in `shared/testing-quadrants.md`; read it first. In one line: judge coverage on a two-axis map — business- vs. technology-facing × tests that *support* building vs. *critique* the finished product — spanning **Q1** unit/component, **Q2** acceptance/E2E from concrete user examples, **Q3** exploratory/usability/UAT driven by a thinking human, and **Q4** performance/load/security/"ilities." A suite living entirely in Q1/Q2 can be all-green and still ship the bugs that matter, so note explicitly which quadrants the existing suite neglects, and apply context-driven judgment throughout — the value of any practice depends on the product's risk profile.

---

## 0. App-Specific Context (fill this in before running)

<!-- The more you give the agent, the sharper the findings. Prior incidents are
the highest-signal input — paste them even if rough. -->

- **Next.js version & router:** App Router / Pages Router / mixed
- **Rendering mix:** Server Components, Server Actions, route handlers, ISR/cached fetches, client-heavy?
- **Data layer:** Prisma / Drizzle / Kysely / raw SQL / other; database engine
- **Third-party SDKs at the boundary:** payments, email, auth, storage, AI, maps, etc.
- **Auth:** NextAuth/Auth.js / Clerk / custom JWT / session strategy
- **Background work:** cron (which scheduler), queues, webhooks
- **Test frameworks present:** Vitest/Jest, RTL, Playwright/Cypress, contract/type-level
- **Hosting:** Vercel / containers / self-hosted (multi-instance?) — relevant for rate-limit and statefulness assumptions
- **CI:** coverage gating yes/no; `tsc --noEmit` gated yes/no
- **Known prior production incidents (most valuable):** paste a list; findings that re-enable these are P0 by default

---

## 1. Test Strategy & Pyramid Shape

- Inventory every test file and classify it: pure unit (mock everything), integration (real DB / real downstream), contract (third-party SDK shape), component (RTL), end-to-end (Playwright/Cypress)
- Flag suites that are unit-heavy without an integration tier — these can't catch coercion bugs (`Date` vs `string` from the DB driver, `{ data, error }` vs throw from SDKs)
- Look for "the test passes, the prod call fails" risk: tests that mock the boundary so completely they prove nothing about real behavior
- Check coverage thresholds in the test config and whether they're actually enforced in CI — but treat the percentage as a weak signal, not a quality measure: coverage proves a line *executed*, not that an assertion pinned its behavior or that the failing path was exercised. A line run by a test that only asserts "didn't throw" is effectively untested. Flag a high coverage number cited as a quality claim, check whether the suite's strength is ever validated by deliberate fault injection (break a function / flip a branch and confirm a test goes red) rather than assumed, and weight coverage expectations by risk — higher for payment, auth, and boundary/decode paths than for trivial glue
- Identify untested critical paths: features that ship with NO test coverage at any tier
- **Push tests to the lowest level that can hold them.** For each behavior covered only by a slow E2E/integration test, ask whether it could be asserted at the unit or component level instead — lower-level tests are faster, more isolated, and pinpoint failures. Flag inverted pyramids (E2E-heavy, unit-light) where one failure can't be localized, and the opposite trap: pure unit suites that mock the boundary (DB, SDK, `fetch`) so completely they assert nothing about real behavior.
- **Find decision logic trapped behind a hard-to-test shell.** When a behavior is only reachable through a slow E2E/integration test, the cause is often that real logic — branching, formatting, validation, entitlement decisions — is fused into a thin framework boundary (a route handler, Server Action, middleware, or Client Component) that can't be exercised without standing up the web server, DB, or browser. The structural fix that makes it unit-testable is to keep that boundary *humble* (it only moves data: read the request, call one plain function, write the response/JSX) and pull the decision logic into a framework-free unit that a test calls directly. Audit for humble shells that still carry untested branching/formatting — a route handler computing entitlement inline, a component formatting currency/dates or deciding what to show in its own body, a presenter-less view doing real work. That buried logic, not the missing E2E, is the real gap; the proposed fix is "extract to a pure unit and test it there." A unit that needs half the system spun up to test it usually signals a missing boundary (or a dependency cycle) — flag that too.
- **Automate by risk, not by reflex.** Not every check earns automation. Look-and-feel, usability, one-off validations, and behavior that realistically can never regress are often cheaper to verify once by hand than to maintain forever; the highest-ROI investment belongs in the unit/component base of the pyramid. Treat E2E/UI tests as the fragile, high-maintenance tip — keep them few, and where lower tiers already cover a behavior, question whether a parallel E2E case still earns its keep. Flag suites that automate trivia while leaving risky paths uncovered, and conversely flag manual scripted regression that is re-run every release and should have been automated.
- **Fast feedback is itself a coverage property.** When the build + test run stretches past ~10 minutes, check-ins stack up and developers stop trusting the signal. Flag a slow CI loop: profile the bottleneck (real-DB access and through-the-UI tests are the usual culprits), push behavior to lower tiers or an in-memory store, parallelize, and move genuinely costly suites to a scheduled/nightly run. A fast green build is the highest-ROI automation a team has.

## 2. Domain-Boundary Contract Tests

- For every TypeScript union/enum with a SQL `CHECK`/`ENUM` (or schema-validator) equivalent, look for a contract test keeping the two in sync. Missing ones are P0 — they enable silent allowlist-drift: every INSERT for a new value fails the constraint until a user reports it
- For every third-party SDK, look for a contract test asserting the SDK's response shape at the boundary. SDK upgrades that change return shape (e.g., throwing → returning `{ data, error }`) should fail CI, not surface in prod logs
- Check that `tsc --noEmit` is gated in CI/pre-commit — many contract tests are type-level assertions that never run at test time
- Audit migrations: does adding a new enum/type value require a CI-enforced paired migration?
- For any runtime validator (Zod/Valibot/etc.) at the request or response boundary, check there's a test that an invalid/unexpected payload is rejected rather than silently coerced

## 3. Database-Coupled Code

- Find every call site of the data layer (ORM queries, raw `query()`/`transaction()`). For each, check whether tests exercise real driver type coercions (timestamp → `Date`, numeric/decimal → `string`, JSON columns → `unknown`) or only hand-rolled mock returns
- Flag any test constructing a mock row with the wrong type for a date/timestamp/numeric column — these silently mask coercion bugs that fire in prod
- Look for SQL/schema constraints (CHECK, UNIQUE, NOT NULL, FK) with no test asserting the code's INSERT shape satisfies them
- Audit JSON-column handling: are the merge/patch/default patterns tested end-to-end against a real store, or only at call-shape?
- Check for missing transaction-rollback coverage on multi-statement mutations
- Audit schema migrations against a realistically *large* dataset, not a handful of seed rows: a migration that runs in seconds on dev data can take hours on a production-sized table, and that duration *is* the real deploy-downtime estimate. Missing large-data migration-timing coverage hides a release-blocking outage — flag it
- Check the provenance and freshness of the test data itself: production data copied into fixtures/seeds must be scrubbed of PII (see the security-audit prompt), and stale seed data that no longer matches current production shapes produces green tests that prove nothing. Flag suites whose confidence rests on data that is neither representative nor refreshed
- Audit **environment schema/config parity**: code is tested against a dev/CI database whose schema can silently drift from production. A `CHECK`/`NOT NULL`/`UNIQUE`/FK constraint that exists in prod but is missing from the test schema lets code that violates it pass every test and fail only on the real production INSERT; leftover columns, differing column order, or absent triggers/indices hide other mismatches the same way. Confirm that test/staging/prod schemas are reconciled from one source of truth, and that the *same* migration script validated in CI/staging is the exact one applied to production (not a hand-edited variant). Green tests prove nothing when the schema under test isn't the schema in production — treat unverified schema parity as a P0 release-risk class

## 4. Server/Client Boundary, Server Actions & Caching

- Audit Server Components that pass data to Client Components: is anything non-serializable crossing the boundary, and is there coverage of the serialized shape the client actually receives?
- For every Server Action, look for tests covering auth/authorization inside the action (actions are public endpoints), input validation, and the revalidation it triggers (`revalidatePath`/`revalidateTag`)
- Check caching correctness: stale-after-mutation bugs where a write isn't followed by the right revalidation, and ISR/`fetch` cache assumptions that no test verifies. Watch for **cache-poisoning / permanent-stale** risk too — a wrong or far-future `Cache-Control`/`Expires`/`revalidate` (or a bug that skips setting it) can pin bad content in a CDN or browser cache that no redeploy can flush, forcing a URL change to recover. Flag mutating responses with no test pinning correct, bounded cache directives, and note where stacking many cache layers makes the served freshness impossible to reason about
- Audit `middleware.ts`: auth gating, redirects, and matcher coverage — does a test confirm protected routes are actually gated and matchers cover them?
- Look for hydration-mismatch risk: server/client render divergence (time, locale, random, `window`-dependent) with no coverage

## 5. Background Jobs, Webhooks & Scheduled Work

- Inventory every webhook endpoint and every scheduled/cron route. Each should have: a signature/auth-gate test, a happy-path test, and a degraded-mode test (downstream failure handled gracefully)
- For webhooks, check signature-verification coverage and **idempotency/replay** coverage — the same event delivered twice must not double-apply
- Audit "the sweep/job runs to completion even when one item throws" — per-item try/catch coverage
- Check that response counters are verified against post-state DB rows, not just the returned object — the two can drift
- Look for "side effect recorded before the awaited downstream call" coverage — the stamp-before-send / write-before-confirm race that leaves work un-retried
- Look for "fire-and-forget promise" risk: detached promises on serverless where the request context is torn down before resolution and work never completes

## 6. Payments / Subscriptions / Entitlement Gating

- Find every premium-grant path (checkout, trial, manual/comp grant). Each should have a fixture + happy-path E2E
- Audit every place that resolves "is this user entitled" (server guard, client hook, session/JWT callback, DB lookup) — they should converge on the same decision for the same state. Divergent gating logic is a recurring class
- Look for "session/JWT says X but DB says Y" race coverage — is the verified source respected, are concurrent paid actions safe during a refresh window?
- Check that revocation paths (cancel, refund, expiry sweep) have tests parallel to the grant paths
- Audit safety guards on automated revocation (e.g., a real paid subscription is never wiped by a trial-expiry sweep)

## 7. Error Surfacing & Observability

- Audit every route handler / Server Action catch block: does it return a structured error (`code` + descriptive message) or a generic catch-all ("Failed to X")? Generic messages hide root cause for weeks — flag every occurrence
- Check that error logs include enough structured detail (DB constraint names, status, IDs where safe) to diagnose without a repro — and aren't truncated past usefulness
- Look for "stable log message" conventions used as alert anchors; renaming the string without updating the alert silently breaks paging
- Audit client-side error rendering: does the UI surface the server's error body, or hardcode a generic string that hides the real cause?

## 8. Auth, Session & Auth-State Edge Cases

- Find tests for the session/JWT resolution path: DB returns a user vs. doesn't, the subscription/role query throws, the user SELECT fails
- Audit `requireAuth`/`getCurrentUser`-style helpers — do they cover "session present but `user.id` missing"?
- Look for cross-environment session coverage: a token minted against one environment's DB used against another (silent 404s)
- Check "user row deleted while session active" and "session expired mid-request"
- Audit OAuth/account-linking, signup-vs-signin distinction, and email-verification gating

## 9. Component & Rendering Coverage (Client)

- For interactive components, check coverage of empty/loading/error/success states, not just the happy render
- Audit form components: validation, submission, optimistic UI, error display, and double-submit prevention
- Look for adversarial input tests (boundary dates, huge strings, unicode, empty arrays) on user-facing inputs
- Check accessibility-affecting behavior (focus management, keyboard nav) where it carries logic, not just styling
- Verify feature-gated UI (free vs paid, role-based) has explicit visibility tests

## 10. Exploratory, Scenario & Product-Critique Coverage (the bugs scripts miss)

- The existing suite almost certainly lives in Q1/Q2 (tests that *support* the team). Audit whether anyone *critiques* the product: is there a charter-driven, time-boxed **exploratory testing** practice (session-based test management — a mission, a time box, notes that make findings reproducible), or does testing stop at scripted assertions? Exploratory testing is *simultaneous test design, execution, and learning* — not ad-hoc clicking — and it's where the most serious bugs (broken multi-step flows, stale-cache surprises, auth-state edge cases, hydration glitches) actually surface. Flag the absence as a P1 process gap.
- Check for **scenario / "soap opera" coverage** of realistic, exaggerated multi-step journeys (sign up → pay → refund → re-subscribe; concurrent tabs mutating the same record; back-button into a stale Server Component; submit a form twice on a flaky connection) rather than isolated per-component tests.
- **Persona coverage:** are adversarial and edge personas exercised — the user who manipulates IDs/params, the double-submitter, the slow-network user, the keyboard-only/screen-reader user, the user on an unsupported browser? Note personas with no representation in the test thinking.
- **Feedback loop:** confirm exploratory findings are converted into automated regression tests — a bug found by hand should become a unit/integration test so it can't silently return.
- Note where exploratory testing is the *right* tool and automation is the wrong one (usability, look-and-feel, one-off investigations) and is simply missing.
- (Search the web for "session-based test management," "exploratory testing charters," and "soap opera testing" to expand these techniques.)

## 11. Non-Functional / "ility" Coverage

- Use an explicit "ility" checklist so the team consciously decides which qualities matter and how important each is — don't let nonfunctional concerns default to "the developers will handle it." Cover at least:
  - **Performance / load / scalability:** are there tests with *measurable* goals (defined concurrent-user count + acceptable response time, Core Web Vitals budgets) and a captured **baseline** so regressions are detectable? Test the whole system, not just the app — DB, network, and any per-instance limits. Confirm load/perf runs execute in an environment that mimics production (and that any result extrapolated from a smaller environment says so explicitly), and that perf tests are re-run when features likely to move the numbers land (heavy queries, new N+1 paths, large payloads) — not deferred to a pre-launch crunch when redesign is no longer affordable. Watch for steadily climbing memory under sustained load (leaks masked by GC) and assert it stays bounded. Flag missing baselines and "should be fast" non-goals.
  - **Reliability:** soak/repeat runs to surface leaks and intermittent failures; assertions against stated SLAs / uptime targets where they exist.
  - **Resilience to dependency failure:** for every out-of-process call (DB, external API, SDK, webhook target), is there coverage that a *slow* or *down* dependency is handled — an explicit timeout fires, retries are bounded with backoff, and the route degrades (fails fast / serves partial content) instead of hanging? The dangerous, frequently-untested case is the slow dependency: blocked requests pile up and exhaust the connection/worker pool, turning one degraded dependency into a whole-app outage (cascading failure). Look for a deliberately-slow/erroring downstream test (fault injection) and a graceful-degradation assertion; flag their absence on any user-facing path that fans out to multiple services.
  - **Security:** see the security-audit prompt — but confirm at minimum that injection, authz, and SSRF paths have negative tests, ideally wired into CI.
  - **Compatibility:** the supported browser/OS/device matrix — is each actually exercised, or assumed?
  - **Installability / deployability:** is the build + migration + deploy path tested as a non-event (the multi-instance, cold-start, and ISR/cache-warm behaviors), not just `next dev` locally? Is the deploy *and rollback* process itself exercised against a staging environment that mirrors production, rather than first attempted in prod? Confirm environment parity — the identical build artifact and the identical migration validated in staging are what reach production (build once, deploy many), since config/schema differences between environments are a leading release-failure source.
  - **Accessibility & i18n** as qualities to verify, not checkboxes (keyboard nav, focus management, RTL, pluralization, locale formatting).
- Performance/security/"ility" tests are listed fourth but should not be done last — flag anywhere they're deferred to the end when redesign is no longer affordable.

## 12. Test Hygiene & Anti-Patterns

- Flag tests that simulate SDK failure the wrong way (throwing where the SDK returns `{ error }`, or vice versa)
- Look for tests asserting call counts only (`toHaveBeenCalledTimes`) without verifying call arguments — they lock in shape, not correctness
- Audit `beforeEach`/`afterEach` env-var manipulation: `process.env` mutations must be restored or state leaks between runs
- Check for tests depending on the system clock / `Date.now` without fake timers — these go red on DST days and year boundaries
- Find skipped tests (`.skip`, `xit`, `xdescribe`) and `// TODO: fix` comments — quantify the latent gap
- Flag weak assertions (`toBeTruthy`/`toBeDefined`) where a stronger `toEqual`/`toBe(<exact>)` is possible
- Audit E2E tests for race conditions: `waitFor` with no timeout, or arbitrary `setTimeout` instead of explicit waits
- Flag inter-dependent tests that must run in a fixed order or share mutable state. Each test should create and tear down its own data and pass in isolation and in any order; a test that leaks rows or leans on a prior test's side effects is a flake waiting to happen. Confirm self-managed, rerunnable fixtures (seed/canonical data refreshed to a known baseline, data the test created cleaned up after) rather than an accreting shared database
- Flag omnibus tests asserting several unrelated business rules at once — one condition per test means a failure pinpoints the cause instead of just saying "something broke"
- Flag **structural coupling** between the test suite and the production shape: a test file mirroring every module one-to-one is coupled to *structure*, not behavior, so any refactor cascades into mass test churn — which pressures the team to stop refactoring. The deeper form is the **Fragile Tests Problem**: business rules verified only by driving the volatile UI through E2E break in bulk on any markup change, so the suite gets disabled rather than maintained. The fix is to verify rules through a stable behavioral seam below the UI (a plain function/service) and reserve E2E for genuine user-journey coverage. Treat tests as part of the system's design — coupled-to-structure tests make the production code rigid, which is a coverage risk in its own right
- Audit E2E/component selector strategy and structure: tests bound to generated DOM ids, class names, or `nth-child` positions break on cosmetic refactors. Prefer stable `data-testid`/role/label hooks, and a layered structure (driver → page-object → test-data) so UI churn is absorbed in one place rather than cascading across the suite. Organize tests by the behavior's *intent*, which rarely changes, not the markup's current *implementation*, which changes constantly. (Search the web for "page object pattern" and "test data builders" to expand these.)

---

## Output Format

Organize all findings into a markdown report grouped by the categories above. For each finding include:

| Field | Description |
|-------|-------------|
| **Priority** | P0 / P1 / P2 / P3 (P0 = enables a known-shape production bug class) |
| **File or Surface** | Exact file path / route / action / cron / component, with line numbers where applicable |
| **Gap** | What is not covered today |
| **Bug-class enabled** | The production failure this gap allows to ship undetected (e.g., "schema-vs-type drift", "SDK-shape upgrade silent break", "counter ↔ DB-state divergence", "Server/Client serialization break", "stale-after-mutation cache") |
| **Proposed test shape** | One sentence describing the test that closes the gap (unit / integration / contract / component / E2E). Do not write the test |
| **Estimated effort** | Half-day / 1 day / 2 days / >2 days |

Begin the report with an executive summary showing a count of gaps per priority level, plus a one-paragraph "what the next production bug looks like if we ship nothing" prediction grounded in the most pressing P0/P1 findings.

---

## Reference: Next.js / full-stack bug classes to use as a checklist

Every category above maps to at least one. Note explicitly when a gap would re-enable one — and prepend any incidents from the App-Specific Context block, which are P0 by default:

- **Fire-and-forget promise:** detached promise on a serverless request, context torn down before resolution, work never completes
- **Side-effect-before-await race:** idempotency stamp / row written before the awaited send/call; the call fails, the stamp persists, the row is never retried
- **DB `Date`/numeric coercion:** a timestamp column returned as a JS `Date` (or numeric as `string`); downstream string ops produce garbage and silently misclassify
- **SDK returns `{ error }` instead of throwing:** `await sdk.method()` resolves with an error object the caller ignores; downstream marks success
- **Schema constraint allowlist drift:** a TS union grows, the SQL `CHECK`/enum doesn't; every INSERT for the new value fails silently
- **Counter ↔ DB-state divergence:** a sweep increments a success counter regardless of whether the row was actually claimed and processed
- **Generic catch-all error message:** a route returns one string for every failure class; the client surfaces it; root cause is invisible
- **Server/Client boundary break:** non-serializable data crossing into a Client Component, or a Server Action missing its own auth check
- **Stale-after-mutation cache:** a write not paired with the correct `revalidatePath`/`revalidateTag`, serving stale data
- **In-memory state on multi-instance host:** a per-process counter/limiter multiplied by N instances, so limits are effectively unbounded
- **Cross-environment session reuse:** a token minted against one DB used against another, 404ing user lookups silently
- **Environment schema/config drift:** a constraint, column, trigger, or index present in production but missing from (or different in) the test/staging schema; code that violates it passes CI and fails only on the real production INSERT, or a migration validated in staging differs from the one run in prod
- **Cascading failure from a slow dependency:** an out-of-process call with no (or too-long) timeout; a slow downstream backs up requests until the pool is exhausted and the whole app stalls — a slow dependency is more dangerous than a down one
- **Permanent-stale cache poisoning:** a wrong or far-future cache directive pins bad content in a CDN/browser cache that no redeploy can flush; recovery requires changing the URL

Findings that enable any of these to recur are P0 by default.
