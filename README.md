# Prompt Library Guide

This repository is a library of prompts for turning product ideas into
requirements, implementation plans, production code, hosting decisions, and
post-build audits. The prompts are written for an LLM that can inspect a
codebase, ask clarifying questions, research current platform guidance, and
pause for human approval at the right moments.

Use the prompts as workflows, not as passive reference docs. Each prompt defines
a role, the required input, the order of work, and the points where the LLM must
stop and wait for the user.

The build prompts (`build-app.md`) stand up an app's foundation and then **delegate
individual features to a dedicated feature-development prompt** (`feature-dev.md`),
fanning independent features out as parallel agents. So "build the app" and "add a
feature" are separate, composable prompts — the former orchestrates many runs of
the latter.

## Prompt Map

- `product/gather-requirements.md`: Turn a rough product idea into a clear,
  testable requirements document.
- `ios/build-app.md`: Stand up an iOS app from approved requirements —
  architecture, project structure, shared infrastructure, and one proving
  end-to-end slice — then decompose the app into feature chunks and orchestrate
  parallel `ios/feature-dev.md` agents (partitioned by file ownership) to build
  them. Use for greenfield or a major re-architecture.
- `ios/feature-dev.md`: Add one feature to an existing iOS codebase, matching its
  architecture and conventions while minimizing blast radius. This is the unit of
  work `ios/build-app.md` fans out; also run it directly to add a single feature
  to a living codebase.
- `ios/security-audit.md`: Audit an iOS codebase for security issues. Report
  findings only.
- `ios/qa/qa-audit.md`: Audit iOS test coverage across all four testing quadrants
  (unit/component, example-driven acceptance, exploratory, and non-functional)
  and identify the production bug classes the current tests may miss. Report gaps
  only.
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
- `web/security-audit.md`: Audit a Next.js codebase for security issues. Report
  findings only.
- `web/qa/qa-audit.md`: Audit Next.js and full-stack TypeScript test coverage
  across all four testing quadrants (unit/component, example-driven acceptance,
  exploratory, and non-functional), focusing on production bug classes like
  schema/type drift, Server/Client boundary failures, stale caches, webhook
  idempotency, and SDK-shape changes. Report gaps only.
- `web/refactoring.md`: Audit a Next.js/TypeScript codebase for maintainability
  and produce a prioritized, behavior-preserving refactoring plan, ranked by
  churn × complexity. Report a plan only; establishes the existing test suite
  (and any `web/qa/qa-audit.md` findings) as the regression safety net the
  refactoring must keep green.

Shared reference (not a standalone prompt — read when a prompt points to it):

- `shared/testing-quadrants.md`: The canonical four-testing-quadrants model and
  context-driven-testing framing. Single source of truth for the QA-audit and
  refactoring-audit prompts so the iOS and web variants don't drift apart.

## Recommended Workflow

For a new product:

1. Start with `product/gather-requirements.md`.
2. For web apps, run `web/set-up-hosting.md` before the build prompt unless
   hosting has already been decided.
3. Run the platform `build-app.md` with the approved requirements. It stands up
   the foundation (architecture, shared infrastructure, and one proving
   end-to-end slice), then **decomposes the product into feature-sized chunks and
   orchestrates `feature-dev.md`** — fanning out independent chunks as parallel
   agents partitioned by file ownership, and serializing chunks that share files
   or depend on each other.
4. To add a feature later to the now-living codebase, run the platform
   `feature-dev.md` directly (it inherits the established architecture and
   conventions, so it confirms only what's genuinely unresolved).
5. After implementation, run the relevant security and QA audit prompts; reach
   for the refactoring audit when maintainability degrades.

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
- Present the feature decomposition and parallelization plan, and wait for
  approval, before fanning out agents. After the fan-out, cross-review features
  against each other and run the full test suite on the integrated whole.

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
- `build-app.md`: stop after clarifying questions, then again after the
  architecture plan, then again after the feature decomposition / parallelization
  plan in Phase 5 — before fanning out `feature-dev.md` agents.
- `feature-dev.md`: stop after clarifying questions, then again after the feature
  plan, before writing the implementation.
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
