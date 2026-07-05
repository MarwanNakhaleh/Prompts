# Prompt Library Guide

This repository is a library of prompts for taking a product from a rough idea
all the way to revenue: validating that the problem and demand are real,
positioning it, turning it into requirements, implementation plans, production
code, and hosting decisions, and running post-build audits. The prompts are
written for an LLM that can inspect a codebase, ask clarifying questions,
research current platform guidance, and pause for human approval at the right
moments.

The work splits into two arcs that meet in the middle, with a third sitting on
top. The **company-building** prompts (`validation/`, `product/`, `marketing/`)
de-risk *what* to build and *who it's for* before and around the engineering. The
**engineering** prompts (`ios/`, `web/`) build, test, and harden it. Validation
feeds product scoping, product scoping feeds the build, and positioning feeds
go-to-market — so a prompt that ends by handing a clean brief to the next one is
doing its job.

Above both arcs sit the **leadership/organization** prompts (`ceo/`, `cto/`,
`cmo/`, `sales/`, `design/`), which direct *who runs the company and how the org
executes*: company and technical strategy, org design, hiring, the operating
cadence, fundraising, reliability, brand, the repeatable sales motion, and the
design of the product itself. `sales/` scales the founder-led motion into a
repeatable function, and `design/` finally gives UX and visual design its own
home. These are not a linear step in the build flow — they're invoked when an
organization-level job arises — and each reads the leadership spine
(`shared/leadership-principles.md`) first, plus the founder or engineering spine
wherever a customer-evidence or engineering judgment is involved.

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
- `web/seo-audit.md`: Audit a Next.js codebase and its production deployment for
  technical and on-page SEO, in pipeline order from the crawler inward —
  fetchability → indexability → canonicalization → metadata/schema → internal
  linking → performance → Search Console query diagnosis → competitive SERP
  analysis — always verifying against production HTML, not just source (a
  site-wide noindex or homepage-pointing canonical in a root layout invalidates
  all downstream work). Buckets GSC queries by intent (product / adjacent /
  mismatch) so zero-click impressions are diagnosed as a position-or-title
  problem rather than a positioning panic. Then remediates in tiers:
  engineering defects are fixed directly, outward-facing copy changes are
  human-gated, and net-new content opportunities hand off to
  `cmo/content-seo-strategy.md`. The engineering half of the organic engine;
  prioritizes fixes by the downstream action a page drives, not raw traffic.
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
  `marketing/connect-ad-platforms.md` for execution under its spend guardrails, and
  the content/ads/outreach it picks route to `marketing/content-engine.md`,
  `marketing/paid-ads.md`, and `marketing/founder-led-sales.md`.
- `marketing/lead-magnet.md`: Design a lead magnet — a complete solution to one
  narrow problem, free or low-cost, that engages a cold or expensive-offer audience
  and reveals the next problem the *core offer* solves. Picks the type (reveal-the-
  problem diagnosis / sample-or-trial / one-step-of-many), the delivery (software,
  info, service, physical), tests the name/headline (the highest-leverage decision),
  and demands give-away-the-secrets quality tied back to what you sell. Consumes
  `marketing/customer-avatars.md` and `marketing/positioning-messaging.md`; the magnet
  is the thing `marketing/content-engine.md`, `marketing/paid-ads.md`, and
  `marketing/founder-led-sales.md` advertise, captured via `marketing/landing-page.md`.
  A free take-rate is interest, not demand — confirm willingness to pay with
  `validation/demand-test.md`.
- `marketing/content-engine.md`: Build an audience-as-asset content machine on the
  one platform the beachhead avatar is on — every piece built to hook, retain, and
  reward, over-giving and under-asking, narrow-niche-first ("king of the puddle"),
  measured by audience *growth rate* and engaged leads rather than vanity follower
  counts. Sets the give:ask ratio and ask mechanics, commits a sustainable cadence,
  and produces the unit toolkit (topic/headline/format hooks, list/step/story
  retention). Asks point at `marketing/lead-magnet.md` or the core offer; the audience
  it builds is the warm list `marketing/lifecycle-email.md` and
  `marketing/founder-led-sales.md` work; the best pieces feed `marketing/paid-ads.md`.
- `marketing/paid-ads.md`: Make and scale paid ad campaigns — the creative
  (call-out + value + CTA), the targeting (lookalikes + filters), and the efficiency
  economics (lifetime-gross-profit-to-CAC ≥ ~3:1, 30-day payback, track→lose→print,
  kill losers / scale winners). Catches the *business-model-not-ad* problem before a
  dollar is wasted. The strategy-and-creative complement to
  `marketing/connect-ad-platforms.md`, which safely executes every dollar under its
  human-approval guardrails — **no autonomous spend.** Consumes
  `marketing/lead-magnet.md` or the core offer, the page from
  `marketing/landing-page.md`, creative from `marketing/content-engine.md`, and the
  LTGP:CAC math from `product/metrics-instrumentation.md`.
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
- `marketing/referral-program.md`: Turn delighted customers into the lowest-cost,
  highest-quality, exponentially-growing lead source — product first (build goodwill
  through six value levers), then the ask (treated as an offer: one/two-sided
  incentives sized to CAC, point-of-sale asks, events, unlockable bonuses). Refuses
  to bolt a referral hack onto an unremarkable product, measures referral *rate*
  against churn, and protects the referrer's relationship by never stranding a
  referred friend. Product improvements feed `product/prioritization.md` and
  `product/activation-onboarding.md`; the ask copy feeds `marketing/lifecycle-email.md`
  and `marketing/landing-page.md`; the referral-vs-churn economics feed
  `product/metrics-instrumentation.md`.

The `ceo/` prompts run the company-altitude jobs — strategy, story, money, people,
and the operating system (each reads `shared/leadership-principles.md` first):

- `ceo/company-strategy.md`: Turn raw company inputs into a real strategy — a small
  set of hard choices (where to play, how to win, what to deliberately *not* do)
  anchored to one hedgehog at the intersection of best-in-world × economic engine ×
  what the company is driven to do. Gathers the raw material one question at a time,
  pressure-tests the hedgehog and the few bets against the brutal facts, names the
  deliberate NOs and the disconfirming condition, and gates the strategy (a one-way
  door pointing the whole company) before writing the brief. Consumes
  `marketing/positioning-messaging.md` and `validation/competitive-landscape.md`;
  its brief sets the goals for `ceo/operating-cadence.md`, the story for
  `ceo/vision-narrative.md`, and the bottleneck `product/prioritization.md` ranks
  against.
- `ceo/vision-narrative.md`: Build the company-altitude narrative — the mission, the
  vision, and the "why now" — used to recruit, raise, and rally, kept grounded
  enough to sound inevitable rather than hyped. Clarifies the inputs, shapes and
  pressure-tests the arc (testing the "why now" hardest and killing every
  superlative), then writes the one-liner, short, and full narratives with a
  claims-and-proof ledger. The narrative is a trust act, so it's gated before any
  outward use. Consumes `ceo/company-strategy.md` (ladders to the hedgehog); feeds
  the investor story in `ceo/fundraising-narrative.md`, the recruiting story in
  `ceo/hiring-key-roles.md` and `ceo/culture-values.md`, and sits above
  `marketing/positioning-messaging.md`.
- `ceo/fundraising-narrative.md`: Run a raise as a deliberate sales process —
  narrative, deck arc, metrics, and a momentum-creating outreach plan — never as a
  moment of being judged. Researches current round/dilution/valuation norms and
  cites them, sizes the round to the milestones it must buy, pre-empts the diligence
  questions, and **gates every investor contact and every send behind explicit human
  approval** (no deck forwarded, no commitment made autonomously). Consumes
  `ceo/financial-model.md` (numbers, runway, use of funds) and
  `ceo/vision-narrative.md` (the story); a closed round feeds
  `ceo/board-investor-update.md` and resets `ceo/financial-model.md` and
  `ceo/hiring-key-roles.md`.
- `ceo/financial-model.md`: Build a driver-based financial model a team can steer by
  — the few real levers, honest unit economics (CAC, LTV, payback), and base/bull/
  bear scenarios that name what kills the company and when. Anti-vanity and
  anti-hockey-stick; tags every input observed/inferred/hoped and gates the drivers,
  economics, and steering dashboard before the model is used to raise, hire, or
  budget. Consumes `validation/pricing-validation.md`, `validation/demand-test.md`,
  `product/metrics-instrumentation.md`, and `ceo/company-strategy.md`; supplies the
  numbers to `ceo/fundraising-narrative.md`, `ceo/board-investor-update.md`,
  `cmo/budget-allocation.md`, and the runway constraint to `ceo/hiring-key-roles.md`
  and `ceo/operating-cadence.md`.
- `ceo/hiring-key-roles.md`: Hire a senior or first-critical role by the rule "first
  who, then what" — define the seat as a scorecard of outcomes (not a résumé), run a
  structured, evidence-gathering loop with real reference checks, and decide with a
  default-to-no-hire when the signal is mixed. **Gates the offer — a one-way,
  trust-laden act — behind explicit human approval.** Consumes `ceo/culture-values.md`
  (the behaviors tested) and `ceo/company-strategy.md` (which seats matter); hands
  function-specific loops to `cto/hire-engineers.md` and `sales/hire-train-reps.md`;
  comp draws against `ceo/financial-model.md` and the first-90-days criteria feed
  `ceo/operating-cadence.md`.
- `ceo/operating-cadence.md`: Install the lightest operating system that turns
  strategy into this week's action and feeds results back fast — goals that ladder
  to the bets, a meeting rhythm where each meeting has a distinct job, a metric
  review with trigger thresholds, and a decision/unblock model. Cuts cadence
  theater; gated before it becomes how the company runs. Consumes
  `ceo/company-strategy.md` (the bets) and the metrics from
  `product/metrics-instrumentation.md` and `ceo/financial-model.md`; produces the
  metric narrative for `ceo/board-investor-update.md` and the priority signal that
  frames `product/prioritization.md` each cycle.
- `ceo/hard-decision.md`: Work a high-stakes, often irreversible call — a pivot, a
  layoff, killing a line, firing an exec — by confronting the brutal facts while
  keeping faith (the Stockdale paradox), weighing options by reversibility, and
  hunting lead bullets over silver ones. Plans the communication as half the
  decision. **Gates any irreversible or outward-facing action (the layoff, firing,
  pivot, or announcement) behind explicit human approval.** A pivot routes back into
  `ceo/company-strategy.md`; a layoff updates `ceo/financial-model.md` and
  `ceo/operating-cadence.md`; any of it becomes honest material for
  `ceo/board-investor-update.md`. Reaches into `shared/founder-principles.md` for
  pivot-or-persevere.
- `ceo/culture-values.md`: Name the few behaviors that actually make this company win
  — defined as things people *do*, concrete enough to hire, promote, reward, and fire
  against — and operationalize them into the systems that shape behavior, refusing
  poster values that cost nothing. Confronts the brilliant-jerk test directly; gated
  before rollout, because announcing values you won't enforce spends credibility you
  don't get back. Consumes `ceo/company-strategy.md`; supplies the behavioral screens
  to `ceo/hiring-key-roles.md`, `cto/hire-engineers.md`, and
  `sales/hire-train-reps.md`, and the bar `ceo/operating-cadence.md` and
  `ceo/hard-decision.md` enforce.
- `ceo/board-investor-update.md`: Turn the board meeting and the investor update into
  a tool, not a performance — lead with the honest narrative (bad news near the top,
  owned), show real metrics, frame the one or two decisions you need the room's
  judgment on, and make every ask assignable to a person. **Gates the send or
  circulation behind explicit human approval**, since it shapes investor trust.
  Consumes `ceo/financial-model.md` (metrics, runway) and `ceo/operating-cadence.md`
  (goals graded); decisions route into `ceo/hard-decision.md`, a fundraise signal
  connects to `ceo/fundraising-narrative.md`, and the narrative ladders to
  `ceo/vision-narrative.md`.

The `cto/` prompts set technical direction and run the engineering organization
above any single app (each reads `shared/leadership-principles.md` plus the relevant
`*/common/engineering-principles.md` first):

- `cto/technical-strategy.md`: Set technical direction across many teams and systems
  — the altitude above any one app — as a small set of deliberate bets tied to the
  business bet, an architecture north star, and an explicit list of what *not* to
  build. Locates the real constraint with delivery evidence (the four DORA-style
  signals), separates core from context, and states the thesis with a disproof
  condition; gated as a multi-team one-way-ish door. Consumes
  `ceo/company-strategy.md`; hands new systems to `web/build-app.md` /
  `ios/build-app.md`, cross-team sequencing to `cto/tech-roadmap.md`, team shape to
  `cto/eng-org-design.md`, contested core-vs-context calls to `cto/build-vs-buy.md`,
  and the reliability bet to `cto/reliability-incident.md`.
- `cto/eng-org-design.md`: Design the engineering organization as system design —
  draw team boundaries first (Conway's law used deliberately) so the architecture you
  want falls out of them. Sorts work into stream-aligned / platform / enabling /
  complicated-subsystem teams, bounds each team's cognitive load, picks the
  interaction modes, and makes "you build it, you run it" real. Gated as a
  one-way-ish door that lands on people's careers. Consumes
  `cto/technical-strategy.md`; the open seats feed `cto/hire-engineers.md`, the
  on-call and ownership model feeds `cto/reliability-incident.md`, and the team
  boundaries reconcile with the strategy's north star.
- `cto/tech-roadmap.md`: Sequence *technical* investment across teams — feature
  enablement, platform, reliability, and debt paydown — with dependencies made
  visible, the investment balance made deliberate (not defaulting to all-features),
  and every item priced against what it displaces. The technical-investment altitude
  above `product/prioritization.md`, which it consumes as one input. Consumes
  `cto/technical-strategy.md` and `cto/eng-org-design.md`; routes new systems to
  `web/build-app.md` / `ios/build-app.md`, single features to the platform
  `feature-dev.md`, debt to `web/refactoring.md` / `ios/refactoring.md`, and
  reliability to `cto/reliability-incident.md`.
- `cto/build-vs-buy.md`: Make a build-vs-buy call on total cost of ownership over the
  life of the thing, on whether it touches the company's differentiation (build your
  core, buy your context), and on reversibility — refusing both the build-everything
  and outsource-everything reflexes. Prices "free" OSS and vendor lock-in honestly
  and designs for exit behind an interface you own. **Gates the irreversible
  commitment — signing a contract, committing a budget — behind explicit human
  approval, preferring a reversible pilot first.** If build, routes to
  `web/build-app.md` / `ios/build-app.md` or the platform `feature-dev.md`; if buy,
  the integration to `feature-dev.md` behind the owned interface; vendor compliance
  to `cto/security-compliance-program.md`.
- `cto/hire-engineers.md`: Build the engineering hiring *system* — a leveling ladder,
  a scorecard of outcomes, a structured work-sample loop that predicts on-the-job
  performance, a bar-raiser guarding consistency, and a closing motion — cutting
  trivia and unstructured culture chats, and defaulting to no-hire on a mixed signal.
  The engineering specialization of `ceo/hiring-key-roles.md`; the *system* gets a
  gate even though each interview doesn't. Consumes `cto/eng-org-design.md` (the
  seats) and `ceo/culture-values.md` (the behaviors); a hire's onboarding ties to
  `cto/eng-org-design.md` and `cto/reliability-incident.md`.
- `cto/delivery-pipeline.md`: Treat the path from commit to happy user as the most
  important product engineering owns — trunk-based development, an automated pipeline
  that builds the artifact once and promotes it, progressive delivery, rehearsed
  rollback, and the four DORA metrics — applying "if it hurts, do it more often."
  Gated before changing how every deploy works. Test-coverage gaps route to
  `web/qa/qa-audit.md` / `ios/qa/qa-audit.md` and the unit-testing prompts; pipeline/
  infra changes to the platform `feature-dev.md`; the reliability side to
  `cto/reliability-incident.md`; team ownership from `cto/eng-org-design.md`.
- `cto/reliability-incident.md`: Stand up reliability as a discipline rather than a
  heroic habit — SLIs from the user journey in, SLOs and an error budget that
  arbitrates speed vs. stability, a humane on-call, incident command, and blameless
  postmortems that produce systemic fixes. **Report + plan first: assess and propose,
  get sign-off, and implement only on a separate, explicitly-gated task** (putting
  people on a pager and committing to an SLO are not analysis side effects).
  Implementation routes to the platform `feature-dev.md`; rollback-on-budget-burn to
  `cto/delivery-pipeline.md`; on-call from `cto/eng-org-design.md`; architectural
  causes to `cto/architecture-review.md` and `cto/tech-roadmap.md`.
- `cto/architecture-review.md`: Review a large system or RFC to make the design
  better and the risks visible — pressure-testing failure modes, the next order of
  magnitude, maintainability, data consistency, and always the simpler alternative
  the author skipped — refusing both the ego pass and the rubber stamp. **Report-only:
  produce findings by severity and a proceed / proceed-with-changes / go-back call; do
  not implement or rewrite the design in the same run.** If it proceeds, build routes
  to `web/build-app.md` / `ios/build-app.md` or the platform `feature-dev.md` carrying
  the required changes; deep scaling to `cto/scaling-plan.md`; security to
  `cto/security-compliance-program.md`; checked against the north star in
  `cto/technical-strategy.md`.
- `cto/scaling-plan.md`: Plan a system to the next order of magnitude (10x, not 10%)
  without guessing — capture a baseline first, build a whole-system load model (app,
  datastore, network, per-instance limits, third-party quotas), find the one binding
  constraint with evidence, and weigh each scaling lever's real consistency/operational
  cost. Names the scaling work deliberately *not* done yet. Gated on the one-way-door
  commitments (a partitioning scheme, a datastore swap). Implementation routes to the
  platform `feature-dev.md`; the SLOs it must hold come from
  `cto/reliability-incident.md`; a major boundary reshape feeds
  `cto/architecture-review.md`.
- `cto/security-compliance-program.md`: Stand up security and compliance as an
  ongoing program — a lightweight threat model, the few controls that move risk at
  this stage, least-privilege identity and secrets posture, vendor/data-flow risk,
  and a framework decision only if a real driver requires it — refusing cargo-culted
  enterprise process and treating a certificate as security. **Report + plan first; a
  standing commitment (adopting a framework, signing a customer security obligation,
  committing budget) is gated separately as an outward-facing promise.** Consumes the
  point-in-time `web/security-audit.md` / `ios/security-audit.md` as recurring inputs;
  control implementation to the platform `feature-dev.md`; CI security gates to
  `cto/delivery-pipeline.md`; control ownership onto `cto/eng-org-design.md`.

The `cmo/` prompts run marketing at the function altitude — the motion, the brand,
the demand engine, and the budget (each reads `shared/leadership-principles.md` and
`shared/founder-principles.md` first):

- `cmo/gtm-strategy.md`: Choose the *one* dominant go-to-market motion (product-led /
  sales-led / community-led / marketplace) the business is built to win on, sequence
  the segments behind a single beachhead, and set the funnel plan and budget shape —
  refusing the portfolio of half-run motions and letting the economics veto the
  fashionable choice. **Gated before any budget is committed, any campaign launches,
  or any outreach goes out.** Consumes `marketing/positioning-messaging.md` and
  `marketing/customer-avatars.md`; routes channel tests to
  `marketing/channel-strategy.md`, the sales motion to `sales/sales-playbook.md` and
  `marketing/founder-led-sales.md`, demand scaling to `cmo/demand-generation.md`,
  content to `cmo/content-seo-strategy.md`, brand to `cmo/brand-strategy.md`,
  measurement to `cmo/marketing-analytics.md`, and dollars to
  `cmo/budget-allocation.md`.
- `cmo/brand-strategy.md`: Build the company-altitude brand as the promise the
  company makes and keeps — the positioning, the few associations worth owning, the
  story (customer as hero), and a voice the whole team can write in — refusing
  logo-worship and borrowed grandeur, and backing every association with a behavior.
  **Outward brand expression (a public manifesto, a rebrand, a new claim) is
  human-gated.** Sits above (never overrides) `marketing/positioning-messaging.md`;
  consumes `marketing/customer-avatars.md`; sets the tone for
  `cmo/content-seo-strategy.md`, `marketing/landing-page.md`,
  `marketing/lifecycle-email.md`, `cmo/pr-influencer-community.md`, and
  `marketing/launch-plan.md`.
- `cmo/demand-generation.md`: Build the demand engine that *scales* a channel already
  proven (it won't scale a loss) — find the power-law channel, push it to its
  efficient frontier under a CAC/LTV/payback ceiling, reserve a disciplined experiment
  slice with kill criteria, and build loops over funnels. **No autonomous spend —
  every dollar is human-gated and paid execution inherits the
  `marketing/connect-ad-platforms.md` guardrails.** Consumes the proven first channel
  from `marketing/channel-strategy.md`; nurture to `marketing/lifecycle-email.md`;
  measurement to `cmo/marketing-analytics.md`; dollars to `cmo/budget-allocation.md`;
  sits inside the motion from `cmo/gtm-strategy.md`.
- `cmo/content-seo-strategy.md`: Build the content/organic engine as a compounding
  asset starting from the buyer's jobs, not the keyword tool — a few owned topic
  clusters tied to the unfair angle, a sustainable production system, distribution
  planned with each piece, and measurement of the downstream action (not pageviews).
  **Outward publishing and paid amplification are human-gated.** Consumes the avatar's
  jobs from `marketing/customer-avatars.md` and the voice from `cmo/brand-strategy.md`;
  feeds `marketing/lifecycle-email.md` and `marketing/landing-page.md`; sits inside
  `cmo/gtm-strategy.md` and reports into `cmo/marketing-analytics.md`.
- `cmo/marketing-analytics.md`: Build marketing measurement that drives reallocation —
  the few behavioral metrics per funnel stage, attribution that's honest about what it
  can't see (triangulation over a prettier last-click chart), and CAC/LTV/payback by
  channel (never only blended). Gated before instrumentation is built, and **no
  flattering-but-false number ships upward as fact**. Consumes the funnel from
  `product/metrics-instrumentation.md` and the channels from `cmo/demand-generation.md`
  / `cmo/gtm-strategy.md`; the instrumentation build routes to
  `product/gather-requirements.md` / the platform `feature-dev.md`; feeds
  `cmo/budget-allocation.md`.
- `cmo/pr-influencer-community.md`: Earn attention rather than buy it across three
  motions — PR (an honestly newsworthy angle), influencers/creators (borrowed trust
  that's real, not bought), and community (a durable owned audience worth belonging
  to) — refusing astroturf, fake reviews, and paid placement disguised as editorial.
  **Every pitch, partnership, community launch, and spend is human-gated.** Consumes
  `cmo/brand-strategy.md`, `marketing/positioning-messaging.md`, and
  `marketing/customer-avatars.md`; feeds the launch-moment push in
  `marketing/launch-plan.md` and builds the owned audience `cmo/demand-generation.md`
  and `marketing/lifecycle-email.md` reach for free; reports into
  `cmo/marketing-analytics.md`.
- `cmo/budget-allocation.md`: Allocate the marketing budget like a portfolio — by
  expected payback and reversibility, funding proven channels to (not past) their
  efficient frontier, reserving a disciplined experiment slice with kill criteria, and
  reallocating on payback evidence by a rule set in advance. **Every committed dollar
  is human-gated; paid execution inherits the `marketing/connect-ad-platforms.md`
  guardrails (no autonomous spend), and one-way-door commitments earn a firmer gate.**
  Consumes CAC/payback from `cmo/marketing-analytics.md`, the budget envelope from
  `ceo/financial-model.md`, and the channels from `cmo/demand-generation.md` /
  `cmo/gtm-strategy.md`; paid execution routes to `marketing/connect-ad-platforms.md`.

The `sales/` prompts scale the founder-led motion into a repeatable function (each
reads `shared/leadership-principles.md` and `shared/founder-principles.md` first, and
follows the first-deals motion in `marketing/founder-led-sales.md`):

- `sales/sales-playbook.md`: Codify the proven founder-led motion into a playbook
  another person can run — stage definitions with buyer-commitment exit criteria, a
  qualification framework adapted (not cargo-culted) to this deal, and an
  outcome-anchored value narrative — never codifying a step that only worked because
  the founder was in the room. The scaling of `marketing/founder-led-sales.md`. Gated
  before it becomes the thing reps run; nothing points at a real prospect without
  approval. Consumes `marketing/positioning-messaging.md`,
  `marketing/customer-avatars.md`, and the founder-led learnings; feeds
  `sales/hire-train-reps.md`, `sales/discovery-demo.md`,
  `sales/outbound-prospecting.md`, and `sales/pipeline-forecast.md`.
- `sales/discovery-demo.md`: Make the diagnose-before-propose half of selling
  teachable — a problem-first question flow anchored in past behavior (with an explicit
  bad-questions list), discovery that doubles as qualification, and a tailored demo
  that proves the *one* outcome the buyer named rather than touring features. Treats a
  compliment as a warning, not a win. Gated before it's run on real buyers; no demo
  misrepresents the product to manufacture an "aha." Details two stages of
  `sales/sales-playbook.md`; new language rolls up to
  `marketing/positioning-messaging.md`; the qualification it surfaces feeds
  `sales/pipeline-forecast.md`.
- `sales/outbound-prospecting.md`: Scale the personal-outreach half of
  `marketing/founder-led-sales.md` into a system that doesn't become spam — a sharp
  ICP and a real trigger, a list of named humans, sequences personalized by research
  (not merge tokens), and value-adding follow-up, measured by positive replies (not
  sends). **Every send is human-gated; start with a small approved batch, read the
  metrics, then scale — never autonomous blasting.** Booked meetings hand to
  `sales/discovery-demo.md` and the playbook; objections roll up to
  `marketing/positioning-messaging.md`; funnel metrics feed
  `sales/pipeline-forecast.md`; paid amplification routes through
  `marketing/connect-ad-platforms.md` under its spend guardrails.
- `sales/hire-train-reps.md`: Hire the first reps to *scale* a proven motion, not
  discover one — a rep scorecard, a comp/quota/OTE plan designed not to reward bad-fit
  closes, and a ramp built on the playbook — and may tell you *not to hire yet* if the
  motion isn't repeatable. The sales sibling of `ceo/hiring-key-roles.md` and
  `cto/hire-engineers.md`. **The offer, comp plan, and any verbal commitment are
  human-gated one-way doors.** The rep ramps on `sales/sales-playbook.md`,
  `sales/discovery-demo.md`, and `sales/outbound-prospecting.md`; pipeline flows into
  `sales/pipeline-forecast.md`; quota plans inform `ceo/operating-cadence.md`.
- `sales/pipeline-forecast.md`: Run the pipeline review and forecasting discipline
  that makes a number mean something — stage hygiene locked to buyer-commitment exit
  criteria, conversion math instead of eyeballing, a weighted commit/best-case/coverage
  forecast, and a review cadence where bad news travels up safely. Refuses
  raw-pipeline vanity and moving goalposts. **A forecast going into a board update or a
  spend decision is an outward-facing commitment — human-gated and defensible.** Stage
  definitions from `sales/sales-playbook.md`, qualification from
  `sales/discovery-demo.md`; feeds `ceo/operating-cadence.md` and the revenue section
  of `ceo/board-investor-update.md`; attainment feeds back into
  `sales/hire-train-reps.md`.
- `sales/objection-negotiation.md`: Build the objection-handling and negotiation
  discipline that closes without giving the company away — surface the *real* objection
  before answering the stated one, defend price by re-anchoring to the outcome (never a
  reflexive discount), reverse risk instead of dropping price, and trade every
  concession for a commitment. Refuses to rescue a bad-fit deal with a discount. **A
  signed contract, a discount, or a custom term is a human-gated one-way door that sets
  precedent for every deal after.** Consumes `marketing/positioning-messaging.md`,
  `marketing/founder-led-sales.md`, and `validation/pricing-validation.md`; plugs into
  the late stages of `sales/sales-playbook.md` and `sales/pipeline-forecast.md`;
  recurring deal-killers feed product prioritization.

The `design/` prompts give UX and visual design its own home — the flows, the system,
and the review (each reads `shared/leadership-principles.md` and
`shared/founder-principles.md` first):

- `design/ux-flows.md`: Design the flows and information architecture for the user's
  job, not the screen — starting from the job-to-be-done and the aha moment, reaching
  for proven patterns over novel invention, designing every state (empty as teacher,
  loading, error, success), and counting steps to first value. Gated on the IA and the
  flows before the full spec. Consumes `product/gather-requirements.md`,
  `marketing/customer-avatars.md`, and `product/activation-onboarding.md`; hands the
  visual language to `design/design-system.md`, the production visual build to the
  `frontend-design` skill, and the wiring to `web/feature-dev.md` /
  `ios/feature-dev.md`.
- `design/design-system.md`: Build the visual language a product is assembled from —
  tokens (primitive → semantic), a small deliberate component set with every state and
  variant, and a distinctive identity that fits the user's job — refusing generic AI
  aesthetics and baking accessibility in from the first token. Gated on the identity
  direction and token foundations before the full spec. Consumes `design/ux-flows.md`
  (the components the flows need), `marketing/positioning-messaging.md`, and
  `marketing/customer-avatars.md`; hands the coded build to the `frontend-design` skill
  (or the figma skills) and the wiring to `web/feature-dev.md` / `ios/feature-dev.md`.
- `design/usability-review.md`: Review an interface that already exists against
  usability heuristics and WCAG — walking each critical flow as the user, auditing the
  off-happy-path states, running the accessibility pass, and separating observed from
  inferred from needs-a-usability-test. **Report-only: diagnose and prioritize by
  impact on the job and on access; do not redesign in the same run.** Fixes route as
  separate tasks to `design/ux-flows.md` (flow/IA), `design/design-system.md` (systemic
  component/token defects), and `web/feature-dev.md` / `ios/feature-dev.md` (build).
  Consumes `marketing/customer-avatars.md` and `product/activation-onboarding.md`.

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
- `shared/leadership-principles.md`: Platform-wide leadership, management, and
  strategy principles (first who, then what; Level-5 leadership — ambition for the
  mission, humility about yourself; confront the brutal facts while keeping faith
  you'll prevail / the Stockdale paradox; find the hedgehog — one simple idea at the
  intersection; turn the flywheel, don't chase the miracle moment; management is a
  learnable skill, not a prize for tenure; delegate the outcome and the context,
  never abdicate the accountability; clarity is a kindness — over-communicate the
  why; culture is what you reward, tolerate, and walk past — not what's on the wall;
  protect focus — the scarcest act is choosing what *not* to do; decide by
  reversibility — speed on two-way doors, rigor on one-way doors; there are no silver
  bullets, only lead bullets; have a definite plan, not vague optimism; build the
  machine, not the output — systems over heroics; install an operating rhythm that
  turns strategy into action and back; make assumptions visible — separate known from
  inferred from hoped; outward-facing and irreversible actions need a human gate). The
  organizational counterpart to `shared/founder-principles.md` (which guards the
  learning) and the engineering-principles files (which guard the code): every prompt
  in `ceo/`, `cto/`, `cmo/`, `sales/`, and `design/` instructs the LLM to read it
  first — it is the canonical lens for the decisions *about the organization and its
  direction* that sit on top of both. Where a judgment is about validation, pricing,
  or positioning it defers to `shared/founder-principles.md`; where it's about
  architecture or implementation it defers to the engineering principles.
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
10. To get customers flowing, work the acquisition engine. Pick *one* channel with
    `marketing/channel-strategy.md`, build the free `marketing/lead-magnet.md` that
    channel advertises, and run it through the matching method:
    `marketing/content-engine.md` (audience-building on one platform),
    `marketing/paid-ads.md` (spend, executed only under
    `marketing/connect-ad-platforms.md`'s human-gated guardrails), or
    `marketing/founder-led-sales.md` (by hand). Nurture the audience with
    `marketing/lifecycle-email.md`, and once the product is genuinely good enough to
    earn it, compound growth with `marketing/referral-program.md`. Dominate one
    channel before adding the next.

For an existing app:

1. Choose the prompt that matches the immediate job: hosting, adding a feature
   (`feature-dev.md`), a major re-architecture (`build-app.md`), security audit,
   QA audit, or refactoring audit.
2. Fill in any context block at the top of the prompt.
3. Let the LLM inspect the repository before asking questions.
4. Preserve the prompt's stop points and approval gates.

The leadership prompts (`ceo/`, `cto/`, `cmo/`, `sales/`, `design/`) are not a
linear step in the build flow — they're invoked when an *organization-level* job
arises: set the company or technical strategy, design the org, run a raise, install
the operating cadence, scale the system, build the sales or marketing engine, or
design the UX. Each cites the leadership spine (`shared/leadership-principles.md`)
first, and where a customer/evidence or engineering judgment is involved it also
reads the founder spine (`shared/founder-principles.md`) or the relevant
engineering-principles file. Several are report- or plan-only in a single run
(`cto/architecture-review.md`, `cto/reliability-incident.md`,
`cto/security-compliance-program.md`, `design/usability-review.md`), like the audit
prompts — they diagnose and recommend, and implementation is a separate, gated task.

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

Leadership (`ceo/`, `cto/`, `cmo/`, `sales/`, `design/`) — each reads
`shared/leadership-principles.md` first, and outward-facing or irreversible actions
need a human gate:

- `ceo/company-strategy.md`: stop after clarifying the raw material (the hedgehog
  inputs and the brutal facts), then again on the hedgehog and the few bets, before
  writing the final strategy — a one-way door pointing the whole company.
- `ceo/vision-narrative.md`: stop after clarifying the narrative inputs, then again
  on the story arc; the narrative goes out only after explicit approval, because
  recruiting and raising on a story is a trust act.
- `ceo/fundraising-narrative.md`: stop after clarifying the raise, then again on the
  narrative, deck arc, and process; no investor is contacted, no deck sent, and no
  commitment made without explicit human approval — every investor email is gated.
- `ceo/financial-model.md`: stop after clarifying the drivers, then again on the
  drivers and unit economics, before the model is used to raise, hire, or budget.
- `ceo/hiring-key-roles.md`: stop after clarifying the seat, then again on the
  scorecard and interview loop; no offer (verbal or written) is extended without
  explicit human approval.
- `ceo/operating-cadence.md`: stop after clarifying what the system must connect,
  then again on the operating-system design, before it becomes how the company runs.
- `ceo/hard-decision.md`: stop after confronting the brutal facts, then again on the
  framing and options; no irreversible or outward-facing action (a layoff, firing,
  public pivot, or committed bet) executes without explicit human approval.
- `ceo/culture-values.md`: stop after clarifying the real culture, then again on the
  few behaviors, then again on the values AND the enforcement mechanics, before
  rollout.
- `ceo/board-investor-update.md`: stop after clarifying the period and the ask, then
  again on the narrative and deck; the update is not sent or circulated without
  explicit human approval.
- `cto/technical-strategy.md`: stop after clarifying the business before the
  architecture, then again on the proposed strategy (a multi-team one-way-ish door),
  before the handoff.
- `cto/eng-org-design.md`: stop after clarifying before redrawing boundaries, then
  again on the org design; reorganizing people needs an explicit human decision.
- `cto/tech-roadmap.md`: stop after clarifying the horizon and the investment
  balance, then again on the sequence AND the cut pile, before work commits.
- `cto/build-vs-buy.md`: stop after clarifying the strategic stakes, then again on
  the recommendation; signing a contract or committing a budget needs explicit human
  approval, with the reversible path (a pilot, a short contract) preferred first.
- `cto/hire-engineers.md`: stop after clarifying the seat and the bar, then again on
  the hiring system before the loop runs — the system is gated even though each
  individual interview is not.
- `cto/delivery-pipeline.md`: stop after clarifying the current pipeline and the
  pain, then again on the pipeline and metrics plan, before changing how every deploy
  works.
- `cto/reliability-incident.md`: report + plan first — present the assessment and the
  proposed system for sign-off; implementation is a separate, explicitly-gated task,
  because putting people on a pager and committing to an SLO are not analysis side
  effects.
- `cto/architecture-review.md`: report-only — produce findings by severity and a
  proceed / proceed-with-changes / go-back call; do not implement or rewrite the
  design in the same run.
- `cto/scaling-plan.md`: stop after clarifying the target and the evidence, then
  again on the scaling plan, before committing a one-way-door change (a partitioning
  scheme, a datastore swap).
- `cto/security-compliance-program.md`: report + plan first — sign off on the
  program; any standing commitment (adopting a framework, a customer security
  obligation, a budget) is a separately human-gated step.
- `cmo/gtm-strategy.md`: stop after clarifying the motion, segment, and economics,
  then again on the GTM plan and budget shape, before any budget is committed, any
  campaign launches, or any outreach goes out.
- `cmo/brand-strategy.md`: stop after clarifying what the brand must carry, then
  again on the brand platform; any outward brand expression (a manifesto, a rebrand,
  a new claim) is human-gated.
- `cmo/demand-generation.md`: stop after clarifying the proven channel and the
  economics, then again on the engine and the spend gates; no autonomous spend —
  every committed dollar is human-gated and paid execution inherits the
  `marketing/connect-ad-platforms.md` guardrails.
- `cmo/content-seo-strategy.md`: stop after clarifying the jobs and the goal, then
  again on the topic strategy and distribution; outward publishing and paid
  amplification are human-gated.
- `cmo/marketing-analytics.md`: stop after clarifying the decisions and the data,
  then again on the metric set and the attribution approach; no flattering-but-false
  number ships upward as fact.
- `cmo/pr-influencer-community.md`: stop after clarifying the angle and the audience,
  then again on the earned-attention plan; every pitch, partnership, community
  launch, and spend is human-gated.
- `cmo/budget-allocation.md`: stop after clarifying the budget and the returns, then
  again on the allocation and reallocation rules; every committed dollar is
  human-gated (no autonomous spend), and one-way-door commitments earn a firmer gate.
- `sales/sales-playbook.md`: stop after clarifying the motion you're codifying, then
  again on the assembled playbook; nothing points at a real prospect (no outreach,
  template, or sequence) without explicit human approval.
- `sales/discovery-demo.md`: stop after clarifying the buyer and the diagnosis, then
  again on the discovery flow and demo structure, before any call is run on a real
  buyer.
- `sales/outbound-prospecting.md`: stop after clarifying the ICP and the trigger,
  then again on the sequence and the gating metrics; no message sends and no list is
  contacted without explicit human approval — start with a small approved batch.
- `sales/hire-train-reps.md`: stop after clarifying readiness and the seat (which may
  be "don't hire yet"), then again on the scorecard, comp plan, and ramp; no offer or
  comp plan reaches a candidate without explicit human approval.
- `sales/pipeline-forecast.md`: stop after clarifying the pipeline and what the
  forecast is for, then again on the forecast discipline; any forecast leaving the
  room (a board update, a spend or hiring decision) is a human-gated, defensible
  commitment.
- `sales/objection-negotiation.md`: stop after clarifying the real objections and
  what's negotiable, then again on the objection and negotiation system; no contract,
  discount, or custom term is offered without explicit human approval.
- `design/ux-flows.md`: stop after clarifying the job and context, then again on the
  IA and the flows, before writing the full spec.
- `design/design-system.md`: stop after clarifying the audience and scope, then again
  on the identity direction and token foundations, before producing the full
  component spec.
- `design/usability-review.md`: report-only — diagnose and prioritize the findings;
  do not redesign in the same run, and route the fixes out as separate tasks.

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
