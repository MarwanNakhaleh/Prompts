# Prompt Library Guide

This repository is a library of prompts for taking a product from a rough idea
all the way to revenue: validating that the problem and demand are real,
positioning it, turning it into requirements, implementation plans, production
code, and hosting decisions, and running post-build audits. The prompts are
written for an LLM that can inspect a codebase, ask clarifying questions,
research current platform guidance, and pause for human approval at the right
moments.

The work splits into two arcs that meet in the middle. The **company-building**
prompts (`validation/`, `product/`, `marketing/`) de-risk *what* to build and
*who it's for* before and around the engineering. The **engineering** prompts
(`ios/`, `web/`) build, test, and harden it. Validation feeds product scoping,
product scoping feeds the build, and positioning feeds go-to-market — so a prompt
that ends by handing a clean brief to the next one is doing its job.

Use the prompts as workflows, not as passive reference docs. Each prompt defines
a role, the required input, the order of work, and the points where the LLM must
stop and wait for the user.

The build prompts (`build-app.md`) stand up an app's foundation and then **delegate
individual features to a dedicated feature-development prompt** (`feature-dev.md`),
fanning independent features out as parallel agents. So "build the app" and "add a
feature" are separate, composable prompts — the former orchestrates many runs of
the latter.

## Prompt Map

- `validation/customer-interviews.md`: Run customer-discovery interviews that
  surface the truth instead of flattery (the Mom Test). Pins down the riskiest
  belief and the decision the answers will inform, builds a bias-free interview
  script with an explicit bad-questions list, role-plays or coaches the
  conversations, and synthesizes the notes into a persevere / pivot /
  inconclusive call — counting only real problems people already spend time or
  money to solve. Run this first, before scoping or building anything.
- `validation/demand-test.md`: Manufacture and read a real willingness-to-*pay*
  signal before building the solution — the narrow validation experiment focused
  on commitment and money. Picks the method (pricing-anchored fake-door / pre-sell
  / deposit / paid-ad smoke test / concierge pre-sell / LOI), sets the pass
  threshold and minimum sample *before* spending a dollar, runs it without
  stranding any real buyer, and reads conversion-to-a-costly-action honestly
  (discounting warm-audience bias). Assumes the problem is already validated (run
  `validation/customer-interviews.md` first); a pass yields pre-customers and feeds
  `product/mvp-scoping.md`. Use when the question is "will they actually pay?",
  not "do they like it?".
- `validation/riskiest-assumption-test.md`: The meta-experiment-design prompt —
  surface the leap-of-faith assumptions a venture rests on (desirability /
  viability / feasibility), rank them by impact × uncertainty, isolate the single
  riskiest one, and design the cheapest experiment that would confirm or kill it
  with a pre-set threshold. Routes to the right specific test for execution
  (`customer-interviews` for problem risk, `demand-test` for willingness-to-pay,
  `pricing-validation` for price, `mvp-scoping` for a build experiment) and names
  the next assumption to test. Use when you're not sure *what* to validate first.
- `validation/pricing-validation.md`: Discover a defensible price and pricing
  model before hardcoding a number — value-based pricing anchored to the
  alternative's cost, the right pricing metric (per seat / usage / outcome),
  good-better-best tiering, and Van Westendorp-style sensitivity questions as a
  *soft* signal that must be confirmed with a real charge (routes to
  `validation/demand-test.md`). Distinct from demand-test, which proves they'll pay
  *at* a price; this finds the right price. Biased hard against underpricing.
- `validation/competitive-landscape.md`: Map the alternatives a customer uses
  today — always including "do nothing," a spreadsheet, or a manual workaround —
  their switching costs, and where the differentiated wedge / market whitespace is.
  Researches with cited sources, compares on customer-valued dimensions (not
  feature checklists), and feeds the competitive-alternative input to
  `marketing/positioning-messaging.md` and the current-alternative field of
  `marketing/customer-avatars.md`.
- `product/mvp-scoping.md`: Cut a *validated* idea down to the smallest product
  that tests one falsifiable hypothesis, with a pass/fail metric set before any
  build. Picks the lightest MVP type (landing page / fake-door / concierge /
  Wizard-of-Oz / single-feature / thin slice), states what's built vs. faked vs.
  deferred, and outputs an experiment brief whose "build for real" rows hand off
  to `product/gather-requirements.md`. The bridge from validation to engineering;
  it does not re-litigate whether the problem is real or gather full requirements.
- `product/gather-requirements.md`: Turn a rough product idea into a clear,
  testable requirements document.
- `product/activation-onboarding.md`: Design the first-run experience so a new
  user reaches the "aha" (first real value) as fast as possible, and define +
  instrument the activation metric that predicts retention. Identifies the aha
  moment, sets a behavioral activation metric within a time window (derived from
  where retained users diverge from churned ones), maps and de-frictions the path
  to it, and designs onboarding as a guided path — not a feature tour — with empty
  states as teachers and work done *for* the user. Buildable parts hand off to
  `product/gather-requirements.md` / the platform `feature-dev.md`; the metric and
  events feed analytics. Run it once there's a product to activate into.
- `product/metrics-instrumentation.md`: Choose the North Star metric (tied to
  delivered customer value), map the AARRR / pirate funnel (Acquisition,
  Activation, Retention, Referral, Revenue), pick the few actionable metrics that
  matter at the current stage, and specify what to instrument so the team learns
  instead of guessing. Anti-vanity throughout; situates the activation metric from
  `product/activation-onboarding.md` inside the whole funnel and routes the
  instrumentation build to `gather-requirements.md` / the platform `feature-dev.md`.
- `product/prioritization.md`: Triage a backlog / feature requests / competing
  bets against the *one* current bottleneck so the team works on the
  highest-leverage thing, not the loudest. Scores items with a fit-for-purpose
  framework (RICE / ICE / now-next-later), discounts impact by evidence, reframes
  feature requests into the underlying job, and defaults to cut — outputting a
  ranked now/next/later list tied to the bottleneck. Top items route to
  `gather-requirements.md` / the platform `feature-dev.md`.
- `ios/build-app.md`: Stand up an iOS app from approved requirements —
  architecture, project structure, shared infrastructure, and one proving
  end-to-end slice — then decompose the app into feature chunks and orchestrate
  parallel `ios/feature-dev.md` agents (partitioned by file ownership) to build
  them. Use for greenfield or a major re-architecture.
- `ios/feature-dev.md`: Add one feature to an existing iOS codebase, matching its
  architecture and conventions while minimizing blast radius. This is the unit of
  work `ios/build-app.md` fans out; also run it directly to add a single feature
  to a living codebase.
- `ios/bug-fix.md`: Diagnose and fix a bug in an existing iOS codebase.
  Reproduces the bug reliably, writes a failing test that pins the root cause,
  applies the minimal fix, and verifies no regression. Separate from
  `ios/feature-dev.md` — a bug fix must never silently include refactoring.
- `ios/security-audit.md`: Audit an iOS codebase for security issues. Report
  findings only.
- `ios/qa/qa-audit.md`: Audit iOS test coverage across all four testing quadrants
  (unit/component, example-driven acceptance, exploratory, and non-functional)
  and identify the production bug classes the current tests may miss. Report gaps
  only.
- `ios/qa/unit-testing.md`: Write the unit test suite for an iOS feature. Covers
  identifying humble shells vs. logic-bearing types, mock strategy, naming,
  Arrange/Act/Assert structure, boundary conditions, async/`@MainActor` patterns,
  date injection, in-memory stores, and the F.I.R.S.T. checklist. Companion to
  `ios/feature-dev.md` — run it when writing or reviewing unit tests for a feature.
- `ios/refactoring.md`: Audit an iOS codebase for maintainability and produce a
  prioritized, behavior-preserving refactoring plan, ranked by churn × complexity.
  Report a plan only; establishes the existing test suite (and any `ios/qa/qa-audit.md`
  findings) as the regression safety net the refactoring must keep green.
- `web/set-up-hosting.md`: Choose the simplest cost-effective hosting setup for
  a web app before implementation is locked to a platform.
- `web/build-app.md`: Stand up a Next.js app against confirmed requirements and a
  confirmed hosting target — architecture, project structure, shared
  infrastructure, and one proving end-to-end slice — then decompose the product
  into feature chunks and orchestrate parallel `web/feature-dev.md` agents
  (partitioned by file ownership) to build them. Use for greenfield or a major
  re-architecture.
- `web/feature-dev.md`: Add one feature to an existing Next.js codebase, matching
  its rendering model and conventions, with authorization and input validation
  built in. This is the unit of work `web/build-app.md` fans out; also run it
  directly to add a single feature to a living codebase.
- `web/bug-fix.md`: Diagnose and fix a bug in an existing Next.js/TypeScript
  codebase. Reproduces the bug reliably, writes a failing test that pins the
  root cause, applies the minimal fix, and verifies no regression. Covers
  Next.js-specific failure classes: stale cache, Server/Client boundary
  violations, ORM type coercion, webhook idempotency, and client self-rate-limiting.
- `web/security-audit.md`: Audit a Next.js codebase for security issues. Report
  findings only.
- `web/qa/qa-audit.md`: Audit Next.js and full-stack TypeScript test coverage
  across all four testing quadrants (unit/component, example-driven acceptance,
  exploratory, and non-functional), focusing on production bug classes like
  schema/type drift, Server/Client boundary failures, stale caches, webhook
  idempotency, and SDK-shape changes. Report gaps only.
- `web/qa/unit-testing.md`: Write the unit test suite for a Next.js/TypeScript
  feature. Covers identifying humble shells vs. logic-bearing units, mock strategy
  (fakes vs. real), naming, Arrange/Act/Assert structure, boundary conditions,
  Zod/Valibot schema testing, discriminated union coverage, async patterns, date
  injection, RTL component testing, and the F.I.R.S.T. checklist. Companion to
  `web/feature-dev.md` — run it when writing or reviewing unit tests for a feature.
- `web/refactoring.md`: Audit a Next.js/TypeScript codebase for maintainability
  and produce a prioritized, behavior-preserving refactoring plan, ranked by
  churn × complexity. Report a plan only; establishes the existing test suite
  (and any `web/qa/qa-audit.md` findings) as the regression safety net the
  refactoring must keep green.
- `marketing/customer-avatars.md`: Synthesize discovery evidence (interviews,
  sales calls, support tickets, analytics cohorts) into a small, distinct set of
  customer avatars — segmented by job-to-be-done and triggering situation, not
  demographics — each profiled with its current alternative, pains, where it
  already is, buying role, and objections, and every field tagged Evidence vs.
  Assumption. Names one primary beachhead and an explicit anti-avatar. The
  targeting foundation: its beachhead/anti-avatar feed
  `marketing/positioning-messaging.md`, and its "where they already are" fields
  feed channel and launch planning. Refuses to invent personas — run
  `validation/customer-interviews.md` first so the set rests on evidence.
- `marketing/positioning-messaging.md`: Define who the product is for, the
  competitive alternative it displaces (including "do nothing"), the unique
  wedge, the value it enables, and the market category — then write the message
  hierarchy (positioning statement, headline, outcome-tied value props with
  proof, objection handling, words to use/avoid). The foundational marketing
  artifact: its brief feeds the landing page, sales script, ads, and launch
  plan. Strongest after `validation/customer-interviews.md`, since it anchors
  every claim in a real customer and a real alternative rather than hype.
- `marketing/connect-ad-platforms.md`: Set up MCP connections to ad platforms
  (Meta/Facebook/Instagram, Google Ads, LinkedIn, etc.) so the LLM can research
  and — when explicitly gated — execute campaigns. Clarifies research-vs-execution
  intent and budget authority, recommends official servers over third-party,
  connects and verifies each platform **read-only first**, and enforces hard
  guardrails: platform-level budget caps, human approval before every spend or
  campaign mutation (no autonomous spend), least-privilege OAuth, and secrets kept
  out of chat/logs. An operational setup prompt, not a strategy one — its
  read-only connections feed the paid-ad smoke test in `validation/demand-test.md`
  and future channel/launch research; write access is used only inside an
  explicitly-gated execution step. Note: ad-platform MCP servers and their
  read/write capabilities change fast — the prompt verifies current capability
  against official docs rather than trusting a fixed list.
- `marketing/landing-page.md`: Design a conversion-first landing page — the
  argument, structure, and copy aimed at one visitor and one action. Owns
  message-match to the traffic source, the above-the-fold 5-second promise, the
  section skeleton (hero → problem → solution → proof → objections → CTA),
  outcome-led value props with proof placed where doubt peaks, inline objection
  handling, and friction-stripped CTAs — then *delegates the visual/component
  build to the `frontend-design` skill and the wiring/instrumentation to
  `web/feature-dev.md`*. Consumes `marketing/positioning-messaging.md` and the
  beachhead avatar from `marketing/customer-avatars.md`; outputs a content +
  conversion spec with an A/B test plan.
- `marketing/founder-led-sales.md`: Land the first ~10 customers by hand through
  personal, targeted founder outreach and a discovery-to-close motion that leads
  with diagnosis over pitch (Mom-Test discipline applied to selling). Consumes the
  beachhead/objections from `marketing/customer-avatars.md` and the message from
  `marketing/positioning-messaging.md`; qualifies hard, asks for the close, and
  mines every call for learning that feeds positioning and prioritization. The
  deliberately-unscalable first-sales playbook.
- `marketing/channel-strategy.md`: Choose *one* acquisition channel to test first
  and design the experiment with a kill criterion, instead of spreading thin.
  Brainstorms the full channel set, ranks to the few matched to where the beachhead
  avatar already is and the economics (CAC vs. price), and specs a cheap test with
  its metric, double-down threshold, and kill criterion. Paid channels route to
  `marketing/connect-ad-platforms.md` for execution under its spend guardrails.
- `marketing/launch-plan.md`: Sequence a launch (Product Hunt / Show HN / waitlist
  / email list / communities) as one concentrated moment with a single goal and
  metric — won in pre-launch prep. Picks channels where the beachhead avatar is,
  builds the audience and assets ahead, scripts the day-of run-of-show and fast
  founder response, and points the spike at a destination that converts. Consumes
  `marketing/positioning-messaging.md`, `marketing/customer-avatars.md`, and
  `marketing/landing-page.md`; sets honest spike-not-hockey-stick expectations.
- `marketing/lifecycle-email.md`: Design behaviorally-triggered email sequences
  across the lifecycle (welcome/onboarding → activation nudge → retention →
  win-back, plus key transactional moments). Each email serves the activation /
  retention metric, has one goal and one CTA, fires on behavior rather than a
  time-based blast, and measures the downstream action (not opens). Consumes the
  aha moment from `product/activation-onboarding.md` and the voice from
  `marketing/customer-avatars.md`; triggers route to product/engineering to wire up.

Shared reference (not a standalone prompt — read when a prompt points to it):

- `shared/testing-quadrants.md`: The canonical four-testing-quadrants model and
  context-driven-testing framing. Single source of truth for the QA-audit and
  refactoring-audit prompts so the iOS and web variants don't drift apart.
- `shared/founder-principles.md`: Platform-wide company-building principles
  (evidence over invention; the only real validation is a costly action; talk to
  customers and trust behavior over predictions; the smallest test that settles the
  question wins, fake before you build; define the metric and threshold before you
  run; vanity metrics are forbidden; narrow beats broad; lead with the customer's
  outcome in the customer's words; every claim needs proof and you never fabricate
  it; sequence the de-risking problem→demand→solution→activation→channel; make
  assumptions visible — separate evidence from inference from hope; treat every
  conclusion as a living hypothesis; a cheap "no" now beats an expensive one later;
  outward-facing and money-spending actions need a human gate). The business-side
  counterpart to the engineering-principles files: every prompt in `validation/`,
  `product/`, and `marketing/` instructs the LLM to read it first — it is the
  canonical lens for validation, product, and marketing decisions.
- `ios/common/engineering-principles.md`: Platform-wide iOS engineering principles
  (simplicity is the deliverable; names reveal intent — including scope-based
  name length, side-effect naming, and encapsulate/prefer-positive conditionals;
  functions do one thing at one level of abstraction; comments compensate for
  failure to express in code; leave it cleaner than you found it / Boy Scout
  Rule; dependency rule; don't marry the framework; lean on the language to hold
  boundaries; composition root; keep configurable data at high levels — never
  bake environment-specific values into the binary; if it hurts do it more
  frequently; build quality in — testing is not a phase; resilience; concurrency
  is a separate concern — synchronize as little as possible, prefer encapsulated
  locking; tell don't ask / Law of Demeter; feature envy — move methods to the
  type whose data they use; temporal coupling — expose execution order in the
  signature; encapsulate boundary conditions; use explanatory variables; return
  empty objects not nil / avoid sentinel error returns; true vs. accidental
  duplication; implement boundaries at the inflection point). Every `ios/` prompt
  instructs the LLM to read this file — it is the canonical lens for
  architecture, implementation, refactoring, and audit decisions.
- `web/common/engineering-principles.md`: Platform-wide web/Next.js engineering
  principles (simplicity is the deliverable; names reveal intent — including
  scope-based name length, side-effect naming, and encapsulate/prefer-positive
  conditionals; functions do one thing at one level of abstraction; comments
  compensate for failure to express in code; leave it cleaner than you found it /
  Boy Scout Rule; dependency rule; don't marry the framework; lean on the
  toolchain to hold boundaries; composition root; minimize dependencies; keep
  configurable data at high levels — never bake environment-specific values into
  the artifact, smoke-test config after deploy; if it hurts do it more frequently;
  build quality in — testing is not a phase; resilience; concurrency is a
  separate concern — synchronize as little as possible, prefer encapsulated
  locking; tell don't ask / Law of Demeter; feature envy — move functions to the
  module whose data they use; temporal coupling — expose execution order in types;
  encapsulate boundary conditions; use explanatory variables; return empty objects
  not null / avoid sentinel error returns; true vs. accidental duplication;
  implement boundaries at the inflection point). Every `web/` prompt instructs
  the LLM to read this file — it is the canonical lens for architecture,
  implementation, refactoring, and audit decisions.

Meta-prompt (library maintenance — not a product workflow):

- `improvement.md`: Reading-loop prompt that advances the library's knowledge
  base. It runs in two arcs against the current book in `reading_progress.json`,
  starting at the saved position: the **technical** loop reads all prompts in
  `ios/` and `web/` and applies engineering insights from the technical books
  (`knowledge-base/technical/`), and the **business** loop reads all prompts in
  `validation/`, `product/`, and `marketing/` plus `shared/founder-principles.md`
  and applies company-building insights from the business books
  (`knowledge-base/business/`). Use the block matching the current book's domain.
  At the end of each run it updates `reading_progress.json` and `README.md` if the
  prompt structure changed. Run this to improve the prompts as new material is
  read; do not run it as part of a product workflow.

## Recommended Workflow

For a new product:

1. De-risk before you build. Run `validation/customer-interviews.md` to confirm
   the problem is real, frequent, and painful enough that people already spend
   time or money on it. Don't skip to requirements on the strength of an idea you
   like — a false positive caught here is the cheapest one you'll ever catch.
   Then run `validation/demand-test.md` to prove they'll actually *pay* — a
   pricing-anchored fake-door, pre-sell, or paid-ad smoke test — before you commit
   to building. A pass here hands you pre-customers; a fail is the cheapest save
   you'll ever get.
2. Once the problem and demand are validated, run `product/mvp-scoping.md` to decide the
   smallest thing that tests your riskiest assumption, with a pass/fail metric
   set up front. It hands the "build for real" parts to the next step and keeps
   the faked/manual parts out of engineering.
3. Run `product/gather-requirements.md` on the parts the MVP actually needs built.
4. In parallel with build, run `marketing/customer-avatars.md` to synthesize your
   discovery into a small set of evidence-backed avatars with one beachhead, then
   `marketing/positioning-messaging.md` to lock the wedge; the avatar set feeds
   positioning, and the positioning brief feeds the landing page, sales, and launch.
5. For web apps, run `web/set-up-hosting.md` before the build prompt unless
   hosting has already been decided.
6. Run the platform `build-app.md` with the approved requirements. It stands up
   the foundation (architecture, shared infrastructure, and one proving
   end-to-end slice), then **decomposes the product into feature-sized chunks and
   orchestrates `feature-dev.md`** — fanning out independent chunks as parallel
   agents partitioned by file ownership, and serializing chunks that share files
   or depend on each other.
7. To add a feature later to the now-living codebase, run the platform
   `feature-dev.md` directly (it inherits the established architecture and
   conventions, so it confirms only what's genuinely unresolved). Run
   `qa/unit-testing.md` alongside or immediately after to write the unit tests
   for that feature — it takes the Phase 1 conditions of satisfaction as input.
8. After implementation, run the relevant security and QA audit prompts; reach
   for the refactoring audit when maintainability degrades.
9. With the product live, run `product/activation-onboarding.md` to drive new
   users to first value and instrument the activation metric, and
   `marketing/landing-page.md` to convert the traffic your positioning and
   channels send — both consume the briefs produced upstream.

For an existing app:

1. Choose the prompt that matches the immediate job: hosting, adding a feature
   (`feature-dev.md`), a major re-architecture (`build-app.md`), security audit,
   QA audit, or refactoring audit.
2. Fill in any context block at the top of the prompt.
3. Let the LLM inspect the repository before asking questions.
4. Preserve the prompt's stop points and approval gates.

When the job is improving maintainability of code that already works, run the
QA audit first and the refactoring audit second: the refactoring prompt consumes
the QA audit's coverage findings (or audits the codebase itself) to fix the set
of tests — unit, integration, E2E/browser, and any Markdown-described test cases —
that must keep passing, and treats untested hotspots as needing characterization
tests before they are safe to touch.

## How an LLM Should Execute These Prompts

Read the selected prompt from top to bottom before acting. Follow its phases in
order. Do not skip Phase 1, even if the user provides a detailed request.

For workflow prompts:

- Phase 1 is for clarifying the work. Ask the questions the prompt requires,
  then stop and wait.
- Phase 2 is for pressure-testing or researching current platform guidance.
  Prefer official docs and cite sources when a decision depends on current
  behavior.
- Phase 3 is a decision or architecture checkpoint. Present the recommendation
  or plan and wait for approval.
- Phase 4 is where implementation, final requirements, or setup instructions
  happen after approval.

For `build-app.md` (stand up an app, then delegate features):

- Phases 1–3 clarify requirements, research current guidance, and get the
  architecture signed off. Phase 4 builds **only** the foundation, the shared
  infrastructure, and one proving end-to-end slice — not every feature.
- Phase 5 decomposes the product into feature-sized chunks (each one a
  `feature-dev.md` unit of work), maps their dependencies and file ownership,
  then sequences and **fans them out as parallel `feature-dev.md` agents**.
  Independent chunks (disjoint files) run concurrently; chunks that share files
  or depend on each other are serialized so parallel agents never collide.
- Because the architecture was already approved in Phase 3, Phase 5 runs
  **automatically**: state the decomposition and parallelization plan for the
  record, then fan out the independent chunks without waiting for further
  approval (pause only for a genuinely unresolved chunk scope or shared-schema
  decision). After the fan-out, cross-review features against each other and run
  the full test suite on the integrated whole.

For `feature-dev.md` (add one feature to a living codebase):

- It assumes the architecture and conventions already exist (from `build-app.md`
  or an established repo). Phase 2 studies the codebase and mirrors its patterns
  rather than imposing new ones.
- When invoked as a chunk from `build-app.md`, it inherits the handoff context
  (architecture, reusable building blocks, acceptance criteria) and confirms only
  what is genuinely unresolved instead of re-asking settled questions.

For audit prompts (security, QA, and refactoring):

- Do not implement fixes, refactors, or tests.
- Inspect the codebase and report findings only.
- Use the output format defined by the prompt.
- Lead with severity, priority, impact, and concrete file references.
- The refactoring audit additionally reports a *behavior-preserving* plan: every
  proposed move must preserve observable behavior, is ranked by churn × complexity,
  and names the existing test(s) that protect it (or flags characterization tests
  as a prerequisite). Anything that would change behavior is out of scope for the
  refactor and must be called out as a separate task.

For all prompts:

- Make assumptions visible. Do not silently invent product, architecture,
  hosting, security, or compliance decisions.
- Prefer the simplest solution that satisfies the real requirement.
- Separate what is known from what is inferred from what still needs a human
  answer.
- Stop when the prompt says to stop. A good pause prevents expensive rework.

## Asking Human-Focused Requirements Questions

The user should not need to speak in engineering jargon to give useful answers.
Ask questions that are concrete, answerable, and tied to a decision you need to
make.

Good questioning rules:

- Ask one question at a time when exploring product requirements.
- Use multiple choice whenever possible.
- Put the recommended option first when you have enough context to recommend.
- Include an "I'm not sure" or "recommend for me" option when appropriate.
- Explain why the question matters in one short sentence.
- Ask about user outcomes before implementation details.
- Ask about real constraints: budget, timeline, launch date, compliance, team
  ownership, expected usage, and must-use systems.
- Avoid broad questions like "Any other requirements?" until the end of a
  section.
- Avoid jargon unless the user has already used it. If a technical term is
  necessary, pair it with plain language.
- Summarize the answer back before using it to make a major decision.

Use this shape:

```text
Question: Who is the first version for?

Why it matters: This decides which workflows are in scope for v1.

Options:
1. Internal team only
2. Existing customers
3. New public users
4. A specific pilot customer
5. Not sure - recommend based on the idea
```

For technical prompts, grouped questions are acceptable, but keep each question
short and decision-oriented. If the LLM can infer the answer from the codebase,
it should do that first and show the evidence instead of asking the user.

## Requirement Areas to Cover

When gathering requirements, make sure the user has answered the areas that
change scope or architecture:

- Problem, target users, and success criteria.
- Core v1 flows and what is explicitly out of scope.
- Conditions of satisfaction: concrete input-to-output examples that define
  "done" for each core behavior, including worst-case and best-case examples.
  These double as the acceptance tests.
- Ripple effects: other features, screens, shared data, caches, migrations, and
  integrations a change will touch, so design and tests account for them.
- Screens, states, and edge cases.
- Data stored, imported, exported, or deleted.
- Authentication, roles, permissions, and sensitive data.
- External integrations and whether they already exist.
- Performance, traffic, data volume, and availability expectations.
- Accessibility, localization, analytics, and observability needs.
- Testing expectations and the flows that must be verified before launch.
- Budget, timeline, maintainers, environments, and operational ownership.
- Compliance, security, privacy, and data-residency constraints.

If any of these do not matter for the specific task, say so briefly and move on.
Do not force irrelevant questions just to complete a checklist.

## Hosting-Specific Guidance

Hosting is a separate architecture decision for web apps. Do not choose hosting
inside `web/build-app.md`.

Run `web/set-up-hosting.md` when:

- The app's host is unknown.
- The app may need containers, workers, queues, websockets, cron, private
  networking, or long-running requests.
- The app may fit a managed Next.js platform, but cost, scale, compliance, or
  provider limits are unclear.
- Existing cloud infrastructure may matter.

Only run `web/build-app.md` after the hosting target and deployment model are
confirmed. The build prompt should then use that host's documented constraints
as implementation constraints.

## Approval Gates

Respect these gates unless the user explicitly overrides them:

- Product requirements: stop after clarifying questions, then stop again after
  pressure-testing scope.
- `validation/customer-interviews.md`: stop after clarifying the learning goal
  (riskiest belief + the decision the answers inform), then again after the
  interview script and commitment ask, before any fieldwork.
- `validation/demand-test.md`: stop after clarifying the costly buying signal and
  the decision it informs, then again on the offer, price, pass metric, and
  threshold, before spending on traffic or making any promise to a real person.
- `validation/riskiest-assumption-test.md`: stop after enumerating the
  assumptions, then again on which assumption is riskiest and which experiment +
  threshold will test it, before specifying or running anything.
- `validation/pricing-validation.md`: stop after clarifying value, alternative,
  and segment, then again on the pricing model and the test plan, before fielding
  any pricing question or charge.
- `validation/competitive-landscape.md`: no implementation gate (research/report),
  but stop after presenting the landscape map and recommended wedge for sign-off
  before writing the brief.
- `product/mvp-scoping.md`: stop after clarifying the hypothesis, then again on
  the scope cut AND the pass/fail metric + threshold, before anything is built.
- `product/activation-onboarding.md`: stop after clarifying the aha moment, then
  again on the activation metric AND the target path, before designing the flow.
- `product/metrics-instrumentation.md`: stop after clarifying the core value and
  bottleneck, then again on the North Star and the funnel metrics, before
  specifying instrumentation.
- `product/prioritization.md`: stop after clarifying the current bottleneck and
  candidates, then again on the ranked list and cut pile, before any item routes
  to build.
- `marketing/landing-page.md`: stop after clarifying the conversion goal and
  visitor, then again on the page structure and above-the-fold promise, before
  writing full copy.
- `marketing/connect-ad-platforms.md`: stop after clarifying which platforms,
  research-vs-execution intent, and budget authority, then again on the chosen
  servers and the guardrail model, before touching any configuration. Connect
  read-only first; any spend or campaign mutation is a separate, explicitly
  human-approved step — never autonomous.
- `marketing/customer-avatars.md`: stop after clarifying the evidence and the
  decision the avatars drive, then again on the candidate avatar set (count,
  distinguishing dimensions, proposed beachhead), before fleshing out full profiles.
- `marketing/positioning-messaging.md`: stop after clarifying the five
  positioning inputs, then again on the assembled positioning frame, before
  writing the messaging.
- `marketing/founder-led-sales.md`: stop after clarifying the target, offer, and
  prospect list, then again on the outreach + discovery plan, before any outreach.
- `marketing/channel-strategy.md`: stop after clarifying the avatar and economics,
  then again on the channel(s) to test and the kill criteria, before spending.
- `marketing/launch-plan.md`: stop after clarifying the launch goal and audience,
  then again on channels and the backward-planned timeline, before producing assets.
- `marketing/lifecycle-email.md`: stop after clarifying the activation/retention
  goal and lifecycle stages, then again on the sequence map, before writing copy.
- `build-app.md`: stop after clarifying questions, then again after the
  architecture plan. After that approval, Phase 5 proceeds automatically —
  decomposing the product into features and fanning out parallel `feature-dev.md`
  agents without a further gate (pausing only for a genuinely unresolved chunk
  scope or shared-schema decision).
- `feature-dev.md`: when run standalone, stop after clarifying questions, then
  again after the feature plan, before writing the implementation. When fanned
  out automatically as a chunk from `build-app.md`, it inherits the approved
  architecture and runs without re-gating.
- Hosting prompt: stop after unresolved decision questions, then stop again
  after the hosting recommendation.
- Audit prompts (security, QA, refactoring): no approval gate is needed before
  reporting findings or a plan, but do not fix, refactor, or write tests unless
  the user starts a separate implementation task.

## Final Output Standards

When producing a final artifact from these prompts:

- Be specific enough that another engineer can act without reading the chat.
- Include open questions and assumptions explicitly.
- Tie recommendations to user goals, code evidence, or official docs.
- Keep scope tight. Capture future ideas separately instead of smuggling them
  into v1.
- Prefer short, testable requirements over broad intent statements.
- Include verification expectations: tests, smoke checks, audits, or acceptance
  criteria.
