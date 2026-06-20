# Web Engineering Principles

These principles apply to every prompt in `web/` — build-app, feature-dev, set-up-hosting, refactoring, security, and QA. They are the background framework for every architecture, implementation, and audit decision.

## Simplicity is the deliverable
If two solutions work, ship the one that's easier to read and delete. Optimize for readability and the next engineer who maintains this. Unnecessary abstraction, premature generalization, speculative flexibility, and dependency bloat are bugs.

## Don't gold-plate
Solve the stated problem, not imagined future scale. The simplest version that fully works is the target.

## True vs. accidental duplication
Before unifying code, confirm the duplication is real. True duplication is one concept with one reason to change — every copy must change together, always. Accidental (false) duplication is code that merely looks alike today but serves different use cases and will diverge (two screens, two endpoints, two roles). Unifying accidental duplication couples things that change for different reasons and is painful to unwind — flag it as *deliberately keep separate*. A database row that happens to look like the view/response model is the classic false positive: keep them distinct types. When in doubt, let the duplication stand until the shared rule is proven.

## Dependency Rule — high-level policy must not depend on low-level detail
Source-code dependencies point inward toward the domain. The framework, the database/ORM, and any external SDK are details that should depend on your domain logic, not define it — so they stay swappable and the logic stays testable in isolation without the web server or database running. Don't leak raw DB/ORM rows or API DTOs into Client Components or through API responses; map to types you own at the boundary so storage and transport can change without cascading changes. Organize the top level so it announces the domain and its use cases — what the app *does* — not which framework built it.

## Don't marry the framework — frameworks and SDKs are details
A framework is a detail you use, not one you marry. Keep Next.js/React-specific types, Prisma/Drizzle-generated types, and third-party SDK annotations out of your domain logic. Wrap them behind a boundary (interface/port) you own and confine them to an outer layer — so you can adopt, upgrade, or replace the framework without rewriting the core.

## Lean on the toolchain to hold boundaries
Module/package seams, ESLint import rules (`no-restricted-imports` / `import/no-restricted-paths`), TypeScript project references, and `server-only` / `client-only` markers can make a forbidden dependency fail the build rather than slipping through review. Prefer that over trusting convention. Flag barrel files (`index.ts` re-export hubs) and overly-broad public exports that defeat this by making everything reachable from everywhere.

## Composition root — wire once, inject inward
Wire dependencies in one composition root (a single wiring module or per-request factory) rather than constructing them inline across route handlers, Server Actions, and components. Business logic should not `new` up a Prisma/Drizzle client or an SDK directly. This keeps the framework and SDKs as swappable outer details, gives tests one seam to substitute fakes, and stops framework/vendor types leaking inward.

## Minimize dependencies
Each dependency is a liability — a supply-chain risk, a bundle-size cost, and a maintenance burden. Prefer the platform and framework's built-ins.

## Resilience — assume the network and every dependency can fail or stall
Put an explicit timeout on every out-of-process call. Prefer failing fast and degrading a feature (fail a widget, not the whole page) over hanging the whole response — a slow dependency is more dangerous than a down one, because blocked requests pile up and exhaust the pool. Bound retries with backoff. Make retried or replayed effects idempotent. Read external responses tolerantly: extract only the fields you use, ignore unknown/extra ones.

## Surface trade-offs explicitly
Don't bury decisions in code. A constraint, a risk, or a rejected alternative worth knowing belongs in a plan, a comment, or a handoff note — not silent in the implementation.

## Ask when uncertain
A question is cheaper than a wrong implementation. If a decision is genuinely ambiguous, surface it rather than guessing.

## Cite current documentation
When a decision rests on current framework or host behavior, cite the official Next.js documentation or the confirmed host's docs. Training data can be behind; official sources take precedence over community convention.
