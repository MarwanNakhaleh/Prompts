# Founder Principles

These are the platform-wide principles for the company-building prompts in
`validation/`, `product/`, and `marketing/` — the business-side counterpart to
`ios/common/engineering-principles.md` and `web/common/engineering-principles.md`.
Every prompt in those folders instructs the LLM to read this file first; it is the
canonical lens for the judgments those prompts make.

Where the engineering principles protect the integrity of the *code*, these
protect the integrity of the *learning*. They exist to stop a team from spending
months — and a budget — building something nobody wants, wrapped in numbers that
only ever went up. The recurring failure mode in going from zero to a real
business is not bad execution; it is confident motion in the wrong direction. Each
principle below is a guardrail against that.

These distill widely-taught customer-development and lean-startup thinking; when a
point is contested or a user wants the deeper argument, point them to search the
web for the primary sources rather than reproducing them here.

## The principles

- **Evidence over invention.** Opinions are noise; behavior is signal. Every
  consequential claim — about who the customer is, what they want, what they'll
  pay — traces to something a real person actually did, said unprompted, or paid
  for. Anything you can't trace is an assumption; label it as one and plan to test
  it. A persona, a market size, or a value prop spun from imagination is fiction
  dressed as research, and it makes bad decisions feel safe. Borrowed evidence
  misleads just as badly: "what the company that won did" is a story
  reverse-engineered from its outcome, not a proven cause — the same traits run
  through the companies that lost, so copying a winner's playbook imports its
  survivorship, not its success. The same trap hides inside your *own* wins: a
  number that went up tells you *that* something worked, never *why* — and a
  success whose cause you can't name can't be repeated and quietly tempts you to
  credit the wrong thing. Being unsure why you're winning is its own danger, so
  trace the mechanism behind a win as rigorously as you'd dissect a loss.

- **The only real validation is a costly action.** Money, a signed commitment, a
  booked sales call, time spent, reputation staked. Likes, survey "yeses,"
  email-only signups, and "I would totally buy that" cost the person nothing and
  prove nothing. When you need to know if something is real, ask for a stake — and
  weigh the answer by what it cost them to give.

- **Talk to customers, and trust behavior over predictions.** Get out and ask —
  before you build, and while you build. But ask about their life and their past,
  not your idea: what they did, the last time the problem bit, what they already
  tried, what they already pay for. What someone has done is evidence; what they
  say they'll do is a guess they're bad at. And a compliment is a warning sign that
  you pitched instead of listened. Know what conversations can and can't prove:
  where the risk is in the *market* (do they want it, will they pay), talking
  settles it; where the risk is in the *product* (a marketplace, an ad network, a
  game — anything where "if you can build/grow it, of course I'll buy" is the
  honest answer), no conversation validates it, so confirm they're not opposed and
  start building sooner.

- **The smallest test that settles the question wins.** Optimize for validated
  learning, not for shipping features or looking impressive. Fake before you build:
  a landing page, a concierge run done by hand, a Wizard-of-Oz, a pre-sell can
  answer the question for a fraction of the cost of the real thing. If a cheaper
  test would change your mind, run it first — be willing to say "don't build
  anything yet." Smallest in *cost*, but never starved of *volume*: most people
  wildly underestimate how much a fair test takes, and a test run at a fraction of
  the volume it needs returns a false "no," not an answer. Name the volume a real
  read requires before you run, and don't let a thin sample masquerade as a verdict.

- **Define the metric and the threshold before you run.** Name the one behavioral
  metric that proves or kills the hypothesis, and the number that means pass and
  the number that means fail — *before* the data lands. A metric set afterward is a
  rationalization, not a test. No moving goalposts.

- **Vanity metrics are forbidden.** Totals that only ever go up — pageviews,
  cumulative signups, registered users — measure your optimism, not your traction.
  Measure behavior tied to the hypothesis: conversion to a costly action, repeat
  use, retention, the action where the people who stay diverge from the people who
  leave.

- **Narrow beats broad.** One beachhead segment, one primary avatar, one
  conversion action, one hypothesis per experiment. A wedge that's mildly useful to
  everyone loses to one that's essential to someone. "For everyone" is for no one.
  You earn the right to broaden by first winning somewhere specific. And narrow
  toward what you can be *the best* at, not merely what you're good at: competence
  at something — even something profitable, even your current core — is not reason
  enough to build on it if someone else will always do it better. The hardest
  focus is dropping a thing you're good at to concentrate where you can actually
  win.

- **Lead with the customer's outcome, in the customer's words.** People buy what
  they get to do or stop doing, not the thing you built. Frame every value prop as
  an outcome with the feature as supporting proof, and say it in the language a
  real customer used — never internal jargon, invented category names, or hype.

- **Every claim needs proof, and you never fabricate it.** A value prop without
  proof is a claim; a claim without proof is noise. Tie each to a metric, a
  mechanism, a real quote, or a demonstrable fact. Never invent a testimonial, a
  logo, a number, a persona, or a customer quote — if the proof doesn't exist yet,
  mark it as an asset to gather, and say so.

- **Sequence the de-risking.** Problem → demand → solution → activation → channel.
  Confirm the problem is real before testing whether they'll pay; confirm they'll
  pay before building; get users to first value before pouring on acquisition.
  Skipping ahead just means you find out later, when the mistake is expensive. A
  false positive caught early is the cheapest save you'll ever get.

- **Make assumptions visible; separate evidence from inference from hope.** Never
  silently invent a product, pricing, positioning, or targeting decision. Surface
  it, recommend with your reasoning, and let the human make the call. State plainly
  which of your inputs are things a customer proved, things you inferred, and
  things you're hoping are true.

- **Treat every conclusion as a living hypothesis.** Avatars, positioning,
  activation moments, messaging — ship the strongest version the evidence supports,
  then keep testing it against real reactions and update. The brief is the current
  best version, never the final truth. And surface what's still unproven loudly, so
  it gets tested instead of enshrined.

- **Steer with pivot-or-persevere, and pivot sooner than feels comfortable.**
  Progress is validated learning, not features shipped or money spent — so plan in
  reverse: decide what you need to learn, then what to measure, then the smallest
  thing to build to measure it. Set a baseline, tune toward the model, and when
  honest, well-run tuning stops moving the metric, that's the signal to *pivot* — a
  structured change of strategy (the segment, the problem, how you capture value,
  the growth engine) that keeps one foot in what you've already proven, not a fresh
  start and not a cosmetic tweak. Almost everyone who pivots wishes they had done it
  sooner; a venture's real runway is the number of pivots it has left, so get to
  each decision faster and don't let going-nowhere "success" talk you out of it.

- **A cheap "no" now beats an expensive one later.** Finding out that nobody has
  the problem, nobody will pay, or the channel doesn't work is the *point* of these
  experiments — celebrate it, don't bury it. The whole discipline exists to make
  killing a bad idea cheap, fast, and free of ego.

- **Outward-facing and money-spending actions need a human gate.** Spending ad
  budget, launching a campaign, emailing a list, or making a promise to a real
  person is not a side effect of analysis. Gate every such action behind explicit
  human approval, prefer the lower-risk read-only path first, and never strand or
  deceive someone who took you up on an offer — the trust of your earliest
  customers is the scarcest thing you have.
