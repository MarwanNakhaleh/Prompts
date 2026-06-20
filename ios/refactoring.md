# iOS Refactoring Audit Prompt for Claude Code

> Acting as a principal iOS engineer specializing in maintainability and large-codebase health, perform a comprehensive refactoring audit of this codebase. **Do not implement any changes** — document a prioritized refactoring plan only. Every proposed refactoring must be behavior-preserving; flag any that would change observable behavior as out of scope for a refactor. Prioritize by churn × complexity (the code most often edited and hardest to read pays back the most), not by what's merely ugly. Do not propose refactoring code that is stable, untouched, and working unless it actively blocks something.

Read `ios/common/engineering-principles.md` — it is the canonical reference for why findings in this audit are flagged (dependency rule, don't marry the framework, compiler-enforced boundaries, composition root, true vs. accidental duplication, and more).

Consult `ios/resources.md` for the authoritative documentation source for each application slice you audit (networking, auth, secure storage, concurrency, payments, etc.). Use those sources to verify current Apple guidance and catch deprecated APIs before recommending a refactoring direction.

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
- **Work in tiny, test-preserving steps.** Every proposed refactoring should decompose into increments where the test suite is green before and after each step. If a move cannot be broken into safe, passing increments, it is either too large (split it) or it changes behavior (flag it as out of scope for a refactor). Apply the Boy Scout Rule as the guiding philosophy: each edit leaves the touched area slightly cleaner — a better name, a shorter function, a removed dead branch. A refactoring plan is a sequenced backlog of such edits, not a big-bang rewrite.

## 2. Oversized & Overloaded Units

- Flag Massive View Controllers (UIKit) and giant SwiftUI `body`/view files mixing layout, navigation, networking, and business logic
- Identify long methods, deep nesting, long parameter lists, and high cyclomatic complexity — candidates for Extract Method / Extract Type / Introduce Parameter Object
- Find "god" types (managers/services/coordinators) that have accreted unrelated responsibilities — candidates for splitting by responsibility. Apply the **naming test for over-responsibility**: if a type's name contains vague aggregation words like `Processor`, `Manager`, `Handler`, `Coordinator`, or `Service` (without a specific domain qualifier), that name is a near-certain signal of accumulated unrelated concerns. As a quick check: if you cannot describe the type's purpose in about 25 words without using "if", "and", "or", or "but", it almost certainly has more than one responsibility — the connective tissue in that description reveals where to draw the split.
- Flag huge `switch`/`if-else` ladders over a type that could become polymorphism or an enum with associated values
- Flag **flag arguments** (boolean parameters) as a reliable signal the function does two things — split into two clearly-named functions (`renderForSuite()` / `renderForSingleTest()` instead of `render(isSuite: Bool)`)
- Flag **mixed abstraction levels** within a single function: high-level policy operations (`getHtml()`) should not coexist in the same body as low-level string manipulation (`.append("\n")`). Each function should contain steps that are one level of abstraction below its name (the Stepdown Rule); when you can extract a sub-function with a name that is not merely a restatement of the implementation, the function is doing more than one thing
- Flag **command-query separation violations**: functions that both mutate state and return information about the object lead to ambiguous call sites; prefer two separate operations — one that changes state (returns nothing) and one that returns information (has no side effects)
- Flag **Law of Demeter violations** (train wrecks): method chains that navigate through multiple objects' internals — e.g., `viewController.session.user.preferences.theme` — expose hidden structure, couple the caller to every intermediate type, and require updates across the chain when any link's shape changes. The fix is usually to add a method to the nearest object that performs the operation internally, or to decompose the chain into named local values whose types are clearly owned by the caller
- Flag **feature envy**: a method that accesses more data or calls more methods from another type than from its own is a signal it wants to live in that other type. Look for methods dominated by `other.getX()`, `other.getY()`, `other.getZ()` feeding a local calculation — move the method to the type whose data it craves. Common exceptions: deliberate Strategy and Visitor patterns that intentionally cross class lines
- Flag **hidden temporal coupling**: methods that must be called in a specific order but don't express this constraint in their signatures. A comment like "call configure() before use" will eventually be violated under time pressure. The fix: have step N return a value that step N+1 requires as a parameter, so out-of-order calls fail to compile. Flag initializers or setup patterns where the order is enforced only by convention rather than the type system
- Flag **names that require comments to explain what they do**: rename until the comment is unnecessary. A comment explaining *what* a type, method, or variable does is evidence of a naming or structural failure, not documentation virtue
- Flag **complex inline conditionals** that should be extracted into intent-revealing predicate methods — `if canBeCompacted()` is easier to reason about than a multi-clause boolean expression inline in an `if` or `guard`. A conditional that needs a comment to explain what it's checking is a predicate waiting to be named
- Flag **negative conditionals** where a positive form would be clearer — when `!shouldNotProcess()` is semantically equivalent to `canProcess()`, prefer the positive form. Negatives add a mental inversion step that accumulates across a function body

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
- Flag dependency construction and DI-framework/singleton references scattered through feature, domain, and view types. Consolidating object-graph construction into a single **composition root** at the app entry point (`@main` `App`/`AppDelegate`) — and passing dependencies inward through protocols — is a behavior-preserving move that restores testability and lets the same core run under different configurations (dev/test/prod, previews). A domain or view type that reaches for a shared container or singleton directly is the target

## 6. Type Safety & Swift Idiom

- Flag force-unwraps (`!`), `try!`, and implicitly-unwrapped optionals that a safer pattern would replace
- Flag **nil-returning methods** where a special-case object would eliminate defensive guard chains in every caller. A `guard let x = getSomething() else { return }` pattern repeated at every call site is a signal the method could return an empty collection, a zero-value struct, or a null object instead, consolidating "nothing found" handling in one place and removing scattered guards from callers
- Flag **sentinel error returns**: methods that signal failure via a magic return value (−1, 0, `""`, a `"Error: ..."` string) rather than throwing or returning a `Result<Value, Error>`. Callers cannot reliably distinguish a sentinel from a legitimate return value, must remember to check, and often don't — creating silent failure paths. Replace with `throws`, `Result<Value, Error>`, or an `Optional` scoped only to the genuine "no result" case
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
- **Identify boundaries drawn along the wrong axis.** When recent features required modifying many otherwise-unrelated types or files simultaneously — the "cross-cutting concern" signal — the decomposition was likely made along functional behavior rather than along axes of change. A well-placed boundary lets a new capability be added as a new Swift package, module, or group without touching existing ones (the Open-Closed Principle in practice). Flag areas where successive features have consistently forced broad, coordinated edits as candidates for redrawing the boundary along the dimension that actually changes together
- Identify circular dependencies and unclear dependency direction (UI depending on infrastructure, etc.). Break cycles by inverting a dependency (extract a protocol) or hoisting the shared piece into a lower-level module/package
- Flag tight coupling that obstructs testing or reuse — candidates for protocol seams / dependency inversion. Dependencies should run toward **stability**: volatile code (a view, a specific vendor adapter) may depend on stable, abstract core types, but a stable/widely-depended-on type that imports a volatile one is a refactoring target — the volatile detail makes the stable core hard to change
- Assess opportunities to extract independent code into Swift packages/modules to enforce boundaries and speed builds
- Enforce boundaries with the language, not just convention. Where the architecture relies on review/discipline to keep illegal dependencies out (a view reaching into networking, a feature importing another feature's internals, a use case touching a `Codable` DTO), use access control (`private`/`fileprivate`/`internal`) and separate modules/SPM packages so the forbidden dependency fails to compile rather than passing review. Flag widely-`public` types that leak implementation detail across a boundary where a tighter access level — or moving the type behind a module seam — would let the compiler hold the line; the smallest-access default is the cheapest enforcement.
- Flag **access-visibility inflation** as an architectural smell in its own right: types marked `public` or `open` only because they happen to live in a different file or group — not because external callers genuinely need them — are a sign that access control is being ignored rather than used as an architectural tool. When every type in a module is publicly visible, the chosen architectural style (layered, feature-sliced, ports and adapters, component-based) becomes effectively meaningless: any caller can reach any concrete implementation directly, bypassing the intended boundary. The architectural style on paper and the structure the compiler actually enforces collapse into the same flat topology. For each widely-public type, ask: "Is there a concrete external caller that needs this, or is it public by habit?" Tighten where the answer is "habit."

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