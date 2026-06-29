# MCP Resources

Organized by application slice. When a prompt instructs you to research current spec, SDK, or upstream behavior, start here — these are the authoritative, regularly-updated sources. Official documentation (the MCP spec, the official SDKs, each vendor's API docs, OWASP) takes precedence over training-data memory and community convention whenever they conflict. The MCP spec is versioned and dated and the vendor servers move fast, so confirm current behavior against these sources rather than recalling it.

---

## MCP Specification & Core Concepts

The protocol itself: the tool / resource / prompt primitives, the transports, capability negotiation, and the authorization model for remote servers — the decisions that determine what a server can expose and how a host connects to it.

- **MCP Documentation Home** — https://modelcontextprotocol.io
  The canonical entry point for the protocol: what tools, resources, and prompts are, how a server declares them, the client/server lifecycle, and the conceptual model `mcp/build-server.md` Phase 2 requires you to ground yourself in. Prefer this over memory; primitives and content types are added and stabilized between spec revisions.

- **MCP Specification** — https://spec.modelcontextprotocol.io
  The dated, versioned protocol specification — the authoritative wording for the JSON-RPC message shapes, the initialize/capability-negotiation handshake, structured output and content types, and server/client capabilities. Always check which spec version you are targeting; `mcp/build-server.md` Phase 2 asks you to flag anything recently added, stabilized, or deprecated (elicitation, sampling, resource links) because it changes the design.

- **MCP Transports** — https://modelcontextprotocol.io/docs/concepts/transports
  The two transports the spec defines — **stdio** (local, simplest, the default for a v1 server) and **streamable HTTP** (remote) — and their trade-offs. `mcp/build-server.md` Phase 1 and Phase 3 turn this choice into the installation, auth, and security-surface decision: local stdio carries no OAuth obligation; a remote endpoint does.

- **MCP Authorization Specification** — https://modelcontextprotocol.io/specification/draft/basic/authorization
  The authorization model for **remote** servers: OAuth 2.1 with PKCE, the authorization-server metadata discovery flow, and token handling. This is the spec obligation behind `mcp/build-server.md`'s rule that a remote endpoint carries an OAuth 2.1 + PKCE requirement that local stdio does not. Verify the current draft/version — the auth portion of the spec has changed materially across revisions.

---

## MCP SDKs

The official libraries that implement the protocol so you build tool handlers, not JSON-RPC plumbing. Python + FastMCP is the common default and aligns with most vendor SDKs; the TypeScript SDK is the alternative. Neither is a web framework.

- **MCP Python SDK (FastMCP)** — https://github.com/modelcontextprotocol/python-sdk
  The official Python SDK and the **default** for this category. FastMCP's decorator API registers tools, resources, and prompts and handles the handshake and transport. The reference for the server skeleton, capability negotiation, and structured output in `mcp/build-server.md` Phase 4 when Python is the chosen baseline. Check the README for the current minimum runtime and transport support before pinning a version.

- **MCP TypeScript SDK** — https://github.com/modelcontextprotocol/typescript-sdk
  The official `@modelcontextprotocol/sdk` package — the **alternative** baseline when the team is TypeScript-first. Same protocol coverage as the Python SDK (tools, resources, prompts, stdio + streamable HTTP). It is an MCP server SDK, **not** a web framework; `mcp/build-server.md` Phase 1 flags this explicitly so a server isn't accidentally built as an Express app.

- **MCP Servers (reference & examples)** — https://github.com/modelcontextprotocol/servers
  The official collection of reference server implementations and examples. Consult it before building from scratch — `mcp/build-server.md`'s "compose official servers before reimplementing" rule starts here: if a reference or official server already wraps an upstream, proxy it rather than rebuilding its API. Also the best source of current, idiomatic tool-registration patterns for both SDKs.

---

## Tool & Prompt Design

The tool name, description, and argument schema are the model's entire interface to a capability — the highest-leverage reliability lever, and the part `mcp/build-server.md` keeps in versioned prompt files separate from adapter code.

- **MCP Tools concept** — https://modelcontextprotocol.io/docs/concepts/tools
  How tools are declared, named, described, and given input schemas, plus annotations (read-only / destructive hints) the host can surface. The authoritative reference for the I/O and tool-description conventions in `mcp/build-server.md` Phase 4 — validate your own inputs strictly at the boundary, return structured output you own, and treat the description + schema as the contract the model reads.

- **MCP Prompts concept** — https://modelcontextprotocol.io/docs/concepts/prompts
  How a server exposes reusable prompt templates as a primitive. Distinct from the versioned tool-description prompt *files* `mcp/build-server.md` keeps for the model-swap boundary; consult this when a server should expose prompts as a first-class capability rather than only tools.

- **Anthropic Documentation** — https://docs.anthropic.com
  The home for Anthropic's guidance on tool use and on writing tools and tool descriptions that an autonomous model calls correctly — descriptions, naming, argument schemas, and error messages as a reliability concern. Use the current tool-use and agent-tooling pages here as the canonical reference behind `mcp/build-server.md`'s "tool descriptions are the interface to the model" principle. (Locate the current tool-use page from the docs index rather than assuming a fixed slug; the structure changes.)

---

## Auth, OAuth & Secrets

Whose credentials a server uses, the OAuth flow for remote servers and upstreams, and where tokens live — keyed by an abstract account, least-privilege, and revocable.

- **OAuth 2.1** — https://oauth.net/2.1/
  The authorization-framework consolidation the MCP remote-server auth model builds on. The reference for the grant types, the security hardening over OAuth 2.0, and the flow `mcp/build-server.md` Phase 3 requires for any remote endpoint and for per-upstream OAuth.

- **PKCE (Proof Key for Code Exchange)** — https://oauth.net/2/pkce/
  The code-exchange protection MCP mandates for remote-server auth and that you should use for every authorization-code flow. The reference for the `code_verifier` / `code_challenge` mechanism named in the spec's authorization section and in `mcp/build-server.md` Phase 3.

- **OWASP Cheat Sheet — Secrets Management** — https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
  Authoritative guidance on where secrets live, rotation, and never committing them to code, logs, or git. Backs `mcp/build-server.md`'s non-negotiable: tokens and client secrets live in the credential store / environment, structured per-account keyed by an abstract `account_id` with the narrowest scopes that do the job, and there is always a revocation/teardown path — and they never leak into logs or the append-only audit trail.

---

## Input Validation & Structured Output

Strict schema enforcement at the tool boundary, and tolerant reading of upstream responses — validate your own inputs strictly, read external responses leniently.

- **Pydantic Docs** — https://docs.pydantic.dev
  The standard validation layer for Python tool handlers (and the model layer FastMCP uses for tool argument schemas). The reference for declaring strict input models at the boundary and for the response shapes a tool returns. `mcp/build-server.md` Phase 4 requires validating every tool input with a schema before processing; in Python this is the tool.

- **Zod Docs** — https://zod.dev
  The TypeScript-side equivalent — schema validation for tool inputs and structured output when the TS SDK is the baseline. Used the same way the web category uses it: shared parse-at-the-boundary validation that rejects malformed calls before they reach an adapter.

- **OWASP Cheat Sheet — Input Validation** — https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html
  Allowlist-over-blocklist strategies, canonicalization, and injection-resistant validation. The principle behind "validate your own inputs strictly; read external responses tolerantly (extract only the fields you use)" in `mcp/build-server.md` Phase 3 and Phase 4 — a tool is called by an autonomous model, so its boundary must reject oversized, malformed, and wrong-scope inputs explicitly.

---

## Upstream Ad Platforms (example upstreams)

These are **example** upstreams — the first server built in this category wraps ad platforms, so the build prompt's "verify each upstream's current capability yourself" instruction needs concrete targets. The pattern generalizes to any API a server wraps; capabilities and auth flows here change fast and **must be re-verified** against the live docs, never recalled.

- **Meta Marketing API** — https://developers.facebook.com/docs/marketing-apis/
  The official Meta (Facebook/Instagram) ads API a server wraps for campaign data and management. There is also an official Meta Ads MCP / AI connector effort — check Meta for Developers for its current status, what it exposes, and whether it is read-only or write-capable before deciding to compose it versus calling the API directly. Per-account OAuth with least-privilege scopes; app review may gate the write scopes.

- **Google Ads API** — https://developers.google.com/google-ads/api/docs/start
  The official Google Ads API: GAQL queries, account hierarchy, and campaign management. The reference for any capability the MCP server can't get from the official MCP server below.

- **Google Ads MCP Server** — https://developers.google.com/google-ads/api/docs/developer-toolkit/mcp-server
  Google's official open-source MCP server for the Ads API — typically **read-only** (GAQL `search`, `list_accessible_customers`). The composable upstream server for read paths; writes go through the Google Ads API directly. The concrete example behind `mcp/build-server.md`'s "compose official servers before reimplementing" rule. Verify its current capability surface against this page — it expands over time.

- **TikTok Marketing API** — https://business-api.tiktok.com/portal
  The TikTok for Business Marketing API (v1.3 at time of writing) for campaign and ad-group data and management. Pair with the TikTok for Developers portal below for app registration and the Business API SDKs. Note: **no GA official MCP server yet**, and an app audit may be required before production scopes are granted — confirm the current state before assuming a capability exists.

- **TikTok for Developers** — https://developers.tiktok.com/
  App registration, OAuth, scope catalog, and the official Business API SDKs for the TikTok upstream. The reference for the auth flow and the SDK an adapter wraps when there is no official MCP server to compose.

---

## Resilience & Reliability

Every upstream call has a timeout, retries are bounded and backed off, mutations are idempotent, and a slow or failing upstream degrades one capability rather than hanging the whole server.

- **Release It! patterns (Michael Nygard)** — https://pragprog.com/titles/mnee2/release-it-second-edition/
  The source text for the resilience vocabulary `mcp/build-server.md` Phase 3 uses — Circuit Breaker, Bulkhead, Timeout, and the stability anti-patterns (cascading failure, unbounded retry). Cite this when a resilience choice is contested. Also listed under Source Texts below.

- **Polly resilience patterns** — https://www.pollydocs.org
  A concrete, well-documented catalog of resilience strategies (retry with backoff and jitter, circuit breaker, timeout, bulkhead isolation, fallback) with clear semantics. A stable reference for *how* each pattern behaves when wiring the resilience helpers, regardless of implementation language.

- **Azure Architecture — Resiliency patterns** — https://learn.microsoft.com/azure/architecture/patterns/category/resiliency
  Microsoft's pattern catalog for retry, circuit breaker, bulkhead, throttling, and compensating-transaction patterns. Use alongside Polly when you need the pattern's intent and trade-offs rather than a specific API.

- **Tenacity** — https://tenacity.readthedocs.io
  The standard Python retry library — bounded attempts, exponential backoff with jitter, and retry-on-exception predicates. The concrete tool for the "bounded retries with backoff" requirement in `mcp/build-server.md` Phase 3 when the baseline is Python. Pair retries with idempotency keys on mutations so a retried write doesn't double-apply.

---

## Testing & Evals

Tools tested without hitting live upstreams, the agreed call→result examples as contract tests, manual exercising via the Inspector, and evals that measure tool-call reliability per runtime model — mapped across the four quadrants.

- **pytest** — https://docs.pytest.org
  The standard Python test runner for unit tests of verbs/policy in plain functions with adapter fakes (Q1) and example-driven contract tests for the agreed tool call→result examples (Q2). `mcp/build-server.md` Phase 4 stands up the test harness with adapter fakes here; `mcp/feature-dev.md` mirrors the convention per capability.

- **Vitest** — https://vitest.dev
  The TypeScript-side test runner, used the same way when the TS SDK is the baseline — unit tests for verbs/policy and contract tests for tool examples.

- **MCP Inspector** — https://github.com/modelcontextprotocol/inspector
  The official tool for manually exercising a server's tools, resources, and prompts over either transport without a full host — the Q3 exploratory pass. Use it to verify the steel-thread tool in `mcp/build-server.md` Phase 4 connects end-to-end (transport, auth, adapter, policy seam) before tools pile on, and to reproduce a misbehaving tool call by hand.

- **Agile Testing Quadrants** — https://lisacrispin.com/2011/11/08/using-the-agile-testing-quadrants/
  Lisa Crispin's explanation of the Q1–Q4 model that `shared/testing-quadrants.md` distills and that `mcp/build-server.md` Phase 3's test strategy is organized around. Use when deciding which tier a test belongs in. For MCP, the Q4 non-functional corner explicitly includes the **eval harness** that measures tool-call reliability — because prompts and tool descriptions tuned against one runtime model rarely transfer cleanly, evals must be re-run per model whenever the model is swappable.

- **Recorded-fixture / contract testing** — see `shared/testing-quadrants.md`
  The recorded-fixture approach to faking upstreams: capture a real upstream response once, replay it as a fixture, and assert the adapter maps it to the response shape you own. This keeps the agreed call→result examples as fast, deterministic contract tests without hitting a live API — the "fakes / recorded contract fixtures" path in `mcp/build-server.md` Phase 1 and Phase 4.

---

## Observability

Structured logging and tracing for tool calls and upstream latency — so a misbehaving autonomous caller and a slow upstream are both diagnosable, without leaking secrets or PII into the logs.

- **OpenTelemetry Docs** — https://opentelemetry.io/docs/
  The vendor-neutral standard for traces, metrics, and logs. The reference for tracing a tool call from registration through the policy layer and an adapter to the upstream and back, and for measuring per-upstream latency against the timeouts set in `mcp/build-server.md` Phase 3. Structured logs and spans must never contain secrets or PII, consistent with the audit-log rule.

---

## Deployment & Transport Hosting

stdio for local; for a remote server, hosting that supports a long-lived, streamable-HTTP connection rather than a short request/response function.

- **Railway Docs** — https://docs.railway.com
  Container-based hosting suited to a long-lived remote MCP endpoint over streamable HTTP, with managed state and private networking. Use when `mcp/build-server.md` Phase 1 selects a remote transport and the server needs a persistent connection a short-lived serverless function can't hold. For local stdio servers no hosting is needed — the host launches the process directly. In general, prefer any host that supports long-lived / streamable connections and per-account secret storage; avoid functions with short hard timeouts for the streamable-HTTP transport.

---

## Code Quality & Architecture (Source Texts)

The intellectual sources `mcp/common/engineering-principles.md` distills. Cite these when a principle is contested or needs deeper justification.

- **Clean Code** — Robert C. Martin (O'Reilly, 2008) — https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882
  Source for naming, single-function, comments, and Simple Design four-rules principles in `engineering-principles.md` — the readability and deletability bar `mcp/build-server.md` holds tool handlers and adapters to.

- **Clean Architecture** — Robert C. Martin (Pearson, 2017) — https://www.amazon.com/Clean-Architecture-Craftsmans-Software-Structure/dp/0134494164
  Source for the Dependency Rule and the Humble Object pattern named in `engineering-principles.md` — the ports-and-adapters boundary that keeps the core verbs and policy independent of the SDK, the vendor APIs, and the transport in `mcp/build-server.md` Phase 3.

- **Growing Object-Oriented Software, Guided by Tests** — Freeman & Pryce (Addison-Wesley, 2009) — https://www.amazon.com/Growing-Object-Oriented-Software-Guided-Tests/dp/0321503627
  Source for the outside-in, write-test → write-code → run → learn discipline and the "test it without the server" approach behind the steel-thread slice in `mcp/build-server.md` Phase 4.

- **Design Patterns: Elements of Reusable Object-Oriented Software** — Gamma, Helm, Johnson & Vlissides (Addison-Wesley, 1994) — https://www.amazon.com/Design-Patterns-Elements-Reusable-Object-Oriented/dp/0201633612
  Source for "favor composition over inheritance" and for the named pattern targets (Adapter, Strategy, Facade) behind the per-upstream adapter boundary and the policy seam in `engineering-principles.md`.

- **Implementation Patterns** — Kent Beck (Addison-Wesley, 2007) — https://www.amazon.com/Implementation-Patterns-Kent-Beck/dp/0321413091
  Source for the field-lifetime and rate-of-change diagnostics in `engineering-principles.md` — the lens for keeping per-request state local and separating things that change at different rates (adapter code vs. tool-description prompts).

- **Release It!** — Michael T. Nygard (Pragmatic Bookshelf, 2nd ed. 2018) — https://pragprog.com/titles/mnee2/release-it-second-edition/
  Source for the resilience material in `mcp/build-server.md` Phase 3: the Circuit Breaker, Bulkhead, and Timeout stability patterns and the cascading-failure / unbounded-retry anti-patterns they prevent. The reference whenever a resilience requirement needs justification.

- **Building Microservices** — Sam Newman (O'Reilly, 2nd ed. 2021) — https://www.amazon.com/Building-Microservices-Designing-Fine-Grained-Systems/dp/1492034029
  Background for "draw boundaries along axes of change" and "identify the actors who drive change" in `mcp/build-server.md` Phase 5's decomposition — each upstream is one actor, the policy layer is its own actor, and a new upstream should be addable as a new adapter chunk without touching existing ones (Open-Closed).
