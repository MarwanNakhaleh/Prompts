# Next.js Technical SEO Audit Prompt for Claude Code

> Acting as a principal technical SEO engineer specializing in Next.js, perform a comprehensive SEO audit of this codebase and its production deployment. **Do not implement any fixes in the audit run** — document findings only, tiered by severity, then hand the approved fixes to `web/feature-dev.md` (or apply them in a separate, explicitly approved implementation pass). Reason about how Google actually processes the site: what Googlebot can fetch, render, index, and choose to rank — in that order — because a failure at any earlier stage makes every later optimization worthless.

Read `web/common/engineering-principles.md` — configuration-at-high-levels and never-baking-environment-values-into-the-artifact matter here (`NEXT_PUBLIC_*` vars are build-time; a canonical or site URL burned in at build with the wrong value ships wrong everywhere).

**Method — audit in pipeline order: fetchability → indexability → canonicalization → content → CTR. Never start with content.**

- **Verify against production HTML, not source.** Fetch live pages (`curl -s https://domain/ | grep -i 'robots\|canonical'`) and check GSC's URL Inspection. The single most expensive failure mode this prompt exists to prevent: a site-wide `noindex, nofollow` (or a site-wide canonical pointing every page at the homepage) sitting in a root layout for months while everyone optimized content on pages Google was forbidden to index. Source can lie — middleware, headers, and hosting config can override it.
- **A blocker at stage N invalidates all work at stages > N.** If the site is noindexed, stop and flag it as an emergency finding before auditing anything else.
- **Use Search Console data as the diagnostic, not vanity reporting.** Query-level impressions + clicks + position tell you *which stage* is failing (see §8).
- **Study who actually ranks.** For each money query, check the live SERP: who wins, with what page type, title pattern, and schema. Mimic the winning patterns; don't invent.

---

## 0. App-Specific Context (fill this in before running)

<!-- The more you provide, the more precise the findings. Leave blank and the
agent will infer from the code. -->

- **Production domain & canonical host:** apex vs www; any legacy domains
- **Next.js version & router:** App Router / Pages Router / mixed
- **Rendering:** which marketing/content pages are static vs dynamic vs client components
- **Content sources:** MDX / JSON / CMS / hardcoded pages; slug derivation
- **Search Console access:** query/page/impression data available? (paste top queries if so)
- **Money queries:** the 5-15 searches a buyer would use to find this product
- **Known competitors ranking for those queries:**
- **Prior SEO incidents or migrations:**

---

## 1. Fetchability & Crawl (can Googlebot get the bytes?)

- `robots.txt` / `robots.ts`: confirm nothing blocks **render assets** (`/_next/static/`, fonts, CSS, JS). Blocking them "for crawl-budget hygiene" breaks Googlebot's rendering and is a real regression seen in the wild — disallow app/dashboard routes, never the asset pipeline
- Host canonicalization at the infrastructure level: HTTP→HTTPS and www↔apex must 301 (load balancer / `next.config` `redirects()`), not serve duplicate content on both hosts. Verify with `curl -sI` on all four variants
- Confirm no auth wall, geo block, or bot challenge in front of marketing pages
- Check response codes on every sitemap URL — no 404s, no redirect chains

## 2. Indexability (is Google allowed to index it?)

- **Grep every layout for `robots` metadata first.** A `noindex` in a root or shared layout silently blocks the whole subtree. Marketing pages must be indexable; auth/dashboard/utility layouts should carry their **own** noindex rather than relying on a site-wide default in either direction
- Verify meta robots in **production** HTML for the homepage and 3-4 key pages, and cross-check GSC's "Excluded by noindex" count
- Utility pages (login, signup, password reset, legal) — noindexed and excluded from the sitemap, so they don't absorb impressions the content pages should get
- `X-Robots-Tag` headers on generated routes (OG images, feeds) that shouldn't index

## 3. Canonicalization & Duplication

- **No site-wide `alternates.canonical` in a shared layout** — it canonicalizes every non-overriding page to one URL (usually the homepage), which de-indexes the rest. Every indexable page sets a **self**-canonical
- Canonical URLs use the production host from env, and remember `NEXT_PUBLIC_*` is resolved at **build** time — verify the built artifact, not just the code
- Route collisions: a static `page.tsx` silently shadowing a dynamic `[slug]` route serving richer content
- Content clusters cannibalizing each other (several thin posts targeting one query) — flag for consolidation via 301s

## 4. Sitemap & Feeds

- Every real content page present; no noindexed or utility pages
- `lastModified` is honest (derived from git or content dates), **not** `new Date()` on every build — fake freshness erodes trust in the whole sitemap
- Slug derivation is total: a missing `slug` field in one content JSON produced `/blog/undefined` URLs in a real audit. Test the slug path, not just the happy case
- Sitemap submitted in GSC; RSS/Atom feed exists for content sections

## 5. On-Page Metadata

- Unique title + description per page; **no duplicated brand suffix** (a title template *plus* a manually appended brand yields "Page | Brand | Brand")
- Client Component pages can't export `metadata` — they silently inherit the parent layout's title/description. Give them a segment `layout.tsx` or convert; a "use client" page with a `metadata` export is a build error
- Titles front-load the target query and give a reason to click (a concrete artifact: "with openssl commands", "free checker", a number). Descriptions under ~160 chars
- `metadataBase` set; OG/Twitter present on shareable pages

## 6. Structured Data

- JSON-LD emitted through an escaping helper (escape `<` — an unescaped `</script>` in content breaks out of the script tag; this is an XSS-adjacent bug, not just an SEO one)
- **FAQ/HowTo schema must mirror content visibly rendered on the page** — schema-only Q&A violates Google policy and risks manual action
- Organization + WebSite site-wide; Article with required `image` and a real `author`; BreadcrumbList backed by visible breadcrumbs; Product/Offer on pricing
- Validate with the Rich Results Test on production URLs

## 7. Internal Linking & Information Architecture

- **Orphan audit:** every tool, guide, and content page reachable from homepage nav/footer/hub. Free-tool and guide pages are the most commonly orphaned — and often the highest-intent pages on the site
- One dedicated page per primary keyword. **The homepage absorbing most impressions across many queries is a smell**, not a success: it means no dedicated page exists to rank precisely
- Contextual inline links between cluster pages (guide ↔ tool ↔ comparison), not just nav links
- Breadcrumbs on deep content

## 8. Search Console Query Analysis (diagnose the funnel stage)

Pull top queries with impressions, clicks, and position, then bucket by intent before reacting:

- **Product intent** (query = what you sell): these pages must exist and rank. Missing page → build it; ranking page with 0 clicks → title/CTR problem
- **Adjacent intent** (searcher is mid-task on a neighboring problem): not "wrong traffic" — a content-funnel opportunity. Build the definitive answer and route it to the product
- **True mismatch** (homonym/branding collision): usually low volume; handle with a disambiguation page, not a panic rename
- **Impressions with zero clicks are a position or title problem, not a positioning problem.** Check average position first: page 2-3 → ranking work; page 1 with no clicks → rewrite the title/description. Do not conclude "wrong audience" or "rename the product" from impression data alone — impressions on adjacent queries mean Google considers you *relevant to the niche*, which is the hard part
- Fresh or recovering sites: expect 2-6 weeks after an indexing fix before judging; note the recheck date

## 9. Competitive SERP Analysis

- For each money query, record who ranks 1-3, the page **type** (tool / guide / glossary / listicle / docs), title pattern, and schema in use
- Identify the repeatable patterns winners share (URL structure, topic clusters, free-tool interlinking, visible FAQs) and the gaps where no strong page exists — those gaps are the cheapest wins
- Comparison pages ("X vs Y", "free alternatives to Z") rank and convert **because they are fair** — acknowledging competitor strengths is a ranking feature, not a concession
- Note listicles/directories that own commercial queries; getting *into* them is off-page work worth listing

## 10. Content Strategy Fit (report gaps, don't write content here)

- Map existing pages to the intent buckets from §8; list missing dedicated pages for product-intent and high-volume adjacent-intent queries
- Troubleshooting/error-message queries ("X not received", "Y failed") are high-intent and weakly contested in B2B niches — flag them as a cluster if absent
- Every content page ends in a next step tied to the product (tool run, signup), with the CTA matched to the searcher's stage
- Check voice/quality guardrails exist (no AI-tell phrasing, technical specificity); flag thin or duplicate-`html_content` style data bugs in content files

---

## Output Format

Organize findings into a markdown report grouped by the categories above, **tiered**:

| Tier | Meaning |
|------|---------|
| **T1 — Emergency** | Blocks indexing/fetching entirely (site-wide noindex, blocked render assets, homepage-pointing canonical). Fix and deploy before anything else |
| **T2 — High** | Wrong-host duplication, sitemap lies, schema policy violations, orphaned money pages |
| **T3 — CTR/Rank** | Title/meta rewrites, internal-link additions, position improvements |
| **T4 — Content gaps** | Missing pages per intent bucket, cluster consolidations (route to content work) |

For each finding: file & line (or production URL), what's wrong, how it was verified (source vs production HTML vs GSC), impact, and the remediation. Begin with an executive summary: finding counts per tier, the current funnel-stage diagnosis (fetch / index / rank / CTR / content), and the recheck date for any in-flight recovery. End with the handoff list for `web/feature-dev.md` and, where GSC access exists, the exact pages to request reindexing for after deploy.
