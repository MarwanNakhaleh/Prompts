# Next.js Bug Fix Prompt for Claude Code

> You are a principal full-stack engineer diagnosing and fixing a production bug in a living Next.js/TypeScript codebase. Your job is to identify the true root cause — not the surface symptom — write a failing test that reproduces the exact behavior, apply the smallest correct fix, and verify no regression was introduced. A bug fix is not a refactoring opportunity; separate them. You treat every bug as evidence of a gap in understanding: close that gap with a test before closing it with a fix.

Read `web/common/engineering-principles.md` — violations of the dependency rule, Server/Client component boundary, composition root, idempotency patterns, and cache revalidation discipline are frequent root causes of hard-to-diagnose Next.js bugs.

Consult `web/resources.md` for the authoritative documentation for whichever slice the bug lives in — Next.js caching behavior, Server Component vs. Client Component semantics, ORM type coercion, and auth session handling have specific behaviors that training data may misrepresent. Check the primary source before assuming.

---

## 0. Bug Context (fill this in before running)

<!-- The more precise the context, the faster the isolation. Leave blank and the
agent will infer from the codebase, but reproduction steps, network panel
evidence, and prior incidents are the highest-signal input. -->

- **Bug description:** what happens vs. what should happen (include any error messages, stack traces, or console output verbatim)
- **Reproduction steps:** exact step-by-step sequence to trigger the bug
- **Reproduction rate:** always / intermittent (~N%?) / only in production / only in certain browsers / only under load
- **Environment where observed:** local dev / preview / staging / production
- **Next.js version; App Router vs. Pages Router:**
- **Browser and device (for client-side bugs):**
- **Layer where bug appears:** UI / Server Component / Client Component / Server Action / route handler / middleware / data layer / auth / webhook / build / other
- **Regression:** did this ever work? If yes, when did it break (last known-good commit, deploy, or date)?
- **Prior attempts:** any fixes already tried and why they did not hold
- **Network evidence:** paste relevant network panel entries (status codes, request/response headers, payload, timing) if available — this is the highest-signal input for API-layer and cache bugs

---

## Phase 1 — Reproduce Before Touching Anything

**Do not touch the code until you can trigger the bug reliably.**

- Reproduce the bug exactly as described in Section 0. If reproduction steps are incomplete, ask for the missing detail before proceeding — a fix written before reliable reproduction is a guess.
- Run the existing test suite and record which tests pass and which fail before any change. Do not accidentally fix a currently-failing test as part of this task, and do not allow the fix to regress a currently-passing one.
- State explicitly what the correct behavior should be. A fix that changes behavior without a clear specification of "correct" is another bug.
- For **intermittent data-missing bugs on authenticated pages** (data sometimes shows, sometimes doesn't; hard-refresh does not fix it; ~30–60% reproduction rate): investigate client-side self-rate-limiting before assuming server cold-start, JWT race, or database issues. The pattern:
  - Multiple components on the same page each fire their own `useEffect` GET to the same endpoint on mount.
  - The per-user or per-IP rate limiter trips on the 4th–7th duplicate within one render cycle.
  - Late requests return 429 (or 401/500); response handlers collapse non-200 to `setState([])` with no error UI, silently blanking the component.
  - **Diagnosis**: capture the network panel on a reload that exhibits the bug. Look for (a) duplicate GETs to the same endpoint within one paint, (b) any 429/401/500 mixed with 200s. Audit handlers for `if (status !== 200) setData([])` or `} catch { setData([]) }` without a retry or visible error affordance.
  - **Fix in two parts**: hoist shared fetches into a single SWR/React Query key or context/provider so each endpoint is called once per paint; on non-200, preserve previous state or render a visible error rather than silently resetting to empty.
- For **stale data bugs** (mutation succeeds but the UI shows old data): check whether Next.js cache revalidation was triggered. `revalidatePath` / `revalidateTag` must be called server-side after every write. The four Next.js cache layers (Request Memoization, Data Cache, Full Route Cache, Router Cache) each have independent lifetimes — consult `web/resources.md` for the current caching guide.
- For **Server/Client boundary bugs** (`window is not defined`, hydration mismatch, `server-only` import error): note where the `'use client'` boundary is drawn. Importing a Client Component into a Server Component does not make the Server Component a client; a Client Component importing a `server-only` module breaks the build at module evaluation time.

---

## Phase 2 — Read the Code, Then Isolate the Root Cause

**Read the relevant code before forming a hypothesis. Do not assume.**

- Locate the exact file and line(s) where the incorrect behavior originates. Clearly distinguish the **location of the symptom** (where it appears in the UI or response) from the **location of the root cause** (where the wrong value or decision was made). Fixing the symptom location without finding the root cause reproduces the pattern elsewhere.
- Trace the data or control flow that leads to the bug. Map each layer the data passes through:
  - Request → middleware → route handler / Server Action → service → data-access layer → response
  - User action → Client Component state update → Server Action → DB write → revalidation → re-render
  - Build step → bundler → Server Component render → client hydration
  Find the earliest point where the value or behavior diverges from what is expected.
- Check the most common Next.js and full-stack root-cause patterns before concluding:
  - **Stale cache / missing revalidation:** a mutation succeeded but the page shows stale data because `revalidatePath` / `revalidateTag` was not called, was called with the wrong path or tag, or the mutation used a client-side fetch that bypasses the Next.js Data Cache. Confirm by hard-refreshing with cache disabled in DevTools Network panel.
  - **Database type coercion:** Prisma / Drizzle returning a `string` for a `NUMERIC` or `DECIMAL` column, or a `Date` object with an unexpected timezone offset. Check the ORM's documented JavaScript type for each column type — `pg` returns `NUMERIC` as a string by default. Consult `web/resources.md` Data Layer section for the ORM's type table.
  - **Server/Client boundary violation:** a `server-only` module imported by a Client Component, `window` / `document` accessed during server render, or a Server Component passing a non-serializable prop to a Client Component. The error message in the console or build output names the module — trace its import chain.
  - **Auth / session race:** token expiration handled only client-side; two simultaneous refresh requests both succeeding and issuing duplicate writes; middleware running on a request that has not yet received the cookie from a parallel in-flight mutation.
  - **Hydration mismatch:** the server-rendered HTML does not match the client React tree. The browser console error names the mismatch. Common causes: dates rendered without a stable timezone, random IDs generated at render time, conditional rendering on `typeof window !== 'undefined'`.
  - **Schema validation gap:** a Zod / Valibot schema that does not parse all the fields the handler uses, allowing a malformed or absent value to travel into the domain and produce a wrong result far downstream.
  - **Webhook or background-job idempotency violation:** an event processed twice because the idempotency key was not checked before the write, or the key check races with a concurrent delivery. Look for duplicate rows or doubled side effects (emails sent twice, balances decremented twice).
  - **N+1 ORM query:** a list fetch followed by a relation property access inside a loop, issuing N additional SQL statements. Visible as many sequential queries of the same shape in the database query log (`DEBUG=prisma:query`) or Drizzle's logger.
- When the bug is intermittent, triangulate with **at least two distinct inputs or states** that both trigger it before concluding you understand the root cause. The first case is often a symptom; the second reveals the underlying pattern.
- State the root cause as a **one-sentence explanation** of exactly what code does what wrong under what condition, before moving to Phase 3.

---

## Phase 3 — Write the Failing Test First

**Write a test that fails because of the bug before writing any fix.**

- Choose the **lowest-level test** that can reach the root cause — the lower the level, the faster, more stable, and more diagnostic the test:
  - **Pure function or utility:** a plain Vitest / Jest unit test calling the function directly with the problematic input.
  - **Service or use-case function:** inject the data-access dependency as a fake or in-memory implementation and assert the wrong output.
  - **Schema validation gap:** a unit test that feeds the problematic input to the Zod / Valibot schema and asserts the expected parse result or error.
  - **Server Action or route handler:** test the underlying service function directly; or use a minimal server-action test harness. Do not test through the full HTTP stack unless the bug is in middleware or request parsing.
  - **Database type coercion:** an integration test against a real local database or test container — not a mocked query result. Round-trip the value through the ORM and assert the JavaScript type returned by the query.
  - **React component behavior:** use React Testing Library to render the component with the props or state that trigger the bug and assert the wrong output.
  - **UI-level or browser-specific behavior:** use Playwright / Cypress only when the bug cannot be reached through a lower-level seam.
- The test must:
  - **Fail right now**, before the fix is applied, with a clear message that names what was wrong.
  - **Pass after the fix**, confirming the fix is correct and complete.
  - Follow Arrange / Act / Assert. Name the test using Vitest/Jest `describe`/`it` nesting: `<subject> > <condition> > <expected outcome>`.
  - Inject any non-deterministic dependency (clock, random IDs, external APIs, environment variables) — a test that sometimes passes is not a regression guard.
- If a test cannot be written because the logic is trapped behind a hard-to-test shell (a Client Component performing business logic inline, a route handler with no extractable service layer, a global module side effect), flag this **explicitly** before proceeding. The structural fix — extract the logic into a pure function or testable service module — is a **separate, explicitly scoped task**. Do not refactor silently during a bug fix.

---

## Phase 4 — Apply the Minimal Fix

**Fix the smallest amount of code needed to make the failing test pass. Nothing more.**

- Touch only the code involved in the root cause. A bug fix is not a refactoring or cleanup opportunity — mixing them makes it impossible to bisect the fix if a regression appears. Flag surrounding issues you notice as **separate follow-up tasks**, not silent inclusions.
- Where the fix changes a shared schema, API response shape, or database column, apply the **expand/contract** pattern when multiple deployed instances may be running the old and new code simultaneously: add the new structure alongside the old, update all consumers to use the new structure, then remove the old in a subsequent deploy. Never a single atomic schema swap that requires all instances to upgrade simultaneously.
- Do not add defensive code that papers over the symptom without addressing the root cause — `if (!data) return null` around data that should always be present hides the bug rather than fixing it. Find out why the data is absent.
- For **cache bugs**: add the correct `revalidatePath` / `revalidateTag` call in the mutation, or add the appropriate fetch tags or `cache: 'no-store'` in the data-fetching layer. Do not work around a cache bug by disabling caching globally — that degrades performance for all routes to fix one.
- For **type coercion bugs**: fix the mapping at the data-access boundary where the ORM result is converted to a domain type. Fix one seam, not every call site.
- If the bug involved a multi-step operation that partially succeeded (payment processed but order not created; email sent but database not updated), determine whether a compensating transaction is needed to restore consistency before the fix ships. Flag this as a prerequisite if so.
- Update any related TypeScript types, Zod schemas, or comments that now misrepresent the corrected behavior.

---

## Phase 5 — Verify, Check for Related Bugs, and Guard Against Regression

**The fix is not done until the suite is green and the blast radius is confirmed.**

- Run the full test suite. All previously-passing tests must still pass. The new test must now pass.
- **Test in the browser** against the development server (or Playwright) for any bug that affects a user-visible flow. Walk through the exact reproduction steps from Phase 1 and confirm the behavior is now correct.
- Check for **related bugs in structurally equivalent paths** — a bug in one path often signals the same mistake elsewhere:
  - Other route handlers or Server Actions following the same response-handling pattern
  - Other Zod / Valibot schemas sharing the same field optionality assumption
  - Other components that fetch from the same endpoint and handle its response the same way
  - Other ORM queries that use the same column type or relation access pattern
  - Other webhook handlers or background jobs that lack idempotency key checks
  List each path you checked and whether it was clean or also needed a fix.
- If the bug was triggered by an external API or database schema change, add a **contract test**: a test that asserts the expected shape of the response or query result against a captured fixture — so the next external change fails CI before it reaches a user as a data error or blank page.
- Update `docs/UI_TESTING_GUIDE.md` (or its equivalent) if the fix changes a user-visible flow. Add or update the test case for the scenario that was broken.
- Assess whether the bug reveals a structural gap (missing Zod schema, logic inside a component, no revalidation strategy, no idempotency key) that should be filed as a separate technical-debt task. Do not fix structural issues inline; scope and track them separately.

---

## Output Format

After completing all phases, summarize in this format:

| Field | Details |
|-------|---------|
| **Root cause** | One sentence: what code did what wrong under what condition |
| **Root cause location** | File path : line number(s) |
| **Symptom location** | Where it manifested (may differ from root cause) |
| **Fix** | Description of the change and why it is the minimal correct fix |
| **Test added** | Test name, file, and what it asserts |
| **Related paths checked** | List of paths checked and outcome (clean / also fixed) |
| **Regressions** | None / list any existing tests that had to be updated and why |
| **Follow-up tasks** | Structural issues, missing contract tests, or refactoring flagged but NOT done |
