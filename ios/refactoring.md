# iOS Refactoring Audit Prompt for Claude Code

> Acting as a principal iOS engineer specializing in maintainability and large-codebase health, perform a comprehensive refactoring audit of this codebase. **Do not implement any changes** — document a prioritized refactoring plan only. Every proposed refactoring must be behavior-preserving; flag any that would change observable behavior as out of scope for a refactor. Prioritize by churn × complexity (the code most often edited and hardest to read pays back the most), not by what's merely ugly. Do not propose refactoring code that is stable, untouched, and working unless it actively blocks something.

---

## 0. App-Specific Context (fill this in before running)

<!-- The more you provide, the sharper and more sequenced the plan. -->

- **UI framework / language:** SwiftUI / UIKit / mixed; Swift version; min target
- **Architecture in use (or intended):** MVC / MVVM / TCA / VIPER / Clean / ad hoc
- **Concurrency model:** async/await + actors / Combine / GCD / completion handlers / mixed; Swift 6 strict concurrency on/off
- **Persistence & networking:** (SwiftData/Core Data/Realm/…; URLSession/Alamofire/…)
- **DI approach:** constructor injection / singletons / a container / none
- **Test coverage reality:** which layers are tested vs. bare (this determines what's safe to refactor first)
- **Known pain points:** files everyone dreads touching, frequent merge conflicts, slow-to-change areas
- **Constraints:** areas that are off-limits, a migration already in flight (e.g. UIKit→SwiftUI), deprecation deadlines

---

## 1. Refactoring Strategy & Safety Net

- Identify hotspots: cross-reference file size / cyclomatic complexity with edit frequency (git churn if available). Rank refactoring targets by churn × complexity, not aesthetics
- **Establish the regression baseline first.** Inventory the full existing test suite — unit (XCTest/Swift Testing), integration, contract, snapshot, and UI automation (XCUITest) — plus any manual or browser/QA test cases described in Markdown (a UI testing guide, a release checklist). This enumerated set is the behavior that must continue to pass after every refactor; treat it as the contract the refactoring must preserve, and cite the specific covering test(s) in each finding's safety-net column. Surface this inventory in the report so the reader knows exactly which tests gate the work. Read the baseline through all four testing quadrants (see `shared/testing-quadrants.md`) — not only Q1 unit tests but Q2 acceptance/UI tests and any automated or Markdown-described Q3/Q4 checks — so behavior preservation is judged against the whole safety net, not just its fast part
- Lean on the QA audit for coverage truth: if `ios/qa/qa-audit.md` has been run, ingest its findings to locate thin/zero-coverage areas rather than re-deriving them; if it hasn't, do a lightweight coverage pass yourself here. For each proposed target, state whether a real safety net exists — code with no tests needs **characterization tests written first** (call this out as a hard prerequisite, not an afterthought, naming the missing coverage), and the safe-to-refactor-first order should follow actual coverage, not assumption
- Flag anywhere a "refactor" is really tangled with a behavior change; recommend splitting behavior changes out so the refactor stays pure
- Note any large mechanical refactor (rename, signature change) that a tool/compiler can do safely vs. one needing manual judgment

## 2. Oversized & Overloaded Units

- Flag Massive View Controllers (UIKit) and giant SwiftUI `body`/view files mixing layout, navigation, networking, and business logic
- Identify long methods, deep nesting, long parameter lists, and high cyclomatic complexity — candidates for Extract Method / Extract Type / Introduce Parameter Object
- Find "god" types (managers/services/coordinators) that have accreted unrelated responsibilities — candidates for splitting by responsibility
- Flag huge `switch`/`if-else` ladders over a type that could become polymorphism or an enum with associated values

## 3. Separation of Concerns & Architecture

- Identify business logic living in views / view controllers that belongs in a view model, use case, or service layer
- Flag networking, persistence, or formatting performed directly inside UI types
- Audit consistency with the codebase's stated architecture — where does it diverge, and is the divergence load-bearing or just drift?
- Review the SwiftUI view tree for extraction opportunities: repeated or large inline subviews that should become reusable components
- Check view-model boundaries: are they testable (no UIKit/SwiftUI imports leaking in), and is presentation logic isolated from view logic?
- Apply the **Dependency Rule**: source-code dependencies should point inward toward higher-level policy (the domain / business rules), with SwiftUI/UIKit, networking, and persistence as outer *details* reached through protocols the domain owns — never the reverse. Flag a domain type that imports `SwiftUI`/`CoreData`/`URLSession` or is shaped by a `Codable` DTO or a managed object, and name the fix: define a protocol/port at the boundary and push the detail behind it
- Draw boundaries along **axes of change** — where two concerns change at different rates and for different reasons (UI vs. business rules, business rules vs. a vendor SDK) — not on instinct. Where fast-churning and slow-churning code are fused in one type, propose the seam; where a "boundary" only separates things that always change together, flag it as needless indirection to collapse
- Apply the **Humble Object** move for testability: when hard-to-test logic (formatting, branching, state derivation, entitlement decisions) is welded into a framework-bound shell (a SwiftUI `body`, a view controller, a store/client wrapper), split it — keep the shell humble (it renders or runs only what it's handed) and extract the decision logic into a plain, framework-free type (view model / presenter / pure function) that unit tests call directly. This is often the highest-leverage refactor for a low-coverage hotspot, because it converts a UI-test-only behavior into a unit-testable one

## 4. Concurrency Modernization (behavior-preserving only)

- Identify completion-handler APIs with safe async/await equivalents; flag as candidates for an async refactor (only where semantics are genuinely preserved)
- Flag manual GCD (`DispatchQueue`) usage that structured concurrency would express more clearly
- Find mixed Combine + async/await where consolidating onto one model would reduce cognitive load — note migration risk
- Audit `@MainActor` placement and actor boundaries for clarity, and flag concurrency code that is correct-but-inscrutable as a readability refactor (distinct from a correctness fix)

## 5. State & Observation Patterns

- Flag legacy `ObservableObject`/`@Published` that could move to the `@Observable` macro where the deployment target allows (note this can have behavior nuances — verify, don't assume)
- Identify `@ObservedObject` used where `@StateObject` is correct (this is also a correctness issue — flag as such)
- Audit prop/binding drilling that a better state-ownership or environment design would simplify
- Find scattered/global mutable state and singletons that obstruct testing — candidates for dependency injection

## 6. Type Safety & Swift Idiom

- Flag force-unwraps (`!`), `try!`, and implicitly-unwrapped optionals that a safer pattern would replace
- Identify stringly-typed code (string keys, raw string states) that could become enums or strong types
- Flag overuse of `Any`/`AnyObject` and weak typing at boundaries
- Find non-idiomatic patterns: manual loops where `map`/`filter`/`reduce` reads better, reference types that could be value types, opportunities for protocol-oriented design
- Identify magic numbers/strings that should be named constants, and hardcoded user-facing strings that should be localized

## 7. Duplication & Reuse

- **Before proposing any unification, confirm the duplication is real.** True duplication is one concept with one reason to change — every copy must change together, always. Accidental (false) duplication is code that merely looks alike today but serves different use cases and will diverge (two screens, two flows, two roles). Unifying accidental duplication couples things that change independently and is harder to unwind later than the duplication was to tolerate — so flag it as *deliberately keep separate*, not a refactor. A wire/persistence DTO that happens to look like a domain or view type is the classic false positive: keep them distinct
- Find copy-pasted or near-duplicate logic (networking, validation, formatting, view construction) that should be unified — name the Extract / Pull-Up move
- Identify parallel implementations that have drifted and should converge on one source of truth
- Flag repeated boilerplate that a small helper, extension, or generic would collapse (without over-abstracting)

## 8. Dead Code, Cruft & Deprecations

- Identify unused types, methods, properties, and resources; commented-out code; unreachable branches
- Flag feature-flag debris and abandoned migration scaffolding
- Find usages of deprecated UIKit/SwiftUI/Foundation APIs and note the current replacement
- Flag stale `// TODO`/`// FIXME`/`// HACK` markers and assess whether each represents real latent debt

## 9. Module & Dependency Structure

- Review file/group/module organization — does it reflect the architecture, or is everything in a few mega-targets? Ask whether the top-level structure **announces the domain** (the app's features and use cases) or merely the UI framework, leaving the domain scattered. Reorganizing to reveal intent is a valid refactor where ownership is unclear
- Identify circular dependencies and unclear dependency direction (UI depending on infrastructure, etc.). Break cycles by inverting a dependency (extract a protocol) or hoisting the shared piece into a lower-level module/package
- Flag tight coupling that obstructs testing or reuse — candidates for protocol seams / dependency inversion. Dependencies should run toward **stability**: volatile code (a view, a specific vendor adapter) may depend on stable, abstract core types, but a stable/widely-depended-on type that imports a volatile one is a refactoring target — the volatile detail makes the stable core hard to change
- Assess opportunities to extract independent code into Swift packages/modules to enforce boundaries and speed builds

---

## Output Format

Organize all findings into a markdown report grouped by the categories above. For each proposed refactoring include:

| Field | Description |
|-------|-------------|
| **Priority** | P0 / P1 / P2 / P3 (P0 = high-churn hotspot actively slowing delivery or breeding bugs) |
| **File or Surface** | Exact file path / type / view, with line numbers where applicable |
| **Smell** | The named code smell or problem (e.g., "Massive View Controller", "force-unwrap chain", "duplicated networking", "business logic in view") |
| **Proposed refactoring** | The named, behavior-preserving move (e.g., Extract View Model, Replace Conditional with Polymorphism, Introduce Parameter Object) |
| **Behavior risk & safety net** | Risk it changes observable behavior, and the specific covering test(s) from the baseline inventory that would catch a regression — or, if none exist, that characterization tests are a prerequisite |
| **Effort** | Half-day / 1 day / 2 days / >2 days |

Begin the report with an executive summary: a count of opportunities per priority, the top 3–5 hotspots ranked by churn × complexity, a recommended sequence (what to do first, and what needs a test safety net before being touched), and one sentence on what maintaining the status quo costs over the next few quarters.