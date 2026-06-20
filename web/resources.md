# Web Resources

Organized by application slice. When a prompt instructs you to research current framework guidance, start here — these are the authoritative, regularly-updated sources. Official documentation (Next.js, React, TypeScript, OWASP, MDN) takes precedence over community convention whenever they conflict.

---

## Rendering Model

How pages are rendered, when they are generated, and how caching and streaming work — the decisions that determine SSG vs. ISR vs. SSR vs. client rendering.

- **Next.js Docs — App Router** — https://nextjs.org/docs/app
  The canonical source for Server Components, Client Components, Server Actions, route handlers, middleware, streaming, and image optimization. Prefer this over training-data memory; the caching model changes across minor versions.

- **Next.js Caching Guide** — https://nextjs.org/docs/app/building-your-application/caching
  Documents the four caching layers (Request Memoization, Data Cache, Full Route Cache, Router Cache), their lifetimes, and how `revalidatePath`/`revalidateTag` interact with each. Required reading before any ISR or mutation/revalidation decision.

- **Next.js Server Actions** — https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations
  Invocation patterns, progressive enhancement, error handling, and security model. Critical context for `web/security-audit.md`'s point that Server Actions are public endpoints regardless of which UI renders them.

- **Next.js API Reference** — https://nextjs.org/docs/app/api-reference
  Function signatures for `revalidatePath`, `revalidateTag`, `cookies`, `headers`, `redirect`, `notFound`, and `next.config.js` options. The authoritative answer when you are unsure whether a behavior is documented or inferred.

- **React Docs** — https://react.dev
  Primary reference for Server vs. Client Component semantics, Suspense, the `use` API, `useActionState`, `useOptimistic`, `useFormStatus`, form actions, and current state-management conventions. Use alongside Next.js docs — they separate framework concerns from React concerns.

---

## Data Layer

ORM setup, query patterns, migrations, connection pooling, and the schema-to-domain mapping boundary.

- **Prisma Docs** — https://www.prisma.io/docs
  Schema definition, `prisma migrate`, the generated client, relation queries, and transactions. `web/refactoring.md` §4 flags raw ORM queries inline in components as a smell — these docs show the intended boundary-aware usage.

- **Drizzle ORM Docs** — https://orm.drizzle.team/docs/overview
  Schema-as-code, type-safe queries, `drizzle-kit` migrations, and relation handling. Preferred over Prisma when bundle size or edge-runtime compatibility matters.

- **Kysely Docs** — https://kysely.dev/docs/getting-started
  Type-safe SQL query builder without the ORM overhead. Common when you want full SQL control with type safety at the query boundary.

- **node-postgres type coercion** — https://node-postgres.com/features/types
  Documents the JavaScript types the `pg` driver returns for each Postgres column type (e.g., `NUMERIC`/`DECIMAL` → `string`, `TIMESTAMP` → `Date`). The root cause of the DB date/numeric coercion bug class; check here before assuming what type a query returns.

---

## Authentication & Session

Login flows, token lifecycle, cookie security attributes, and session invalidation.

- **Auth.js (NextAuth v5) Docs** — https://authjs.dev
  Session handling, providers, callbacks, JWT vs. database sessions, the adapter API, and App Router integration. Referenced in `web/security-audit.md` §1 and §2 for session validation, cookie config, and middleware gating.

- **Clerk Docs** — https://clerk.com/docs
  Hosted auth with Next.js middleware integration, `currentUser()`/`auth()` helpers, and organization/role support. Common alternative when you need hosted identity with a richer feature set out of the box.

- **OWASP Cheat Sheet — Session Management** — https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
  Authoritative guidance on session fixation, token entropy, cookie attributes (`HttpOnly`, `Secure`, `SameSite`), and session invalidation on logout/password change. The reference the security audit prompt uses for §1 findings.

- **OWASP Cheat Sheet — Authentication** — https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
  Password hashing (bcrypt/argon2/scrypt cost factors), account enumeration prevention, and multi-factor authentication requirements.

---

## Authorization & Access Control

Protecting routes, Server Actions, and handlers — and distinguishing horizontal from vertical privilege escalation.

- **Next.js Authentication Guide** — https://nextjs.org/docs/app/building-your-application/authentication
  Official guidance on middleware matchers, protecting Server Actions, and enforcing auth in Server Components. Directly underpins `web/security-audit.md` §1 and §2.

- **OWASP Cheat Sheet — Authorization** — https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html
  RBAC, ABAC, and PBAC patterns; deny-by-default; horizontal vs. vertical privilege escalation. The reference the security audit uses when auditing every route handler and Server Action.

- **OWASP Cheat Sheet — Access Control** — https://cheatsheetseries.owasp.org/cheatsheets/Access_Control_Cheat_Sheet.html
  Principles for enforcing access decisions consistently across all layers (middleware, handler, service, data layer) rather than in one place only.

---

## Input Validation

Schema enforcement at every inbound boundary — form data, route params, headers, webhooks.

- **Zod Docs** — https://zod.dev
  The most common schema library in Next.js codebases. Used for shared client/server validation, request parsing, and typed form data. `web/feature-dev.md` Phase 4 requires validating all new endpoint/action inputs; Zod is the typical tool.

- **Valibot Docs** — https://valibot.dev
  Bundle-size-focused alternative to Zod with a modular, tree-shakeable API. Consider on edge runtimes where Zod's bundle contribution matters.

- **OWASP Cheat Sheet — Input Validation** — https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html
  Allowlist vs. blocklist strategies, canonicalization, and injection-resistant validation patterns. Referenced by `web/security-audit.md` §5.

---

## API Surface

Route handlers, CORS, rate limiting, method gating, and error response hygiene.

- **Next.js Route Handlers** — https://nextjs.org/docs/app/building-your-application/routing/route-handlers
  Method gating, request parsing, response construction, and CORS headers. The authoritative reference for any new API endpoint added in `web/feature-dev.md`.

- **OWASP Cheat Sheet — REST Security** — https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html
  Input validation, output encoding, CORS configuration, HTTP method gating, and error response hygiene for REST APIs. Backs `web/security-audit.md` §6.

- **OWASP Cheat Sheet — CSRF Prevention** — https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html
  Next.js App Router Server Actions have built-in CSRF protection for same-origin calls; this document covers edge cases and route handler patterns where you must add your own.

- **OWASP Cheat Sheet — SSRF Prevention** — https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
  Required reading for `web/security-audit.md` §6's SSRF bullet: any server-side `fetch` built from user-supplied input (webhook targets, redirect params, image URLs, AI tool calls) is an SSRF vector.

---

## Payments & Webhooks

Stripe integration, webhook signature validation, idempotency, and server-side access grants.

- **Stripe Developer Docs** — https://stripe.com/docs
  Payment Intents, Checkout Sessions, webhook event types, idempotency keys, and the Node.js SDK. The reference for any payment or subscription feature.

- **Stripe — Webhook Security** — https://stripe.com/docs/webhooks/signatures
  `stripe.webhooks.constructEvent()` with HMAC-SHA256 over `Stripe-Signature`, 5-minute clock tolerance. Every webhook Route Handler must call this first and return 2xx before processing asynchronously. `web/security-audit.md` §4 flags missing signature verification as a critical finding.

- **Stripe — Idempotency Keys** — https://docs.stripe.com/api/idempotent_requests
  `Idempotency-Key` header for safe POST retries without double-charging. Required for any POST creating a charge, subscription, or customer. Store the key before the request, not after. Also log incoming webhook event IDs for deduplication.

- **Stripe — Subscription Lifecycle** — https://docs.stripe.com/billing/subscriptions/overview
  All subscription status values and state transitions. Provision on `active`/`trialing`; revoke on `unpaid`/`canceled`. The `incomplete` → `incomplete_expired` silent transition (23-hour window) is a common bug — handle it explicitly. Required reading when `web/security-audit.md` §4 audits whether revocation is parallel to grant.

---

## File Uploads

Multipart handling, type and size validation, storage, and presigned URL security.

- **Next.js File Upload patterns** — https://nextjs.org/docs/app/building-your-application/routing/route-handlers#request-body
  How to receive multipart/form-data in a route handler or Server Action.

- **AWS S3 Presigned URLs** — https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html
  How presigned URLs work, their expiry semantics, and how to scope them to the owning user. `web/security-audit.md` §3 flags guessable or improperly scoped file URLs as a high finding.

- **Cloudflare R2 Docs** — https://developers.cloudflare.com/r2/
  S3-compatible object storage. Common alternative to AWS S3 for Next.js apps hosted on Vercel or edge platforms.

- **OWASP Cheat Sheet — File Upload** — https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
  Type validation (magic bytes, not just extension), size limits, storage location isolation, and filename sanitization. Required reading for any file upload endpoint.

---

## Caching & Revalidation

Next.js cache layers, mutation triggers, and avoiding shared-cache data leaks across users.

- **Next.js Caching Guide** — https://nextjs.org/docs/app/building-your-application/caching
  (See Rendering Model above.) Also the reference for diagnosing stale-cache-after-mutation bugs flagged in `web/security-audit.md` §10 (ISR caching user-specific data in a shared cache).

- **Next.js `revalidatePath` / `revalidateTag`** — https://nextjs.org/docs/app/api-reference/functions/revalidatePath
  API reference for cache invalidation after mutations. `web/feature-dev.md` Phase 4 requires pairing every mutation with the correct revalidation call. Non-obvious: for dynamic routes, `revalidatePath` requires `type: 'layout' | 'page'` — without it, the call silently does nothing.

- **`"use cache"` directive (Next.js 16+)** — https://nextjs.org/docs/app/api-reference/directives/use-cache
  The new file/component/function-level caching directive replacing `unstable_cache`. Closures from outer scopes (e.g., `userId`) are automatically captured into the cache key, so different users get separate cache entries. Cannot call `cookies()` or `headers()` inside — read them outside and pass as arguments to prevent cross-user data leaks.

- **`unstable_cache` (Next.js ≤15)** — https://nextjs.org/docs/app/guides/caching-without-cache-components
  Pre-v16 caching model for wrapping DB queries and non-`fetch` async functions. Critical caveat: `keyParts` must include any external variables in the closure — omitting them causes cache collisions across users. Superseded by `"use cache"` in v16.

---

## Security Headers

CSP, HSTS, frame protection, and the `next.config.js` header setup.

- **Next.js — Security Headers** — https://nextjs.org/docs/app/building-your-application/configuring/content-security-policy
  How to configure CSP (including nonce-based CSP for App Router), `Strict-Transport-Security`, `X-Content-Type-Options`, and `X-Frame-Options` via `next.config.js`. Required reading when `web/security-audit.md` §9 identifies missing security headers.

- **MDN — HTTP Security Headers** — https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers#security
  Authoritative reference for every security-relevant HTTP header: semantics, valid values, and browser support. Use to verify a header's behavior before adding it.

- **OWASP Cheat Sheet — HTTP Security Response Headers** — https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html
  Recommended header values and configurations. The checklist backing `web/security-audit.md` §9.

- **`server-only` package** — https://www.npmjs.com/package/server-only
  Drops a compile-time error if a module marked `server-only` is imported into a Client Component or client bundle. The toolchain enforcement mechanism for the Server/Client boundary described in `web/common/engineering-principles.md`.

---

## Client State & Data Fetching

React Query/SWR, Zustand, Jotai, and avoiding client-side data-fetching anti-patterns.

- **TanStack Query (React Query) Docs** — https://tanstack.com/query/latest/docs/framework/react/overview
  Client-side data fetching, caching, background refetching, and mutation patterns. The reference when `web/refactoring.md` §5 flags duplicated `useEffect`-based fetch logic that should be centralized.

- **SWR Docs** — https://swr.vercel.app/docs/getting-started
  Stale-while-revalidate data fetching hook from Vercel. Common in Next.js codebases; the reference for its deduplication and revalidation semantics.

- **Zustand Docs** — https://docs.pmnd.rs/zustand/getting-started/introduction
  Lightweight client state management. Use when the codebase uses Zustand and a feature needs to integrate with or extend that store.

- **Jotai Docs** — https://jotai.org/docs/introduction
  Atomic state management. An alternative to Zustand, common in Next.js apps that need fine-grained reactivity.

---

## Testing

Vitest, React Testing Library, Playwright, and MSW — and how they map to the four quadrants.

- **Vitest Docs** — https://vitest.dev
  The standard unit/integration test runner for modern Next.js projects. Covers config, mocking, coverage, and watch mode. `web/qa/qa-audit.md` and `web/refactoring.md` treat Vitest in CI as table stakes.

- **React Testing Library** — https://testing-library.com/docs/react-testing-library/intro
  Component testing focused on user-visible behavior. Use query-by-role and query-by-label patterns; avoid coupling tests to generated class names or `nth-child` selectors.

- **Playwright Docs** — https://playwright.dev/docs/intro
  E2E and browser automation. `web/qa/qa-audit.md` treats Playwright as the tip of the pyramid — few, high-value user-journey tests, not a substitute for a unit/integration base.

- **MSW (Mock Service Worker)** — https://mswjs.io
  API mocking at the network level, usable in both browser (Playwright) and Node (Vitest) environments. Intercepts real `fetch` calls, avoiding the "mocked boundary proves nothing" anti-pattern.

- **Jest Docs** — https://jestjs.io/docs/getting-started
  Older but still common in Pages Router projects. Use when the codebase already uses Jest; switching has non-trivial cost.

- **Agile Testing Quadrants** — https://lisacrispin.com/2011/11/08/using-the-agile-testing-quadrants/
  Lisa Crispin's explanation of the Q1–Q4 model referenced in `shared/testing-quadrants.md`. Use when deciding which tier a test belongs in.

---

## Type Safety & Toolchain Boundary Enforcement

TypeScript strict mode, ESLint import rules, project references, and the tools that make architectural boundaries compile-time enforced.

- **TypeScript Docs** — https://www.typescriptlang.org/docs/
  Utility types, strict mode, generics, narrowing, and declaration files. The authoritative reference for any typing decision.

- **TypeScript Project References** — https://www.typescriptlang.org/docs/handbook/project-references.html
  Compiles a multi-package repo as separate TS projects with declared dependency edges. Illegal cross-package imports fail `tsc` rather than relying on lint. The strongest mechanical enforcement for a layered architecture.

- **ESLint `import/no-restricted-paths`** — https://github.com/import-js/eslint-plugin-import/blob/main/docs/rules/no-restricted-paths.md
  Declares that files in one directory cannot import from another (e.g., `app/` cannot import from `infrastructure/` directly). The ESLint-side complement to project references.

- **Dependency Cruiser** — https://github.com/sverweij/dependency-cruiser
  Validates the actual import graph against declared architectural rules and runs in CI. Fills the gap between ESLint rules (file-level) and full module graph enforcement.

---

## Performance & Observability

Core Web Vitals, bundle analysis, structured logging, and distributed tracing.

- **Core Web Vitals** — https://web.dev/articles/vitals
  LCP, CLS, INP — the three metrics `web/build-app.md` Phase 3 requires measurable targets for before implementation begins. Defines what "should be fast" must mean concretely.

- **Lighthouse** — https://developer.chrome.com/docs/lighthouse/overview/
  Automated lab-based performance, accessibility, SEO, and best-practices auditing. The standard Q4 non-functional tool.

- **Vercel Speed Insights** — https://vercel.com/docs/speed-insights
  Real-user Core Web Vitals collection. Integrates with zero config when the hosting target is Vercel.

- **OpenTelemetry for Next.js** — https://nextjs.org/docs/app/building-your-application/optimizing/open-telemetry
  Distributed tracing instrumentation via the `instrumentation.ts` hook. The reference for observability requirements in `web/build-app.md` Phase 3 on non-Vercel platforms.

---

## Dependency & Supply Chain

CVE scanning, lockfile integrity, and supply-chain analysis beyond known CVEs.

- **npm audit** — https://docs.npmjs.com/cli/v10/commands/npm-audit
  Built-in vulnerability scanning. First-line dependency audit tool for `web/security-audit.md` §8.

- **Snyk** — https://snyk.io
  More detailed CVE scanning and fix suggestions than `npm audit`, with GitHub Actions integration. Relevant for unpinned/wildcard version findings.

- **Socket.dev** — https://socket.dev
  Supply-chain analysis that flags packages with unexpected network access, install scripts, and ownership changes — goes deeper than CVE scanning alone.

---

## Hosting & Deployment

Platform-specific runtime constraints that shape implementation decisions.

- **Vercel Docs** — https://vercel.com/docs
  Next.js deployment, environment variables, preview deployments, Edge vs. Node runtime support, function timeouts, ISR behavior, and image optimization. Treat these as implementation constraints when Vercel is the confirmed host.

- **Vercel Function Limits** — https://vercel.com/docs/functions/limitations
  Hard limits: max duration (Hobby 300s, Pro up to 800s), memory 2 GB, request/response payload **4.5 MB** (mandates direct-to-S3 for file uploads), edge code size 1–4 MB, env vars 64 KB total. Check here before any feature that involves large payloads, long-running tasks, or many secrets.

- **Vercel Edge Runtime Restrictions** — https://vercel.com/docs/functions/runtimes/edge
  Banned Node.js APIs at the edge (`fs`, native modules, `eval`, `require()`). Any route touching a DB client or native addon must use the Node.js runtime. Middleware (auth, geo, redirects) remains the valid edge use case.

- **Railway Docs** — https://docs.railway.com
  Container-based deployment with managed Postgres, Redis, and private networking. Use `web/set-up-hosting.md` to evaluate Railway when the app needs WebSockets, long-running jobs, or self-managed compute.

- **Fly.io Docs** — https://fly.io/docs
  Multi-region container hosting. Referenced for apps requiring low-latency global read paths or persistent WebSocket connections.

---

## Security (Audit Reference)

OWASP standards and authoritative security checklists for full-stack web applications.

- **OWASP Top 10** — https://owasp.org/www-project-top-ten/
  The canonical checklist for web application vulnerabilities (injection, broken access control, cryptographic failures, insecure design). `web/security-audit.md` drives its categories from this framing.

- **OWASP ASVS (Application Security Verification Standard)** — https://owasp.org/www-project-application-security-verification-standard/
  Detailed verification requirements organized by level (L1/L2/L3). Use as a structured checklist when a security audit needs to be comprehensive or compliance-driven.

- **OWASP Web Security Testing Guide (WSTG)** — https://owasp.org/www-project-web-security-testing-guide/
  Test procedures for each OWASP Top 10 category. Use when `web/security-audit.md` needs a concrete test procedure to verify a specific control.

- **MDN Web Security** — https://developer.mozilla.org/en-US/docs/Web/Security
  Browser-side security model: SameSite cookies, CORS, CSP, mixed content, clickjacking, and subresource integrity. The authoritative reference for web platform security semantics.

---

## UI & Component Patterns

Design system primitives and component conventions.

- **Tailwind CSS Docs** — https://tailwindcss.com/docs
  Utility-first CSS framework. `web/refactoring.md` §3 flags repeated Tailwind class strings that should become shared components or `cn()` utilities.

- **shadcn/ui** — https://ui.shadcn.com
  Radix-based component collection with Tailwind styling. If the codebase has a `components/ui/` folder, this is likely its source; consult here before building a new component from scratch.

- **Radix UI Primitives** — https://www.radix-ui.com/primitives
  Unstyled, accessible primitives (Dialog, Popover, Select, etc.). The base layer under shadcn/ui and similar systems.

- **Component Gallery** — https://component.gallery
  Real-world component design patterns across production UI systems. Referenced in the global CLAUDE.md for avoiding generic AI aesthetics when building frontend components.

---

## Code Quality & Architecture (Source Texts)

The intellectual sources `web/common/engineering-principles.md` distills. Cite these when a principle is contested or needs deeper justification.

- **Clean Code** — Robert C. Martin (O'Reilly, 2008) — https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882
  Source for naming, single-function, comments, and Simple Design four-rules principles in `engineering-principles.md`.

- **Clean Architecture** — Robert C. Martin (Pearson, 2017) — https://www.amazon.com/Clean-Architecture-Craftsmans-Software-Structure/dp/0134494164
  Source for the Dependency Rule and the Humble Object pattern named explicitly in `engineering-principles.md` and `web/refactoring.md` §4.

- **Growing Object-Oriented Software, Guided by Tests** — Freeman & Pryce (Addison-Wesley, 2009) — https://www.amazon.com/Growing-Object-Oriented-Software-Guided-Tests/dp/0321503627
  Source for the outside-in TDD approach and the "extract to a plain function, test it without the server" discipline in `web/qa/qa-audit.md` §1.

- **Agile Testing** — Crispin & Gregory (Addison-Wesley, 2009) — https://www.amazon.com/Agile-Testing-Practical-Guide-Testers/dp/0321534468
  Source for the four-quadrant testing model in `shared/testing-quadrants.md`.

- **Building Microservices** — Sam Newman (O'Reilly, 2nd ed. 2021) — https://www.amazon.com/Building-Microservices-Designing-Fine-Grained-Systems/dp/1492034029
  Background for "draw boundaries along axes of change" and "identify the actors who drive change" in `web/build-app.md` Phase 5 and `web/refactoring.md` §9.

- **Enterprise Integration Patterns** — Hohpe & Woolf (Addison-Wesley, 2003) — https://www.amazon.com/Enterprise-Integration-Patterns-Designing-Deploying/dp/0321200683
  Background for the resilience requirements in `web/feature-dev.md`: idempotent retries, bounded backoff, and the command/query distinction. The side-effect-before-await race is a classic messaging anti-pattern documented here.
