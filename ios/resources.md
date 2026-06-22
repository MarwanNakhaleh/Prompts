# iOS Resources

Organized by application slice. When a prompt instructs you to research current platform guidance, start here — these are the authoritative, regularly-updated sources. First-party Apple sources take precedence over community convention whenever they conflict.

---

## Swift Language

The language layer every other slice depends on.

- **The Swift Programming Language** — https://docs.swift.org/swift-book/documentation/the-swift-programming-language/
  Authoritative reference for syntax, value semantics, access control, generics, concurrency model, and protocol-oriented design. Check here before accepting community convention on language behavior.

- **Swift API Design Guidelines** — https://www.swift.org/documentation/api-design-guidelines/
  Apple's canonical naming and API-surface rules. The basis for the "names reveal intent" and "one word per concept" principles in `ios/common/engineering-principles.md`.

- **Swift Evolution** — https://github.com/apple/swift-evolution
  Accepted and pending language proposals. The definitive source for understanding *why* a Swift feature works the way it does and what changed between versions.

- **Swift.org Blog** — https://www.swift.org/blog/
  Release announcements and migration guides for new Swift versions. Check here when a language feature the codebase relies on changes behavior between Swift versions.

---

## SwiftUI / UIKit (Presentation Layer)

The rendering and layout slice — where the UI is assembled and how it connects to state.

- **SwiftUI Documentation** — https://developer.apple.com/documentation/swiftui
  Top-level entry point: Views, layout, modifiers, environment, and the full framework reference. Consult before assuming behavior from training data; SwiftUI changes meaningfully across minor OS versions.

- **SwiftUI App Lifecycle** — https://developer.apple.com/documentation/swiftui/app
  The `App` protocol, `@main`, `Scene`, and how the composition root connects to the SwiftUI lifecycle. The authoritative source for how to wire dependencies at startup.

- **UIKit Documentation** — https://developer.apple.com/documentation/uikit
  Full UIKit reference. Use when working in a UIKit or mixed codebase. Also the reference for UIKit/SwiftUI interoperability via `UIViewRepresentable` / `UIViewControllerRepresentable`.

- **Human Interface Guidelines — iOS** — https://developer.apple.com/design/human-interface-guidelines/ios
  Layout, navigation patterns, accessibility, Dynamic Type, dark mode, and gesture conventions. Required reading when a feature specifies matching the platform's UX conventions.

---

## State & Observation

How data flows from the domain into the UI and how views subscribe to changes.

- **Observation Framework (`@Observable`)** — https://developer.apple.com/documentation/observation
  The `@Observable` macro and its behavior differences from `ObservableObject`. Prefer over `ObservableObject`/`@Published` where the deployment target allows; verify behavior nuances here before migrating.

- **WWDC 2023 — Discover Observation in SwiftUI** — https://developer.apple.com/videos/play/wwdc2023/10149/
  Migration path from `ObservableObject` to `@Observable`, ownership semantics, and differences in update granularity.

- **Managing Model Data in Your App** — https://developer.apple.com/documentation/swiftui/managing-model-data-in-your-app
  Explains `@StateObject` vs. `@ObservedObject` ownership semantics. The refactoring prompt flags `@ObservedObject` used where `@StateObject` is correct; verify the distinction here.

- **WWDC 2021 — Demystify SwiftUI** — https://developer.apple.com/videos/play/wwdc2021/10022/
  How SwiftUI's diffing and identity system works under the hood. Use when a feature touches view identity or causes unexpected re-renders.

- **WWDC 2020 — Data Essentials in SwiftUI** — https://developer.apple.com/videos/play/wwdc2020/10040/
  `@State`, `@Binding`, `@StateObject`, `@ObservedObject`, `@EnvironmentObject` lifecycle and ownership. Use when the refactoring prompt flags incorrect property-wrapper usage.

---

## Swift Concurrency

How async work is structured, isolated to actors, and kept free of data races.

- **Swift Concurrency (Apple Developer)** — https://developer.apple.com/documentation/swift/concurrency
  Top-level entry: async/await, structured tasks, actors, and task groups.

- **Actor isolation and `@MainActor`** — https://developer.apple.com/documentation/swift/actor
  Actor declaration, isolation rules, and `nonisolated`. The reference for `@MainActor` boundary decisions and the "concurrency is a separate concern" principle.

- **Sendable and data-race safety** — https://developer.apple.com/documentation/swift/sendable
  How `Sendable` conformance works and what the compiler enforces under Swift 6 strict concurrency.

- **WWDC 2021 — Swift concurrency: Behind the scenes** — https://developer.apple.com/videos/play/wwdc2021/10254/
  The cooperative thread pool, why blocking is dangerous, and the runtime model behind structured concurrency. Essential before any concurrency refactor.

- **WWDC 2022 — Eliminate data races using Swift Concurrency** — https://developer.apple.com/videos/play/wwdc2022/110351/
  Task isolation, `Sendable`, and actor boundaries. The basis for understanding what Swift 6 strict concurrency actually enforces.

- **WWDC 2023 — Beyond the basics of structured concurrency** — https://developer.apple.com/videos/play/wwdc2023/10170/
  Task cancellation, task-local values, clocks, and custom executors. Use when a feature involves cancellation, timeouts, or custom actor executors.

---

## Networking & Resilience

Every out-of-process call — its timeout, retry, and degraded-behavior contract.

- **URLSession Documentation** — https://developer.apple.com/documentation/foundation/urlsession
  The canonical networking reference: configuration, background transfers, challenge handling, and response decoding. The `engineering-principles.md` resilience guidance maps directly to `URLSessionConfiguration` knobs here.

- **URLSessionConfiguration** — https://developer.apple.com/documentation/foundation/urlsessionconfiguration
  `timeoutIntervalForRequest`, `timeoutIntervalForResource`, and cache policy. The reference for setting explicit timeouts on every request as required by the resilience principle.

- **Encoding and Decoding Custom Types** — https://developer.apple.com/documentation/foundation/archives_and_serialization/encoding_and_decoding_custom_types
  How `Codable` synthesis works, custom decoding strategies, and how to decode only the fields you use (tolerant decoding). Relevant to the `engineering-principles.md` guidance to decode external responses tolerantly and handle unknown enum cases with a fallback.

- **App Transport Security** — https://developer.apple.com/documentation/bundleresources/information_property_list/nsapptransportsecurity
  ATS configuration keys and exception justification requirements. Use when a feature adds new remote endpoints or when the security audit questions insecure transport settings.

- **NSPinnedDomains (certificate pinning via ATS)** — https://developer.apple.com/documentation/bundleresources/information_property_list/nsapptransportsecurity/nspinneddomains
  Governs per-domain public-key pinning via ATS Info.plist keys. Required reading before implementing certificate pinning; also documents how to configure pin-rotation strategy.

- **Preventing Insecure Network Connections** — https://developer.apple.com/documentation/security/preventing_insecure_network_connections
  Best-practice guidance on ATS requirements; explains which exceptions are reviewable by App Review and which are never acceptable.

---

## Authentication & Session

Credential storage, biometric gates, and session lifecycle.

- **Keychain Services** — https://developer.apple.com/documentation/security/keychain_services
  Storing and retrieving secrets from the Keychain: `SecItemAdd`, `SecItemCopyMatching`, accessibility classes (`kSecAttrAccessibleWhenUnlockedThisDeviceOnly`), and shared Keychain groups. The authoritative reference whenever the security audit or feature-dev prompt involves credential storage.

- **LocalAuthentication** — https://developer.apple.com/documentation/localauthentication
  Face ID / Touch ID API: `LAContext`, policy evaluation, and `LACredentialType`. Use to understand the correct way to back a biometric gate with a Keychain ACL (`SecAccessControl`) rather than a boolean result.

- **SecAccessControl** — https://developer.apple.com/documentation/security/secaccesscontrol
  How to attach biometric or passcode requirements to a Keychain item at the cryptographic layer. Referenced by `ios/security-audit.md` §2 as the correct defense against callback-bypass on jailbroken devices. A biometric gate backed only by an `LAContext` boolean result is insufficient — the Keychain ACL is the correct enforcement layer.

---

## Secure Storage & Data Protection

Encryption at rest, file protection classes, and backup exclusion.

- **Data Protection (File Protection API)** — https://developer.apple.com/documentation/uikit/protecting_the_user_s_privacy/encrypting_your_app_s_files
  File protection classes (`NSFileProtectionComplete`, etc.) and how iOS hardware encryption interacts with them. Use when the security audit asks about sensitive files written to disk.

- **SwiftData Documentation** — https://developer.apple.com/documentation/swiftdata
  Model macros, `ModelContainer`, `ModelContext`, and migration. The first-party reference for SwiftData persistence decisions; prefer over community tutorials when the security audit or refactoring prompt touches the persistence layer.

- **Core Data Documentation** — https://developer.apple.com/documentation/coredata
  Full Core Data reference for codebases on older deployment targets. Use when the refactoring prompt flags managed-object shapes leaking into the domain layer.

- **WWDC 2023 — What's new in SwiftData** — https://developer.apple.com/videos/play/wwdc2023/10196/
  Custom data stores and compound queries. Consult before writing new persistence code.

---

## Cryptography

Key generation, algorithm selection, and Secure Enclave operations.

- **CryptoKit Documentation** — https://developer.apple.com/documentation/cryptokit
  Apple's first-party cryptography framework: symmetric encryption (AES-GCM, ChaChaPoly), hashing (SHA-2 family), HMAC, and asymmetric keys. Prefer over CommonCrypto and hand-rolled crypto; the security audit prompt flags weak or deprecated algorithms (MD5, SHA-1, ECB mode).

- **Security Framework** — https://developer.apple.com/documentation/security
  Lower-level APIs: `SecRandomCopyBytes` for secure randomness, certificate trust evaluation, and the transform API. Use when CryptoKit doesn't cover the required operation.

- **OWASP Mobile Security Testing Guide — Cryptography** — https://mas.owasp.org/MASTG/tests/ios/MASVS-CRYPTO/
  Test cases for key storage, algorithm selection, and randomness. The checklist the security audit prompt uses when CryptoKit usage needs independent verification.

---

## Navigation & Deep Links

In-app routing, universal links, and handling untrusted inbound URLs.

- **NavigationStack / NavigationSplitView** — https://developer.apple.com/documentation/swiftui/navigationstack
  Programmatic navigation, navigation path, and deep-link handling in SwiftUI. Use when a feature adds a new navigation destination or needs to handle deep links that arrive via URL.

- **Universal Links** — https://developer.apple.com/documentation/xcode/supporting-universal-links-in-your-app
  Associated domains entitlement, Apple App Site Association file, and how the OS routes inbound universal links to your app. Use when the security audit asks about deep-link injection.

- **UIApplicationDelegate URL Handling** — https://developer.apple.com/documentation/uikit/uiapplicationdelegate/application(_:open:options:)
  Custom URL scheme handling. The security audit prompt treats all incoming URL scheme data as untrusted; this is the entry point that must validate and sanitize every parameter.

---

## Payments (StoreKit)

In-app purchase flows, receipt validation, and server-side access grants.

- **StoreKit 2 Documentation** — https://developer.apple.com/documentation/storekit
  Product loading, purchase flow, transaction verification, subscription status, and the `Transaction` API. The authoritative reference for any StoreKit 2 implementation.

- **WWDC 2021 — Meet StoreKit 2** — https://developer.apple.com/videos/play/wwdc2021/10114/
  Overview of the new API design, `Transaction.currentEntitlements`, and server-to-server notifications. Use when replacing StoreKit 1 or implementing subscription gating.

- **App Store Server API** — https://developer.apple.com/documentation/appstoreserverapi
  Server-side transaction lookup and subscription status. The security audit prompt flags any purchase or entitlement decision made only on-device; this is the server-side validation counterpart.

---

## Push Notifications

APNs, notification delivery, and notification service extensions.

- **UserNotifications Framework** — https://developer.apple.com/documentation/usernotifications
  `UNUserNotificationCenter`, authorization, scheduling local notifications, and handling remote notification delivery.

- **Pushing Background Updates to Your App** — https://developer.apple.com/documentation/usernotifications/pushing_background_updates_to_your_app
  Silent push via `content-available`, background fetch wake-up, and the constraints on background execution time.

- **UNNotificationServiceExtension** — https://developer.apple.com/documentation/usernotificationsui/unnotificationcontentextension
  Modifying notification content before display. Use when end-to-end encrypted payloads must be decrypted in the extension rather than in the main app.

---

## Background Tasks

Deferred work, background fetch, and long-running processing.

- **BGTaskScheduler** — https://developer.apple.com/documentation/backgroundtasks
  `BGAppRefreshTask` and `BGProcessingTask` — the modern API for scheduling background work. Required when a feature needs to run code while the app is not in the foreground.

- **Optimizing Your App's Data for iCloud Backup** — https://developer.apple.com/documentation/foundation/optimizing_your_app_s_data_for_icloud_backup
  How to mark files as excluded from backup and why — relevant when background tasks write sensitive cache or temporary data to disk.

---

## PDF Rendering & Generation (PDFKit)

Displaying, annotating, searching, and programmatically creating PDF documents.

- **PDFKit Documentation** — https://developer.apple.com/documentation/pdfkit
  Top-level framework reference. Available on iOS 11.0+. Covers the four core classes — `PDFView` (the display widget), `PDFDocument` (the document model), `PDFPage` (per-page rendering, text, and annotation access), and `PDFAnnotation` (interactive and drawn annotations). Check here before accepting any community tutorial; the API surface has expanded meaningfully across OS versions.

- **PDFView** — https://developer.apple.com/documentation/pdfkit/pdfview
  The single widget that encapsulates display, navigation, zoom, text selection, and user interaction. Embed in SwiftUI via `UIViewRepresentable`. Controls display mode (single page, continuous scroll, two-up), auto-scaling behavior, and background color. Required reading before adding a PDF viewer feature — many behaviors the security and QA audits flag (uncontrolled zoom, missing accessibility, unsanitized annotation inputs) are controlled through `PDFView` settings.

- **PDFDocument** — https://developer.apple.com/documentation/pdfkit/pdfdocument
  Load, search, select, write, and build PDF documents programmatically. Supports `init(url:)`, `init(data:)`, and `dataRepresentation()` for round-tripping. Use `find(string:)` for full-document text search and `page(at:)` to access individual pages. When writing a generated PDF to disk, apply the appropriate `NSFileProtection` class to the resulting file — the security audit prompt checks this.

- **PDFPage** — https://developer.apple.com/documentation/pdfkit/pdfpage
  Per-page rendering, annotation management, and text extraction. `attributedString` returns all text on the page; iterate pages and append to extract a full document's text. `thumbnail(of:for:)` generates page thumbnails for navigation UIs. `addAnnotation(_:)` / `removeAnnotation(_:)` manage annotations at the page level.

- **PDFAnnotation** — https://developer.apple.com/documentation/pdfkit/pdfannotation
  Create and configure interactive annotations (highlights, underlines, freehand ink, stamps, text notes, links). When mutating annotations, use `setValue(_:forAnnotationKey:)` with typed `PDFAnnotationKey` constants rather than stringly-typed key paths. User-supplied annotation text is an injection surface — sanitize before writing to the document and before displaying in any web or email context.

- **WWDC 2017 — Introducing PDFKit on iOS** — https://developer.apple.com/videos/play/wwdc2017/241/
  The session that brought PDFKit to iOS 11. Covers the class hierarchy, the annotation key API, rendering pipeline, and bridging to UIKit. Required background before working in any PDFKit codebase; explains decisions that are not obvious from the headers alone.

- **WWDC 2022 — What's new in PDFKit** — https://developer.apple.com/videos/play/wwdc2022/10089/
  iOS 16 additions: `PDFPageOverlayViewProvider` (live, fully interactive `UIView` overlays rendered on top of PDF pages — the correct pattern for embedding custom UI into a PDF viewer); image-based PDF page creation (a new API for building PDF pages from `UIImage` inputs); automatic form-field recognition; and Live Text integration for scanned documents. If the app targets iOS 16+, prefer `PDFPageOverlayViewProvider` over manual drawing in a custom `PDFPage` subclass.

---

## Dependency Injection & Composition Root

Wiring the object graph at startup and keeping dependencies injectable.

- **Swift Package Manager** — https://www.swift.org/documentation/package-manager/
  Creating packages, declaring targets, and enforcing module boundaries so a forbidden dependency fails to compile. Use when the refactoring prompt recommends extracting code into a separate package.

- **Factory (DI container)** — https://github.com/hmlongco/Factory
  A lightweight DI container for Swift. Consult when the codebase uses Factory for its composition root, or when evaluating DI container options.

- **Point-Free — Swift Dependencies** — https://github.com/pointfreeco/swift-dependencies
  A dependency-injection library focused on testability and controlled environments. Common in TCA codebases; also usable standalone. Check here when the codebase uses `@Dependency`.

- **The Composable Architecture (TCA)** — https://github.com/pointfreeco/swift-composable-architecture
  The full TCA library and documentation. Consult before proposing architectural changes to a TCA codebase — its composition root, reducer structure, and dependency tree differ significantly from MVVM.

---

## Testing

Unit, integration, UI, and snapshot testing across the four quadrants.

- **XCTest Documentation** — https://developer.apple.com/documentation/xctest
  Unit tests, UI tests, performance tests. Reference for `XCTAssert*`, `setUp`/`tearDown`, and `XCUIElement` queries.

- **Swift Testing Documentation** — https://developer.apple.com/documentation/testing
  `@Test`, `@Suite`, `#expect`, `#require` macros. Use for new test targets or when the codebase is migrating from XCTest to Swift Testing.

- **WWDC 2024 — Meet Swift Testing** — https://developer.apple.com/videos/play/wwdc2024/10179/
  Parameterization, tagging, and parallel test execution. Use to understand what Swift Testing adds before choosing between XCTest and Swift Testing for a new test suite.

- **Testing in Xcode** — https://developer.apple.com/documentation/xcode/testing-your-apps-in-xcode
  Test plans, schemes, and coverage reports. The practical reference for running and configuring the test suite in CI.

- **Agile Testing Quadrants** — https://lisacrispin.com/2011/11/08/using-the-agile-testing-quadrants/
  Lisa Crispin's explanation of the Q1–Q4 model referenced in `shared/testing-quadrants.md`. Use when deciding which tier a test belongs in.

---

## Permissions, Privacy & Entitlements

What the app declares it does, and what the OS enforces.

- **Privacy Manifest (PrivacyInfo.xcprivacy)** — https://developer.apple.com/documentation/bundleresources/privacy_manifest_files
  Required for App Store submission when the app or its SDKs use privacy-sensitive APIs. The security audit prompt checks this for accuracy against actual data collection.

- **Requesting Access to Protected Resources** — https://developer.apple.com/documentation/uikit/protecting_the_user_s_privacy/requesting_access_to_protected_resources
  How to request location, contacts, camera, microphone, and photos access. Use to ensure purpose strings accurately reflect minimal data access.

- **Entitlements** — https://developer.apple.com/documentation/bundleresources/entitlements
  Full entitlements reference — App Groups, Keychain sharing, associated domains, background modes. The security audit prompt checks all entitlements for justification.

---

## Accessibility

VoiceOver, Dynamic Type, and inclusive design requirements.

- **Accessibility for iOS and iPadOS** — https://developer.apple.com/accessibility/ios/
  Overview of VoiceOver, Switch Control, Dynamic Type, and display accommodations. The starting point for any accessibility requirement in the feature-dev prompt.

- **SwiftUI Accessibility Modifiers** — https://developer.apple.com/documentation/swiftui/view-accessibility
  `accessibilityLabel`, `accessibilityHint`, `accessibilityValue`, `accessibilityAddTraits`, and `accessibilityIdentifier`. Use when the feature-dev or QA prompt requires accessible UI elements.

- **Dynamic Type Sizes** — https://developer.apple.com/documentation/uikit/uifont/scaling_fonts_automatically
  How Dynamic Type scaling works and how to verify layouts at all size categories.

---

## Security (Audit Reference)

OWASP controls and Apple's security architecture guidance.

- **OWASP Mobile Application Security Verification Standard (MASVS)** — https://mas.owasp.org/MASVS/
  The structured control set the security audit prompt is organized around. Defines L1/L2 requirements for data storage, cryptography, authentication, networking, and platform interaction.

- **OWASP Mobile Security Testing Guide (MASTG)** — https://mas.owasp.org/MASTG/
  Test cases and techniques for each MASVS control. Use when the security audit needs to verify a specific control with a concrete test procedure.

- **Secure Coding Guide (Apple)** — https://developer.apple.com/library/archive/documentation/Security/Conceptual/SecureCodingGuide/Introduction.html
  Apple's foundational guidance on input validation, privilege separation, and avoiding common memory and logic vulnerabilities.

---

## Dependency Management & Supply Chain

Third-party code, version pinning, and CVE exposure.

- **Swift Package Index** — https://swiftpackageindex.com
  Searchable index of Swift packages with compatibility, maintainability scores, and Swift version support. Use when evaluating a dependency before adding it to the project.

- **Package.resolved** — https://www.swift.org/documentation/package-manager/
  SPM's lockfile. The security audit prompt flags unpinned or wildcard versions; `Package.resolved` is what gives builds reproducibility. Verify it is committed and kept current.

---

## Tooling & Build

Xcode, Instruments, and CI configuration.

- **Xcode Release Notes** — https://developer.apple.com/documentation/xcode-release-notes
  What changed in each Xcode version. Check here when a build or test behavior changes unexpectedly after an Xcode update.

- **Instruments User Guide** — https://developer.apple.com/documentation/instruments
  Profiling tools for memory, CPU, hang detection, and network. Use when the build-app or refactoring prompt requires validating performance budgets or verifying a refactor did not regress latency.

- **Swift Macros** — https://developer.apple.com/documentation/swift/macros
  How attached and freestanding macros work and their compile-time expansion behavior. Use when the codebase relies on `@Observable`, `@Model`, or custom macros and you need to understand expansion.

---

## Code Quality & Architecture (Source Texts)

These are the intellectual sources `ios/common/engineering-principles.md` distills. Cite them when a principle is contested or needs deeper justification.

- **Clean Code** — Robert C. Martin (O'Reilly, 2008) — https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882
  Source for the naming, single-function, comments, and Simple Design four-rules principles in `engineering-principles.md`.

- **Clean Architecture** — Robert C. Martin (Pearson, 2017) — https://www.amazon.com/Clean-Architecture-Craftsmans-Software-Structure/dp/0134494164
  Source for the Dependency Rule and the Humble Object pattern, both named explicitly in `engineering-principles.md`.

- **Growing Object-Oriented Software, Guided by Tests** — Freeman & Pryce (Addison-Wesley, 2009) — https://www.amazon.com/Growing-Object-Oriented-Software-Guided-Tests/dp/0321503627
  Source for the outside-in TDD approach, the Humble Object pattern, and how tests drive good design.

- **Agile Testing** — Crispin & Gregory (Addison-Wesley, 2009) — https://www.amazon.com/Agile-Testing-Practical-Guide-Testers/dp/0321534468
  Source for the four-quadrant testing model in `shared/testing-quadrants.md`.

- **Design Patterns: Elements of Reusable Object-Oriented Software** — Gamma, Helm, Johnson & Vlissides (Addison-Wesley, 1994) — https://www.amazon.com/Design-Patterns-Elements-Reusable-Object-Oriented/dp/0201633612
  Source for "favor composition over inheritance" and "inheritance breaks encapsulation" in `ios/common/engineering-principles.md`, and for the named pattern targets (Strategy, State, Observer, Decorator, Factory Method, etc.) referenced in `ios/refactoring.md` as concrete refactoring destinations when replacing large conditionals with polymorphism.

- **Implementation Patterns** — Kent Beck (Addison-Wesley, 2007) — https://www.amazon.com/Implementation-Patterns-Kent-Beck/dp/0321413091
  Source for the field-lifetime and rate-of-change diagnostics in `ios/common/engineering-principles.md` (a field valid only during a method's execution belongs as a local, not persistent state; fields that change at different rates signal mixed responsibilities). Also the source of the Method Object move in `ios/refactoring.md`.

---

## Quick Reference — Slice to Primary Doc

| Slice | Go-to URL |
|-------|-----------|
| Swift language | https://docs.swift.org/swift-book/documentation/the-swift-programming-language/ |
| SwiftUI | https://developer.apple.com/documentation/swiftui |
| UIKit | https://developer.apple.com/documentation/uikit |
| Observation / @Observable | https://developer.apple.com/documentation/observation |
| Swift Concurrency | https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/ |
| Swift 6 migration | https://www.swift.org/migration/documentation/migrationguide/ |
| Networking (URLSession) | https://developer.apple.com/documentation/foundation/urlsession |
| Authentication (biometrics) | https://developer.apple.com/documentation/localauthentication |
| Secure Storage (Keychain) | https://developer.apple.com/documentation/security/keychain_services |
| Cryptography | https://developer.apple.com/documentation/cryptokit |
| Navigation | https://developer.apple.com/documentation/swiftui/navigationstack |
| Deep Links | https://developer.apple.com/documentation/xcode/supporting-associated-domains |
| Payments (StoreKit 2) | https://developer.apple.com/documentation/storekit/in-app-purchase |
| Push Notifications | https://developer.apple.com/documentation/usernotifications/unusernotificationcenter |
| Background Tasks | https://developer.apple.com/documentation/backgroundtasks/bgtaskscheduler |
| PDF (PDFKit) | https://developer.apple.com/documentation/pdfkit |
| Testing (Swift Testing) | https://developer.apple.com/documentation/testing |
| Privacy / Permissions | https://developer.apple.com/documentation/bundleresources/privacy-manifest-files |
| HIG | https://developer.apple.com/design/human-interface-guidelines/ios |
| Security audit (MASVS) | https://mas.owasp.org/MASVS/ |
| Security testing (MASTG) | https://mas.owasp.org/MASTG/ |

---

## Community Sources (second-priority — verify against Apple docs)

- **NSHipster** — https://nshipster.com — Well-cited articles on Apple frameworks and Swift idioms. Good for sparsely documented API surface areas.
- **Hacking with Swift** — https://www.hackingwithswift.com — Broad tutorial coverage. Verify against Apple docs before shipping.
- **Swift by Sundell** — https://www.swiftbysundell.com — Thoughtful articles on Swift design, architecture, and testing.
- **Point-Free** — https://www.pointfree.co — The primary external voice on functional Swift, TCA, and composition-root design.
