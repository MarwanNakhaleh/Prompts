# Shared Reference: The Four Testing Quadrants & Context-Driven Judgment

Canonical framing referenced by the QA-audit prompts (`ios/qa/qa-audit.md`,
`web/qa/qa-audit.md`) and the refactoring-audit prompts (`ios/refactoring.md`,
`web/refactoring.md`). It lives here so the platform variants share one source of
truth and don't drift apart. A prompt that points here should still state the
model in one line inline so it reads coherently on its own, then defer here for
the full version.

## Cover all four testing quadrants, not just the automated ones

Use this two-axis coverage map — business-facing vs. technology-facing × tests
that *support* building the product vs. tests that *critique* the finished
product — so coverage is never judged from a single corner:

- **Q1 — technology-facing, supports the team:** unit and component tests.
- **Q2 — business-facing, supports the team:** acceptance / story / E2E tests
  derived from concrete customer/user examples (the agreed conditions of
  satisfaction).
- **Q3 — business-facing, critiques the product:** exploratory, scenario,
  usability, and UAT — driven by a thinking human, not a script.
- **Q4 — technology-facing, critiques the product:** performance, load,
  security, and the "ilities."

A suite that lives entirely in Q1/Q2 can be all-green and still ship the bugs
that matter. Always note explicitly which quadrants the existing effort neglects.

## Apply context-driven judgment

These categories are a tool, not a rule. The value of any test or practice
depends on the product's risk profile: weight effort toward the paths whose
failure costs the most (payments, auth, data integrity, boundary/decode paths),
and don't automate trivia just because it can be automated. Look-and-feel,
usability, and one-off investigations are often cheaper to verify by hand than to
maintain forever; the highest-ROI automation lives in the fast unit/component
base of the pyramid.

## Using the quadrants as a refactoring safety net

When a refactoring audit fixes the regression baseline — the set of behavior that
must keep passing — read it through all four quadrants, not only the Q1 unit
tests. Q2 acceptance/E2E and any Q3/Q4 checks that are automated or written down
(including Markdown-described browser/QA cases) are part of the contract a
behavior-preserving refactor must not break. Judging preservation against only
the fast part of the safety net is how a "pure refactor" silently breaks a
user-visible flow.

(Search the web for "agile testing quadrants" and "context-driven testing" to
expand on this model.)
