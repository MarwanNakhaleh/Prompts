# iOS Platform & Slice Research — verified 2026-09-12

Dated research snapshot for `ios/` prompts. **Staleness warning:** everything here was verified against live official docs on 2026-09-12. Platform facts rot — re-verify version-specific claims (SDK levels, deprecations, pricing, endpoint behavior) before relying on them in a build after ~one quarter.

Companion to `ios/resources.md` (which lists the authoritative *sources*; this file captures *findings* from a verified research pass on the listed sources).

---

## 1. Platform state (Sept 2026)

| Item | Version | Notes |
|---|---|---|
| iOS stable | **26.6.2** (23G90) | iPadOS 26.7; iOS 27.0 RC (24A435) imminent — Apple ships GA the week after the Sept event |
| Xcode stable | **26.6** (17F113) | iOS 26.5 SDK, Swift 6.3 compiler |
| Xcode next | 27 RC (27A266a) | Swift 6.4, iOS 27 SDK, macOS Tahoe 26.6+ required, Apple-silicon-only, drops ld64 |
| Swift toolchain | 6.3.3 stable | 6.3 (Mar 2026) was C-interop-focused, not concurrency-focused |
| Deployment range | Xcode 26.6: iOS 15–26.5; Xcode 27: iOS 15–27 | |

### SwiftData — USE for offline single-user apps (~50MB), with gotchas

- Apple's default recommendation for new apps; iOS 17.0+; `ModelContainer` does automatic lightweight migration; `SchemaMigrationPlan`/`VersionedSchema`/`MigrationStage` for explicit control.
- **Known issues (forums, Aug–Sept 2026):**
  - `@Attribute(originalName:)` **broken for property renames** in lightweight migration (thread 843639, both Xcode 26.6 and 27) → use `MigrationStage.custom` for renames.
  - `Decimal` loses precision (stored as REAL, thread 838112) → **store money as Int cents**.
  - `#Predicate` can compile but crash at runtime on some optional/NULL cases (thread 842321) → guard nils before building predicates.
  - `@Query` over-invalidation refaults whole lists on unrelated saves (thread 840235) → keep `@Query` scopes narrow.
  - iOS 27 fixed an `@Query` background-actor deadlock; Xcode 27's `@Model` macro expansion now declares `Sendable`.
- Severe pain is CloudKit-sync-specific; offline-only stores are fine.

### Observation & concurrency

- `@Observable` (iOS 17+) is the standard, zero deprecations; `ObservableObject` is back-compat only.
- Xcode 27: `@State` reimplemented as macro (back-deploys iOS 17+); never assign `@State` both at declaration and in `init`. `PreviewProvider` deprecated → `#Preview`.
- Swift 6 language mode still opt-in; for greenfield apps set Swift 6 mode directly, or Swift 5 mode + "Approachable Concurrency" (SE-0466 default MainActor isolation). Typed throws (SE-0413) stable since 6.0.

### Deprecations relevant to typical apps (iOS 26/27)

`UIScreen.main` (26); `Text` `+` concatenation (26); legacy `NSPersistentStore` CloudKit option keys removed (26); URLSession min TLS → 1.2 for apps linked on iOS 26+ (overridable via `tlsMinimumSupportedProtocolVersion`); On Demand Resources → Background Assets (27); legacy MetricKit APIs → `MetricManager` (27). None for UserNotifications, NaturalLanguage, HealthKit core queries, NavigationStack, Keychain Services.

### Swift Testing

Bundled, actively extended (ST-0026–0029 in flight Aug–Sept 2026), significantly faster in Xcode 27. Valid default for new test targets; XCTest still supported.

---

## 2. OpenRouter API specifics (verified live 2026-09-12)

### Streaming (Chat Completions, `stream: true`)

- `Content-Type: text/event-stream`; `data: {JSON}` lines; terminates with `data: [DONE]`.
- **Keep-alive comment lines** like `: OPENROUTER PROCESSING` — skip any line starting with `:` before JSON-parsing or the parse throws.
- **Mid-stream errors arrive inside a 200**: a `data:` event with top-level `error` object + `finish_reason: "error"`; can be the *first and only* event. Also treat a stream ending without terminal `finish_reason` or `[DONE]` as a failure even if partial text rendered.
- **Tool-call deltas**: `choices[].delta.tool_calls[]`; first fragment carries `id`, `type: "function"`, `function.name`; later fragments only `function.arguments` pieces. Accumulate bucketed by `index`, concatenating `arguments`. `finish_reason: "tool_calls"` ends the batch.
- **Usage chunk**: `stream_options.include_usage` is accepted but deprecated/no-effect — full usage is ALWAYS included in a final chunk before `[DONE]`. OpenRouter deviation: that chunk repeats a choice with empty delta + terminal `finish_reason` (treat as accounting frame, not a second terminal). `usage` includes OpenRouter extras: `cost`, `cost_details`, `is_byok`, cached/reasoning token details.
- `X-Generation-Id` response header for correlation.
- **Cancellation stops billing** for DeepSeek, OpenAI, Anthropic, xAI; NOT for Google, Groq, Mistral, Bedrock (they bill fully).
- Streams are not resumable (no `Last-Event-ID`). On mid-stream failure: keep partial text marked incomplete, re-request from scratch, user-visible retry (paid tokens; auto-retry only for pre-stream transport failures).
- Zero Completion Insurance refunds zero-token responses.

### Swift consumption pattern

`URLSession.bytes(for:delegate:)` (iOS 15+) → `for try await line in bytes.lines`; skip empty + `:`-prefixed lines; strip `data: `; `[DONE]` ends; tolerant decoding (optional `choices`, check top-level `error` first, optional tool-call fragments). Dedicated ephemeral `URLSession`: `timeoutIntervalForRequest = 120` (idle-gap timer, reset by each chunk/keep-alive), `timeoutIntervalForResource = 600`, `waitsForConnectivity = true`. No new SSE API in iOS 26 SDK; Foundation Models framework (iOS 26, on-device LLM) exists but has no embeddings API.

### Zero Data Retention

- Per-request enforcement: `"provider": {"zdr": true}` — OR-ed with account/guardrail settings (request-level can only add enforcement, never remove).
- Programmatic ZDR endpoint list: `GET https://openrouter.ai/api/v1/endpoints/zdr` (306 unique models as of 2026-09-12).
- **ZDR covers inference routing only — NOT plugins/tools** (web search is third-party operated) — disclose in UI.
- Account-level per-model-group ZDR toggles exist; per-request `zdr` composes.

### Image generation

- Dedicated API: `POST /api/v1/images` (base64 `b64_json` + `media_type`); model discovery `GET /api/v1/images/models`; all-or-nothing billing; params: `resolution` (512/1K/2K/4K), `aspect_ratio`, `size`, `quality`, `n` (≤10), `input_references` (image-to-image), `provider.*` routing; SSE streaming for partial images on capable models.

### Video generation

- Submit → poll → download: `POST /api/v1/videos` (returns polling URL), poll endpoint, content download endpoint; `GET /api/v1/videos/models`; webhook cookbook exists. **Video models are NOT in the ZDR list** (no ZDR coverage today).

### Verified model lineup + ZDR coverage (2026-09-12 pricing)

| Role | Default | Alt | $/M in | ZDR providers |
|---|---|---|---|---|
| Chat | `deepseek/deepseek-v4-flash-0731` | `deepseek/deepseek-v4-flash` | $0.07 / $0.04 | DeepInfra, SiliconFlow, Novita, Azure, +more |
| Vision | `qwen/qwen3-vl-235b-a22b-instruct` | `z-ai/glm-4.6v` | $0.21 / $0.30 | DeepInfra, Novita, Parasail, Venice / Novita, Z.AI |
| Critic | `z-ai/glm-4.7-flash` | `qwen/qwen3.7-flash` | $0.06 / $0.03 | Novita, Venice (qwen ZDR unverified) |
| Image | `bytedance-seed/seedream-4.5` (ZDR, ~$0.05/img) | `qwen/qwen-image-3` (no ZDR) | per-image | Seed provider |
| Video | `bytedance/seedance-2.0-mini` | 2.0 / 2.0-fast / 2.5 | per-job | none |
| Local | `mlx-community/Qwen3-4B-Instruct-2507-4bit` | 8bit / DWQ-2510 variants | — | n/a (on-device) |

Chinese chat models verified cheap: qwen3.7-flash $0.03/M, deepseek-v4-flash $0.04/M, glm-4.7-flash $0.06/M.

---

## 3. On-device embeddings / RAG (English, ~20k chunks)

**Primary: `NLEmbedding.sentenceEmbedding(for: .english)`** — iOS 14.0+, current, not deprecated. **512-dim** sentence vectors, dynamic (no fixed vocab; works on arbitrary sentences). Embed chunks at index time via `vector(for:)`, store vectors yourself (Apple: "nearest-neighbor search therefore doesn't apply to sentence embeddings"), embed query at runtime, brute-force cosine yourself (`NLDistanceType.cosine` semantics). Model is BiLSTM-era (2020): quality is decent for chat-history retrieval; if retrieval quality tests weak, fallback = `all-MiniLM-L6-v2` (384-dim) converted to Core ML (~90MB weights, ANE, 5-20ms/sentence). `NLContextualEmbedding` (iOS 17+) is per-token BERT-style — Apple's docs explicitly say "for semantic similarity tasks, consider using NLEmbedding." Foundation Models (iOS 26) has NO embedding API.

**Retrieval: brute force wins at this scale.** 20k × 512-dim dot products ≈ 10M MACs → ~1-5ms via Accelerate (`cblas_sgemv` or vDSP); normalize vectors at index time so dot = cosine; 20k×512×4B = 40MB float32 (20MB float16). No Apple vector index exists (SwiftData indexes: binary/R-tree only). ANN is pointless under ~100k vectors. Indexing pass over 20k chunks: ~1-3 min, run incrementally in background.

---

## 4. UserNotifications (accountability scheduling)

### Scheduling pattern: individual dated occurrences, NOT repeating triggers

`UNCalendarNotificationTrigger(repeats: true)` reschedules itself but is opaque: can't cancel next Tuesday's instance alone, can't express an escalation ladder, can't carry per-occurrence state. **Use one dated request per (occurrence × escalation stage)** with deterministic identifiers (e.g. `session-2026-09-14-0730`, `...-ladder-60`, `...-ladder-180`, `...-ladder-morning`), and keep escalation state in your own DB. Removal is `removePendingNotificationRequests(withIdentifiers:)`; "modify" = remove + re-add (don't rely on add-overwrites).

**64-pending-request cap per app** (soonest-firing kept; repeating triggers count as one). 5 sessions/week × 4 requests ≈ 20/week → keep a ≤3-week rolling window and replenish on every launch/foreground by diffing `getPendingNotificationRequests` against expected. Treat the system list as a cache; your DB is the source of truth. Delivery is best-effort by design — never infer a missed workout from a missed notification.

### Actions & categories

`UNNotificationAction` ("Done for Today") in a `UNNotificationCategory` (register via `setNotificationCategories` at launch; set `content.categoryIdentifier`). Categories show ≤2 actions in constrained presentations — order matters. Receipt via `UNUserNotificationCenterDelegate.userNotificationCenter(_:didReceive:)` — matches `actionIdentifier` (`UNNotificationDefaultActionIdentifier` for plain tap). **Cold start works** (system launches app in background) but the delegate MUST be assigned before app finishes launching → `UIApplicationDelegateAdaptor` setting `center.delegate` in `willFinishLaunching`. Implement `willPresent` (`.banner/.list/.sound`) or foreground notifications silently vanish. Delegate callbacks may arrive off-main — hop to MainActor before touching UI state. Deep-link payload in `content.userInfo` (+ optional `targetContentIdentifier`); route via an `@Observable` router holding `NavigationPath`.

### Time Sensitive (iOS 26 change)

`.timeSensitive` interruption level "breaks through system notification controls" (with user permission; per-app toggle readable via `UNNotificationSettings.timeSensitiveSetting`). **iOS 26: `UNAuthorizationOptions.timeSensitive` is deprecated — "Use time-sensitive entitlement"**: add the Time Sensitive Notifications capability (entitlement `com.apple.developer.usernotifications.time-sensitive`), set `content.interruptionLevel = .timeSensitive`, don't pass the deprecated auth option. Reserve for escalation rungs, not routine reminders. Reserve `.active` (default) for everything else; system Focus/DND suppresses it during the user's quiet windows — no app-side schedule needed. AlarmKit is NEW in iOS 26 (prominent alarms, own authorization) — candidate for a hardest-rung wake-up.

### HealthKit read-only snapshot

- `requestAuthorization(toShare: [], read: [.stepCount, .heartRate, .sleepAnalysis, .workoutType()])`; `success` does NOT mean granted; requires `NSHealthShareUsageDescription`; gate on `HKHealthStore.isHealthDataAvailable()`. Users may grant **limited recent windows** — probe with `getRequestStatusForAuthorization` / `earliestAuthorizedSampleDate(for:)`.
- Queries: steps = `HKStatisticsCollectionQuery` (daily anchors) for series, `HKStatisticsQuery` `.cumulativeSum` for a day; heart rate = `.discreteAverage`/`.discreteMinMax` statistics; sleep = `HKSampleQuery` on `sleepAnalysis` (iOS 16+ stage model: `inBed` + overlapping stage samples; sum `allAsleepValues` intersected with the day; watch data has edge gaps); workouts = `HKSampleQuery` (statistics queries cannot run over workouts). `HKAnchoredObjectQuery` is overkill for a recomputed 14-day rolling snapshot.
- **Denial is indistinguishable from empty** (privacy design; `authorizationStatus(for:)` reflects share/write only). Never render "permission denied" from query results, never gate onboarding on read success; neutral empty states + non-accusatory "Connect Apple Health" affordance.

---

## 5. Tooling notes

- Xcode 27 requires macOS Tahoe 26.6+, Apple silicon only.
- `xcodegen` generates `.xcodeproj` from `project.yml` — reproducible project setup, avoids pbxproj merge conflicts (critical when parallel agents add files: agents edit sources; regenerate project from `project.yml` instead of hand-editing pbxproj).
- Build/test via `xcodebuild -project ... -scheme ... -destination 'platform=iOS Simulator,name=...'`; `CODE_SIGNING_ALLOWED=NO` for simulator unit-test builds.

---

## 6. Distribution / TestFlight gotchas (learned 2026-09-13)

**Privacy manifest is mandatory, and its absence fails SILENTLY.** A build uploaded without `PrivacyInfo.xcprivacy` (or missing required-reason declarations for APIs the binary references, e.g. UserDefaults → CA92.1) is accepted by the delivery service ("Upload succeeded"), then quietly disappears during ASC processing: it never appears in TestFlight, no error in xcodebuild, and the report (ITMS-9105x series) goes ONLY to the account's notification email. Signature of this failure: export/upload fully green + zero build rows in TestFlight after >20 min → check the email, check the app bundle for the manifest. Every app ships a privacy manifest from day one: `NSPrivacyTracking false`, empty tracking domains, empty collected-data types (for on-device apps), and accessed-API declarations with reason codes.

**Purpose strings must cover what the BINARY references**, not just what the code intends: static analysis flags framework usage (e.g. any `HKHealthStore` symbol requires `NSHealthShareUsageDescription` present in Info.plist at upload time). Read-only HealthKit needs only the Share string; don't declare Update unless writing.

**ASC API key roles for CLI uploads:** `xcodebuild -allowProvisioningUpdates` with an API key does cloud signing (creates certificates + profiles with no local CSR dance), but **Admin-role keys are required**; App Manager keys get "Cloud signing permission error." Keys can't be role-upgraded after creation — generate a second key at the right role. The key's `-authenticationKeyPath` must be an absolute path to an existing file; keep deploy scripts in sync with where the key actually lives.

**Adopting an existing ASC app record:** if a prior app record exists for the concept (never submitted), adopting it (switch the project's bundle ID to the record's, reuse the unique name) beats fighting ASC name-uniqueness. Watch for: pending Program License Agreement updates block ALL uploads/new apps until the Account Holder accepts at developer.apple.com/account.
