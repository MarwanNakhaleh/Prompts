# iOS Engineering Principles

These principles apply to every prompt in `ios/` — build-app, feature-dev, refactoring, security, and QA. They are the background framework for every architecture, implementation, and audit decision.

## Simplicity is the deliverable
If two solutions work, ship the one that's easier to read and delete. Optimize for readability and the next engineer who maintains this. Unnecessary abstraction, premature generalization, and speculative flexibility are bugs.

## Don't gold-plate
Solve the stated problem, not imagined future ones. The simplest version that fully works is the target.

## True vs. accidental duplication
Before unifying code, confirm the duplication is real. True duplication is one concept with one reason to change — every copy must change together, always. Accidental (false) duplication is code that merely looks alike today but serves different use cases and will diverge (two screens, two flows, two roles). Unifying accidental duplication couples things that change for different reasons and is painful to unwind — flag it as *deliberately keep separate*. A wire/persistence DTO that happens to look like a domain type is the classic false positive: keep them distinct. When in doubt, let the duplication stand until the shared rule is proven.

## Dependency Rule — high-level policy must not depend on low-level detail
Source-code dependencies point inward toward the domain. The UI framework (SwiftUI/UIKit), the network stack, and the persistence layer are details that should depend on your domain logic, not define it — so they stay swappable and the logic stays testable in isolation without a simulator, live server, or real store. A domain type should never import `SwiftUI`, `CoreData`, or `URLSession`, or be shaped by a `Codable` DTO or a managed object. Define a protocol/port at each boundary and push the framework/persistence detail behind it. Map wire/persistence DTOs to domain types at the boundary rather than reusing one across it.

## Don't marry the framework — frameworks and SDKs are details
A framework is a detail you use, not a foundation you marry. Keep third-party framework and SDK types and annotations out of your domain/business objects. Wrap them behind a protocol you own and confine them to an outer layer — so you can adopt, upgrade, or replace the framework without rewriting the core. Apply this to UIKit, SwiftUI, Core Data, Combine, and any third-party SDK.

## Lean on the language to hold boundaries
Use access control (`private`, `fileprivate`, `internal`) and Swift Package Manager module seams so a forbidden dependency fails to compile rather than relying on code review to catch it. Default to the tightest access level and widen only when a real cross-boundary need appears. Where the architecture depends on convention alone to keep illegal imports out, a boundary will eventually be violated under time pressure.

## Composition root — wire once, inject inward
Construct the object graph in a single composition root (the `@main` `App` / `AppDelegate`) — the one "dirty" place allowed to know concrete types, the DI framework, and configuration. Domain logic and views receive their dependencies through protocols and never reach for a shared container, singleton, or DI framework directly. This lets the same core run unchanged under a different configuration (dev/test/prod, or a SwiftUI preview) by swapping only what the root injects. Think of the composition root as a *plugin*: you can have multiple — one for dev, one for test, one for prod, one per customer or region. The core system is never aware of which plugin is active; only the root changes.

## Resilience — assume the network and every backend can fail or stall
Put an explicit timeout on every request — a slow server is more dangerous than a down one, because it hangs the UI while a down one fails fast. Bound retries with backoff. Degrade gracefully — cached/partial content with a retry affordance beats an endless spinner. Make retried writes idempotent. Treat any locally held server entity as a possibly-stale snapshot. Decode external responses tolerantly: handle unknown enum cases with a fallback; extract only the fields you use.

## Implement boundaries at the inflection point
Architectural boundaries cost money to implement and cost money to leave out. Don't add them speculatively up front (YAGNI applies), but watch for the friction that signals one is needed: two concerns changing at different rates, inability to test one thing without spinning up another, or a "small" feature requiring coordinated changes across many unrelated areas. When that friction appears, weigh the ongoing cost of ignoring the boundary against the one-time cost of drawing it — and implement it when the former exceeds the latter. Revisit this judgment frequently; it is not a one-time decision.

## Surface trade-offs explicitly
Don't bury decisions in code. A constraint, a risk, or a rejected alternative worth knowing belongs in a comment, a plan, or a handoff note — not silent in the implementation.

## Ask when uncertain
A question is cheaper than a wrong implementation or rewrite. If a decision is genuinely ambiguous, surface it rather than guessing.

## Cite Apple documentation
When a decision rests on current platform behavior, cite the Apple developer documentation or the relevant WWDC session. Training data can be behind; first-party sources take precedence over community convention.
