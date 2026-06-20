# Next.js Refactoring Audit Prompt for Claude Code

> Acting as a principal full-stack engineer specializing in maintainability and large Next.js/TypeScript codebases, perform a comprehensive refactoring audit of this codebase. **Do not implement any changes** — document a prioritized refactoring plan only. Every proposed refactoring must be behavior-preserving; flag any that would change observable behavior as out of scope for a refactor. Prioritize by churn × complexity, not aesthetics. Do not propose refactoring stable, working, untouched code unless it actively blocks something.

Read `web/common/engineering-principles.md` — it is the canonical reference for why findings in this audit are flagged (dependency rule, don't marry the framework, toolchain-enforced boundaries, composition root, true vs. accidental duplication, and more).

---

## 0. App-Specific Context (fill this in before running)

<!-- The more you provide, the sharper and more sequenced the plan. -->

- **Next.js version & router:** App Router / Pages Router / mixed (a mixed codebase is itself a refactoring axis)
- **Rendering split:** Server Components / Client Components / Server Actions / route handlers — and how deliberately
- **Data layer:** Prisma / Drizzle / Kysely / raw SQL; where query logic currently lives
- **State & data fetching on the client:** React Query/SWR / Redux/Zustand / context / useEffect-based
- **Validation:** shared Zod/Valibot schemas / duplicated client+server / none
- **Styling:** Tailwind / CSS modules / styled — and how consistently
- **Test coverage reality:** which layers are tested vs. bare (determines safe-to-refactor-first order)
- **Known pain points:** files everyone dreads, frequent merge conflicts, slow areas
- **Constraints:** off-limits areas, a migration already underway (e.g. Pages→App), deprecation deadlines

---

## 1. Refactoring Strategy & Safety Net

- Identify hotspots by cross-referencing file size / complexity with edit frequency (git churn if available). Rank targets by churn × complexity
- **Establish the regression baseline first.** Inventory the full existing test suite — unit, component (RTL), integration, contract, and E2E/browser (Playwright/Cypress) — plus any manual or browser test cases described in Markdown (a UI testing guide, a QA checklist). This enumerated set is the behavior that must continue to pass after every refactor; treat it as the contract the refactoring must preserve, and cite the specific covering test(s) in each finding's safety-net column. Surface this inventory in the report so the reader knows exactly which tests gate the work. Read the baseline through all four testing quadrants (see `shared/testing-quadrants.md`) — not only Q1 unit tests but Q2 acceptance/E2E and any automated or Markdown-described Q3/Q4 checks — so behavior preservation is judged against the whole safety net, not just its fast part
- Lean on the QA audit for coverage truth: if `web/qa/qa-audit.md` has been run, ingest its findings to locate thin/zero-coverage areas rather than re-deriving them; if it hasn't, do a lightweight coverage pass yourself here. For each target, state whether a real safety net exists — untested hotspots need **characterization tests written first** (call this out as a hard prerequisite, naming the missing coverage), and the safe-to-refactor-first order should follow actual coverage, not assumption
- Confirm `tsc --noEmit` passes and is gated; type errors undermine safe refactoring. Flag where loose typing makes refactors risky
- Flag any "refactor" entangled with a behavior change and recommend separating them
- Note mechanical refactors a tool/compiler/codemod can do safely vs. those needing judgment

## 2. Server/Client Boundary & Rendering Model

- Flag Client Components (`'use client'`) that do no client-only work and could be Server Components — and the reverse (server code reaching for client patterns)
- Identify `'use client'` placed too high in the tree, forcing large subtrees client-side unnecessarily — candidates for pushing the boundary down
- Find data fetched in client `useEffect` that should be fetched server-side (Server Component / route handler / Server Action)
- Audit App/Pages Router coexistence and flag patterns that belong on one side or are mid-migration
- Review Server Actions vs. route handlers usage for consistency — pick one idiom per concern

## 3. Component Structure & React Idiom

- Flag oversized components mixing data fetching, business logic, and presentation — candidates for Extract Component / lift logic into hooks or a service layer
- Identify `useEffect` overuse: derived state computed in effects, effects synchronizing state that could be computed during render, fetch-in-effect that belongs server-side
- Audit prop drilling that context, composition, or better data colocation would simplify
- Find repeated JSX / markup and repeated Tailwind class strings that should become shared components or extracted patterns
- Identify reusable logic embedded in components that should become custom hooks

## 4. Separation of Concerns & Layering

- Identify business logic living in route handlers, Server Actions, or components that belongs in a dedicated service / `lib` / domain layer
- Flag database/ORM queries written inline across components/handlers that should live behind a data-access layer
- Audit consistency with the codebase's intended architecture — where it drifts, and whether the drift is load-bearing
- Check that the data-access, domain, and presentation layers have clean seams that make them independently testable
- Apply the **Dependency Rule**: source-code dependencies should point inward toward higher-level policy (the domain / business rules), with the framework, ORM, and third-party SDKs as outer *details* that depend on the domain — never the reverse. Flag inward-pointing violations (domain code importing Prisma/Next.js types, an entity whose shape is dictated by a DB row or an API DTO) and name the fix: introduce an interface/port the domain owns and push the detail behind it
- Draw boundaries along **axes of change** — where two concerns change at different rates and for different reasons (UI vs. business rules, business rules vs. a vendor SDK) — not on instinct. Where fast-churning and slow-churning code are fused in one unit, propose the seam; where a "boundary" only separates things that always change together, flag it as needless indirection to collapse
- Flag the **relaxed-layering cheat**: a component, route handler, or Server Action that bypasses the service/domain layer to hit the data-access layer or ORM directly. Even when the dependency graph stays acyclic, skipping the layer that enforces authorization, validation, and business rules is a smell — name the fix (route the access back through the service seam) and call out where the bypass also drops a security or invariant check. Convention alone ("controllers shouldn't call repositories") won't hold; pair the fix with a mechanical guard (see §9)
- Apply the **Humble Object** move for testability: when hard-to-test logic (branching, formatting, entitlement decisions) is welded into a framework-bound shell (a component body, route handler, or Server Action), split it — keep the shell humble (it only moves data) and extract the decision logic into a plain, framework-free function/module that can be unit-tested directly. This is often the highest-leverage refactor for a low-coverage hotspot, because it converts an E2E-only behavior into a unit-testable one

## 5. Data Fetching & Caching Patterns

- Find duplicated fetch/query logic that should be centralized (a typed client, shared query functions, or a data layer)
- Identify inconsistent caching/revalidation usage across `fetch` calls — candidates for a consistent strategy (named, behavior-preserving)
- Flag N+1 query shapes and repeated query patterns suitable for consolidation (note: if it changes results/perf observably, it's not a pure refactor — flag accordingly)
- Audit React Query/SWR (or equivalent) usage for duplicated keys/config that a shared abstraction would unify

## 6. Type Safety & Validation

- Flag `any`, unsafe casts (`as`), and `@ts-ignore`/`@ts-expect-error` that weaken refactoring safety — candidates for proper typing
- Identify untyped or loosely-typed boundaries (API responses, form data, env access) that should be validated and typed once
- Find duplicated validation logic across client and server that should converge on a single shared schema (Zod/Valibot)
- Identify stringly-typed code and magic strings/numbers that should become unions, enums, or named constants
- Flag scattered `process.env` access that should be centralized and validated

## 7. Duplication & Reuse

- **Before proposing any unification, confirm the duplication is real.** True duplication is one concept with one reason to change — every copy must change together, always. Accidental (false) duplication is code that merely looks alike today but serves different use cases and will diverge (two screens, two endpoints, two roles). Unifying accidental duplication couples things that change independently and is harder to unwind later than the duplication was to tolerate — so flag it as *deliberately keep separate*, not a refactor. The DB-row-shaped object that happens to look like the view/response model is the classic false positive: keep them distinct
- Find copy-pasted or near-duplicate logic (fetching, validation, formatting, auth checks) to unify — name the Extract / Pull-Up move
- Identify parallel implementations that have drifted and should converge on one source of truth
- Flag boilerplate a small helper, hook, or generic would collapse — without over-abstracting (premature abstraction is itself a smell to flag)

## 8. Dead Code, Cruft & Deprecations

- Identify unused exports, components, utilities, routes, and assets; commented-out code; unreachable branches
- Flag feature-flag debris and abandoned migration scaffolding
- Find usages of deprecated Next.js/React APIs (legacy data-fetching methods, deprecated config, old image/link patterns) and note current replacements
- Flag stale `// TODO`/`// FIXME`/`// HACK` markers and assess whether each is real latent debt

## 9. Module & Dependency Structure

- Review folder/module organization — does it reflect the architecture, or is logic scattered across `app`/`components`/`lib` without clear ownership? Ask whether the top-level structure **announces the domain** (what the app does — its features and use cases) or merely screams the framework (`app/`, `pages/`, `components/`) while the domain is scattered. Reorganizing so the structure reveals intent is a valid refactor where ownership is currently unclear
- **Identify boundaries drawn along the wrong axis.** When recent features required modifying many otherwise-unrelated routes, handlers, or modules simultaneously — the "cross-cutting concern" signal — the decomposition was likely made along functional behavior rather than along axes of change. A well-placed boundary lets a new capability be added as a new module or package without touching existing ones (the Open-Closed Principle in practice). Flag areas where successive features have consistently forced broad, coordinated edits as candidates for redrawing the boundary along the dimension that actually changes together
- Identify circular imports and barrel-file (`index.ts`) re-export chains that obscure dependencies or bloat bundles. Break cycles by inverting a dependency (extract an interface) or hoisting the shared piece into a lower-level module
- Flag tight coupling and unclear dependency direction (UI importing infrastructure directly) — candidates for clean seams. Dependencies should run toward **stability**: volatile code (UI, a specific vendor adapter) may depend on stable, abstract core code, but a stable/widely-depended-on module that imports a volatile one is a refactoring target — the volatile detail makes the stable core hard to change
- Assess opportunities to extract shared code into internal packages/modules (or a monorepo workspace) to enforce boundaries
- Enforce boundaries mechanically rather than by discipline. Where the architecture depends on convention to keep illegal imports out, lean on the toolchain: module/package boundaries, ESLint import rules (`no-restricted-imports`/`import/no-restricted-paths`), TypeScript project references, and `server-only`/`client-only` markers so a forbidden dependency (UI importing the data layer, server-only code pulled into a Client Component, a feature reaching into another's internals) fails the build instead of slipping through review. Flag barrel files (`index.ts` re-export hubs) and overly-broad public exports that defeat this by making everything reachable from everywhere.
- Flag **export-visibility inflation** as an architectural smell in its own right: modules or files that re-export every internal type — not just the surface they intend to expose — mean that callers can reach any concrete implementation directly, bypassing the intended boundary. When every module freely exports all its internals, the chosen architectural style (layered, feature-sliced, ports and adapters, component-based) becomes effectively meaningless: directory structure becomes labeling, not enforcement. For each barrel file or module, ask: "Are all these exports genuinely part of the public API, or are they exported by habit?" Trim to the intended surface; unexported internals can be freely renamed or reorganized without breaking callers.
- Also watch for coupling that hides in the **data model**: a Prisma/Drizzle row or an API DTO shape threaded through many modules couples them all to that schema, so a column rename ripples everywhere. Map at the boundary to a type you own (see §7) so the schema can change behind one seam

---

## Output Format

Organize all findings into a markdown report grouped by the categories above. For each proposed refactoring include:

| Field | Description |
|-------|-------------|
| **Priority** | P0 / P1 / P2 / P3 (P0 = high-churn hotspot actively slowing delivery or breeding bugs) |
| **File or Surface** | Exact file path / route / component / module, with line numbers where applicable |
| **Smell** | The named code smell or problem (e.g., "god component", "fetch-in-useEffect", "duplicated validation", "business logic in route handler", "any-typed boundary") |
| **Proposed refactoring** | The named, behavior-preserving move (e.g., Extract Hook, Lift to Service Layer, Push Client Boundary Down, Consolidate to Shared Schema) |
| **Behavior risk & safety net** | Risk it changes observable behavior, and the specific covering test(s) from the baseline inventory that would catch a regression — or, if none exist, that characterization tests are a prerequisite |
| **Effort** | Half-day / 1 day / 2 days / >2 days |

Begin the report with an executive summary: a count of opportunities per priority, the top 3–5 hotspots ranked by churn × complexity, a recommended sequence (what to do first, and what needs a test safety net before being touched), and one sentence on what maintaining the status quo costs over the next few quarters.
