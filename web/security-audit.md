# Next.js Security Audit Prompt for Claude Code

> Acting as a principal cybersecurity engineer specializing in full-stack TypeScript and Next.js, perform a comprehensive security audit of this codebase. **Do not implement any fixes** — document findings only. Reason about the framework's real attack surface: Server Actions and route handlers are public endpoints, the Server/Client boundary can leak secrets, and server-side `fetch` is an SSRF vector.

---

## 0. App-Specific Context (fill this in before running)

<!-- The more you provide, the more precise the findings. Leave blank and the
agent will infer from the code. -->

- **Next.js version & router:** App Router / Pages Router / mixed
- **Rendering & server surface:** Server Components, Server Actions, route handlers, middleware, ISR/cached fetches
- **Auth:** NextAuth/Auth.js / Clerk / custom JWT / session strategy; cookie config
- **Data layer & DB:** Prisma / Drizzle / Kysely / raw SQL; engine
- **Sensitive data handled:** (PII, financial, health, legal, credentials, etc.)
- **Third-party SDKs at the boundary:** payments, email, auth, storage, AI, maps
- **File handling:** uploads/downloads, storage provider, presigned URLs
- **Hosting:** Vercel / containers / self-hosted; single vs multi-instance
- **Compliance obligations:** (GDPR, CCPA, HIPAA, PCI, SOC2, etc.)
- **Specific concerns or prior incidents:** 

---

## 1. Authentication & Session Management

- Review all auth flows: signup, login, password reset, email verification, account deletion
- Check session/token handling — JWT expiration, refresh-token rotation, secure storage (httpOnly + Secure + SameSite cookies vs `localStorage`)
- Look for session fixation or hijacking vectors; verify session invalidation on logout/password change
- Verify password hashing uses a modern algorithm (bcrypt/argon2/scrypt) with appropriate cost
- Check for account enumeration via login, signup, or reset error messages and timing
- Review OAuth/social login for proper `state`/PKCE validation and redirect-URI allowlisting

## 2. Authorization & Access Control

- Audit every route handler **and every Server Action** for authorization checks — Server Actions are publicly invocable regardless of which UI renders them
- Check horizontal privilege escalation: can a user manipulate IDs/params to access another user's records or files?
- Check vertical privilege escalation: can a regular user reach admin/privileged functionality?
- Review RBAC/permission logic for consistency across guards, hooks, and middleware — they should converge on the same decision
- Verify `middleware.ts` actually gates protected routes and its matcher covers them (no unprotected paths slipping through)
- Confirm unauthenticated users cannot reach any protected resource, action, or data fetch

## 3. Sensitive Data & File Protection

- Audit storage of sensitive data and user-uploaded files — encrypted at rest where required?
- Check that file/object URLs aren't guessable or sequentially enumerable; presigned URLs are time-limited and scoped to the owning user
- Look for PII over-exposure in API/Server Component responses — endpoints returning more than the client needs
- Check that sensitive fields are masked/redacted in logs, error messages, and responses
- Verify deleted accounts and associated data/files are purged or anonymized
- **Server/Client boundary:** flag secrets or sensitive fields passed from Server into Client Components (serialized into the client bundle/payload), and any secret read without the server-only guarantee (`server-only`, non-`NEXT_PUBLIC_` env)

## 4. Payment & Subscription Security

- Verify webhook signature validation on all incoming payment/provider webhooks
- Check that pricing, plan, and trial logic can't be manipulated client-side
- Review checkout for race conditions — can access be granted before payment confirms?
- Audit subscription/customer-portal access for authorization gaps
- Ensure provider secret keys aren't exposed client-side or in logs
- Check that failed/cancelled subscriptions properly revoke paid access, with revocation parallel to grant

## 5. Input Validation & Injection

- Check all inputs for SQL/NoSQL/command injection; verify parameterized queries / safe ORM usage
- Review XSS (stored, reflected, DOM-based) across rendering paths — especially `dangerouslySetInnerHTML`, user-generated content, and markdown/rich-text rendering
- Assess file-upload handling: type validation, size limits, storage location, filename sanitization
- Check path traversal in any file retrieval/download endpoint
- Verify runtime validation (Zod/Valibot/etc.) at request boundaries and that Server Action inputs are validated, not trusted

## 6. API & Server-Surface Security

- Audit rate limiting on public endpoints (login, signup, reset, contact, AI/expensive routes). Note in-memory limiters on multi-instance hosts — per-process counters are effectively unbounded
- Check mass-assignment in body parsing — are only allowed fields written?
- Review CORS configuration — overly permissive origins?
- Look for information leakage in error responses (stack traces, internal IDs, DB/constraint details)
- Verify content-type validation and method gating on handlers
- **SSRF:** review every server-side `fetch`/request built from user input (URLs, webhooks, image/redirect params, AI tool calls) for SSRF and metadata-endpoint access

## 7. Compliance & Regulatory Considerations

- Review audit-trail implementation — are sensitive actions (data access, account/permission changes, payments) logged immutably?
- Check that audit logs are protected from tampering/unauthorized access
- Assess data-retention and deletion policy — can users export and delete their data?
- Review cookie consent and tracking for compliance and correct gating
- Flag where legal disclaimers / ToS acceptance isn't properly gated or recorded

## 8. Dependency & Supply Chain

- Identify outdated dependencies with known CVEs
- Flag unpinned/wildcard versions
- Check for unused dependencies expanding attack surface
- Review lockfile integrity

## 9. Infrastructure & Configuration

- Review env-var handling — hardcoded secrets? Correct `NEXT_PUBLIC_` boundary (no secret accidentally public)?
- Check for debug mode / verbose logging / dev tooling enabled in production
- Audit Docker/container configs for privilege escalation or unnecessary exposed ports (if self-hosted)
- Review CI/CD configs (e.g., GitHub Actions) for secret leakage and unsafe workflow triggers
- Verify HTTPS enforcement and TLS config
- Review security headers: CSP, HSTS, X-Content-Type-Options, frame protections, and `next.config` header setup

## 10. Business Logic & Framework-Specific Risks

- Look for race conditions in payment flows, access grants, or state transitions
- Check privilege-escalation paths through multi-step workflows (e.g., onboarding → resource access)
- Review trial/freemium logic for bypass; referral/coupon/promo logic for abuse
- Check that ordered processes enforce sequencing and can't be skipped or replayed
- **Caching:** review for sensitive/user-specific data cached in shared caches (ISR, `fetch` cache, CDN) where it could leak across users, and for stale-after-mutation auth state

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

Begin the report with an executive summary showing a count of findings per severity level, plus a brief note on overall security posture and the two or three most urgent items to address first.
