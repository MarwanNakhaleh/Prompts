# Writing Voice Law (all user-facing copy)

A shared reference, like `testing-quadrants.md`: every prompt or task that produces
or edits user-facing copy (landing pages, lifecycle email, blog articles, product UI
strings, transactional emails, support replies) reads this file first. Project repos
may keep a project-specific voice file (product examples, product names); where one
exists it inherits this law and adds to it, never relaxes it.

Applies to every string a user reads. Never applies to code comments, log lines,
commit messages, or internal docs.

## Absolute bans

- **Em dashes in user-facing copy.** Use a colon, a comma, or split the sentence.
- **Negative parallelism**: "It's not X it's Y", "not just X but Y", "Not only... but
  also", "Not because X. But because Y." Assert the thing directly.
- **False intimacy**: "Here's the thing" / "Here's the deal" / "Here's the kicker".
- **Metaphor filler**: journey / navigate / landscape / leverage.

## Word-level bans (2026 tell set)

- showcase / showcasing / underscore / highlighting / emphasizing / fostering /
  enhancing. Say the fact; drop the significance verb.
- delve, utilize, facilitate, encompass, embark. Use: look at, use, help, cover, start.
- "serves as" / "stands as" / "functions as" / "boasts" when they mean is / has.
- tapestry, testament, realm, myriad, plethora, paradigm, synergy, holistic. Kill on sight.
- robust, seamless, comprehensive, pivotal, crucial, meticulous, intricate. Use:
  strong, smooth, complete, key, careful, complex.
- unlock / unleash / harness / elevate / supercharge. Use the plain verb.
- craft / tailored / bespoke. Use: built, made, fit.
- game-changer, groundbreaking, transformative, unprecedented, cutting-edge,
  innovative. Earn it with a number or delete it.
- ever-evolving, fast-paced, digital age. Delete; open with the fact.
- "Additionally," / "Moreover," / "Furthermore," as sentence openers. Use "also",
  "and", or start with the subject.

## Phrase-level bans

- "In today's ... world" / "In a world where". Open with the concrete situation.
- "Let's dive in" / "Let's explore" / "Let's take a look". Delete; the first sentence
  does the work.
- "The result?" / "The answer?" / "And the best part?" one-liners. Put the result in
  a normal sentence.
- "No X. No Y. Just Z." and staccato triples. One sentence, varied shape.
- "Studies have shown" / "Experts agree" without a citation. Name the study or use
  first-party numbers.
- "It's important to note" / "worth noting". If it matters, the sentence carries it.
- "plays a crucial role in" / "underscores the importance of" / "reflects broader
  trends". Name the consequence.
- "refers to the process of" / "is defined as" / "can be broadly categorized into".
  Define by example.
- best practices, low-hanging fruit, move the needle, next level, secret sauce.
  Name the specific action.
- "In conclusion" / "In summary" closing paragraphs. End on the strongest fact or
  the CTA.

## Punctuation and structure bans

- Title Case headings. Sentence case everywhere ("What we check", not "What We Check").
- Curly quotes and apostrophes in email templates and plain-text contexts.
- Bolded lead-ins on every list item (`**Key point:**` x n). At most one bolded
  phrase per section; vary item shapes.
- Uniform bullets (all one line, or all exactly two lines, same grammar). Let lengths
  vary; one bullet can be a fragment, one can carry a number.
- ", highlighting..." / ", underscoring..." / ", emphasizing..." participle tails.
  Fact, period. (The current #1 AI structure.)
- Rule of three twice in a row. Two items, four, or one with a number.
- Every paragraph the same length. One-sentence paragraphs are legal; kill wrap-up
  sentences that restate the section.
- Fake suspense (question line, break, reveal). Zero per document, maximum one.
- Perfect balance (every pro with a con, every list exactly N). Take positions;
  admit asymmetries.

## Rhythm

Vary sentence length as a side effect of substance, not as a trick: a long concrete
thought, then a three-word punch. Fragments are legal. "It works." "No luck." Machines
avoid those; people write them.

## What reads as human (do this)

- Concrete nouns and real numbers. "71% of failures in our last 164 tests were
  certificate config." Specificity is the only structural antidote.
- Plain verbs: ran, checked, sent, failed. Not: executed, validated, facilitated,
  leveraged.
- is / are / has. Not: serves as / features / boasts.
- Committed claims: "the only", "was the first", "we picked 8 checks because they map
  to audit failures."
- First person and reasons. Say why it was built this way.
- Real asymmetry: "This won't catch partner-side misconfig. Nothing does."

## Rewrite ground rules

- Preserve every fact, number, product claim, and SEO keyword. Style changes only.
  If a rewrite would change meaning, stop and flag it.
- Legal copy (privacy, terms, contracts): strip em dashes and the worst tells only.
  Do not restructure.
- Update any test or content snapshot that asserts the old copy.
- Never send email or push copy live unless the human explicitly said to; drafts
  wait for the owner.
