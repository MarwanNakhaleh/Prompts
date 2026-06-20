# iOS Security Audit Prompt for Claude Code

> Acting as a principal cybersecurity engineer specializing in iOS application security, perform a comprehensive security audit of this iOS codebase. **Do not implement any fixes** — document findings only. Where relevant, reference the OWASP Mobile Application Security Verification Standard (MASVS) and current Apple platform security guidance. Treat the device as potentially hostile: assume an attacker may control a jailbroken, hooked, or MITM'd device.

Read `ios/common/engineering-principles.md` — violations of the dependency rule, framework boundaries, and composition root are frequent root causes of security findings (client-side enforcement that belongs server-side, secrets leaking across a boundary, trust placed in a layer that can be bypassed).

**Method — work from the attack surface inward, and think like an attacker.**
- **Map the attack surface first.** Before scoring findings, enumerate every untrusted input and reachable resource: custom URL schemes and universal links, every input field, pasteboard reads, app-group/shared-container and shared-keychain access, app-extension boundaries, `WKWebView` JS bridges, every network endpoint, and every file the app reads. This inventory is the spine of the audit — a vulnerability you never mapped is one you never tested.
- **Drive testing from abuse/misuse cases, not just features.** For each capability, write the adversary's version: "as an attacker on a jailbroken/hooked device, can I bypass the biometric gate by manipulating the callback? read secrets out of the keychain/backup? feed a crafted deep link to trigger an unauthorized action? MITM this request?" Prioritize by architectural risk and known mobile attack patterns, not by code coverage.
- **Combine automated and manual effort — the 80/20 rule.** Static analysis (examines source/binary without running), dynamic/runtime analysis (Frida/objection, MITM proxy), and dependency/CVE scanners find the common ~80% cheaply and belong in CI; the deep ~20% (chained logic flaws, client-side trust assumptions, business-logic abuse) needs a human adversarial mindset. Note where either half is absent.
- (Search the web for "attack surface mapping," "abuse case / misuse case testing," and "OWASP fuzzing / fault injection" to expand these techniques.)

---

## 0. App-Specific Context (fill this in before running)

<!-- The more you provide, the more precise the findings. Leave blank and the
agent will infer from the code. -->

- **UI framework / language:** SwiftUI / UIKit / mixed; Swift version
- **Min deployment target:** 
- **Sensitive data handled:** (PII, financial, health, legal docs, credentials, location, etc.)
- **Backend & auth model:** REST/GraphQL; OAuth / custom JWT / session; token lifetime & refresh
- **Persistence:** Keychain / SwiftData / Core Data / Realm / SQLite / UserDefaults / files
- **Networking stack:** URLSession / Alamofire; cert pinning yes/no
- **Monetization:** StoreKit 2 / StoreKit 1 / third-party; server-side receipt validation yes/no
- **Third-party SDKs & analytics:** (list any that touch sensitive data)
- **Compliance obligations:** (GDPR, CCPA, HIPAA, COPPA, PCI, etc.)
- **Distribution:** App Store / TestFlight / enterprise / MDM-managed
- **Specific concerns or prior incidents:** 

---

## 1. Data Storage & Protection at Rest

- Flag any sensitive data (tokens, credentials, PII, financial/health data) stored in `UserDefaults`, plists, or unencrypted Core Data / SwiftData / Realm / SQLite stores
- Verify the Keychain is used for secrets with an appropriate accessibility class (prefer `...WhenUnlockedThisDeviceOnly`); flag overly permissive `...Always` or device-syncing of secrets that shouldn't sync
- Check Data Protection / file-protection classes on sensitive files (`NSFileProtectionComplete` vs. weaker)
- Review Secure Enclave usage for key material where appropriate
- Check that sensitive data isn't leaked into caches, temp directories, or written to disk unintentionally by frameworks
- Verify sensitive data is excluded from iCloud/iTunes backups where required (`isExcludedFromBackup`, Keychain backup attributes)

## 2. Authentication, Session & Biometrics

- Review all auth flows: login, token refresh, logout, password reset, email verification, account deletion
- Check token/session storage — tokens in Keychain, not `UserDefaults`/plaintext; expiration and refresh-rotation handling; concurrent-refresh races
- Audit biometric auth (`LocalAuthentication` / Face ID / Touch ID): is the result trusted correctly or bypassable by manipulating the callback? Is the gate backed by a Keychain ACL (`SecAccessControl`) rather than a boolean?
- Verify biometric/passcode gates can't be trivially bypassed on jailbroken or hooked devices
- Check for account enumeration via auth error messages surfaced in the app
- Check for credentials/tokens logged to console or analytics

## 3. Network & Transport Security

- Review App Transport Security config in Info.plist — flag `NSAllowsArbitraryLoads` or per-domain exceptions and assess justification
- Confirm all traffic uses TLS; flag any cleartext HTTP
- Review certificate/public-key pinning (if present): correctness, pin-rotation strategy, failure handling; note absence if the threat model warrants pinning
- Audit `URLSession` configuration and any custom `URLSessionDelegate` server-trust evaluation — flag code that overrides/weakens trust validation (a common backdoor accepting invalid certs)
- Check for sensitive data in URL query strings or in logged request/response bodies

## 4. Sensitive Data Leakage on Device

- Check app snapshot/backgrounding: is sensitive content masked before the app enters the background (app-switcher exposure)?
- Audit pasteboard usage — sensitive data on the general `UIPasteboard`; is it non-persistent / excluded from Universal Clipboard / cleared?
- Review logging (`print`, `NSLog`, `os_log`) for sensitive data, especially in release builds
- Check secure text entry and keyboard caching on sensitive fields (`isSecureTextEntry`, correct `textContentType`)
- Review screenshot / screen-recording / mirroring exposure on sensitive screens

## 5. Inter-Process Communication & Deep Links

- Audit custom URL scheme and universal/associated-link handling — is all incoming input treated as untrusted and validated? Can a crafted link trigger unauthorized actions, navigation, or state change? Assess resistance to **fuzzing / fault injection** on these entry points: malformed, oversized, or unexpected-type parameters, encoding tricks, and crafted payloads (injection/path-traversal) that happy-path parsing would mishandle
- Review `openURL`/scene handling for injection or unintended side effects
- Check App Groups, shared containers, and shared Keychain access groups for over-broad sharing
- Audit app extensions (share, widget, notification, custom keyboard) and their trust boundaries and data flow
- Review pasteboard-based IPC and document-provider interactions

## 6. Platform Interaction & WebViews

- Audit `WKWebView`: is JavaScript required, is file access locked down (`allowFileAccessFromFileURLs` etc.), and are loaded origins controlled?
- Review JS bridges (`WKScriptMessageHandler`, `evaluateJavaScript`) for injection and for exposing native capability to web content
- Flag any use of deprecated `UIWebView`
- Confirm untrusted HTML/URLs aren't rendered with native privileges; validate web-content origins

## 7. Cryptography

- Prefer `CryptoKit`/CommonCrypto over hand-rolled crypto; flag weak/deprecated algorithms (MD5, SHA-1, DES, ECB)
- Check key generation, storage, and lifecycle — keys in Keychain/Secure Enclave, not hardcoded or predictably derived
- Look for hardcoded keys, IVs, or salts in source or resources
- Verify secure randomness (`SecRandomCopyBytes`/`CryptoKit`) for security-sensitive use, not `arc4random`-style
- Review any custom TLS, at-rest encryption, or token-signing logic

## 8. Permissions, Privacy & Entitlements

- Review Info.plist purpose strings — flag permissions exceeding actual need (least privilege)
- Audit the entitlements file — App Groups, Keychain sharing, associated domains, background modes all justified?
- Check the Privacy Manifest (`PrivacyInfo.xcprivacy`) for accurate declared data collection and required-reason API usage
- Review third-party SDK data collection against the app's stated privacy posture
- Check handling of sensitive permissions (location, contacts, camera, mic, photos, health) and data minimization

## 9. Runtime & Binary Hardening

- Assess anti-tampering controls appropriate to the threat model: jailbreak/root detection, debugger detection, integrity checks — and how failures are handled
- Check for hooking/instrumentation defenses where warranted (Frida/objection, swizzling abuse)
- Identify sensitive logic enforced only client-side that belongs server-side
- Note reliance on security-through-obscurity and whether critical controls are client-side bypassable
- Confirm debug aids (debug menus, test endpoints, verbose logging) are compiled out of release builds

## 10. Secrets, Build & Configuration

- Scan for hardcoded secrets: API keys, tokens, credentials, signing material in source, plists, `.xcconfig`, asset catalogs, bundled files
- Review build configs — debug flags, test/staging endpoints, or relaxed settings leaking into release
- Check `DEBUG` conditional compilation excludes insecure paths from release
- Audit CI/CD and fastlane/match for secret leakage and secure signing-identity handling
- Review how environment config/secrets are injected at build time
- Flag real production or personal data (PII, financial, health) bundled as seed/fixture/sample files in the app target or committed in test targets — it ships inside the IPA or lives in the repo unscrubbed; use masked, referentially-valid synthetic data instead

## 11. Third-Party Dependencies & Supply Chain

- Inventory dependencies across SPM, CocoaPods, and Carthage
- Identify outdated dependencies or those with known CVEs
- Flag unpinned/wildcard versions; review lockfile integrity (`Package.resolved`, `Podfile.lock`)
- Flag unused/unmaintained dependencies expanding attack surface
- Review embedded binary frameworks/XCFrameworks of unknown provenance and the data/permissions they pull in

## 12. Business Logic & Server Trust

- Identify security decisions made only on-device that should be server-enforced (entitlements, feature gating, payment/subscription state, validation)
- Review in-app purchase / receipt validation — server-side and tamper-resistant?
- Look for race conditions in payment, access-grant, or multi-step workflows
- Check that critical state transitions can't be skipped or replayed by a tampered client

---

## Output Format

Organize all findings into a markdown report grouped by the categories above. For each finding include:

| Field | Description |
|-------|-------------|
| **Severity** | Critical / High / Medium / Low / Informational |
| **File & Line** | Exact file path and line number(s) |
| **Description** | What the vulnerability is |
| **Impact** | What an attacker could achieve by exploiting it |
| **Remediation** | Recommended fix approach |
| **MASVS Ref** | Relevant OWASP MASVS control, where applicable |

Begin the report with an executive summary showing a count of findings per severity level, plus a brief note on the app's overall security posture and the two or three most urgent items to address first.