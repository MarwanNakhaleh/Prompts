# Next.js Technical SEO Audit & Remediation Prompt for Claude Code

> Acting as a principal technical SEO engineer specializing in full-stack TypeScript and Next.js, perform a comprehensive technical and on-page SEO audit of this codebase, then remediate the findings. Reason about how search engines actually experience the site: what Googlebot receives on first fetch (not what hydrates later), what the rendered HTML declares about itself (metadata, canonicals, robots, structured data), and whether the site's architecture lets crawlers discover, render, index, and rank every page that deserves to rank — and nothing that doesn't.

Read `web/common/engineering-principles.md` — SEO regressions are frequently architecture regressions (metadata logic duplicated per-page instead of composed in layouts, Client Components that can't export metadata, framework boundaries leaking `"use client"` up the tree until nothing can be server-rendered).

This prompt is the **engineering half** of the organic engine defined in `cmo/content-seo-strategy.md`. That prompt decides *what to publish and why* (topic clusters tied to the buyer's jobs, downstream actions, distribution); this one ensures *what's published can actually be found, crawled, indexed, and win the click*. Borrow its judgments wherever an audit finding requires a content decision:

- **Rankings without downstream action are a museum.** When prioritizing fixes, weight pages by the downstream action they drive (signups, pipeline, activation), not raw traffic potential. A broken canonical on the pricing page outranks a missing meta description on a low-intent glossary term.
- **Start from the buyer's jobs, not the keyword tool.** When a finding requires new copy (title tags, meta descriptions, headings), write it for the problem the buyer is trying to solve, in their words — the keyword is the map, not the message.
- **Differentiate or don't publish.** Flag thin/doorway/near-duplicate pages as liabilities, not inventory. If a competitor could publish the same page, consolidation usually beats optimization.

**Search the web before auditing each slice.** SEO guidance moves fast (Core Web Vitals thresholds, INP replacing FID, structured-data types gaining/losing rich results, AI-overview/LLM citation behavior, Next.js Metadata API changes). Verify current Google documentation and Next.js docs for each concern before scoring findings against stale knowledge.

**Method — work from the crawler inward.**
- **Map the indexable surface first.** Enumerate every route the app can serve: static pages, dynamic segments, generated params, API-driven content pages. For each, determine: is it in the sitemap? Is it internally linked? What do its robots/canonical/status say? A page you never mapped is a page you never audited — and orphan pages (reachable only via sitemap, never via links) rank poorly.
- **Audit what the bot receives, not what the user sees.** Fetch or build-render key pages and inspect the actual HTML response: server-rendered content vs client-side-only, metadata present in the initial payload, status codes, redirect chains. `curl` the production site where one exists.
- **Combine automated and manual effort.** Lighthouse/PageSpeed, `next build` output, and sitemap/robots validators find the common ~80% cheaply; the deep ~20% (cannibalization, misdirected intent, internal-link architecture, E-E-A-T signals) needs human judgment against the SERPs actually being contested.

---

## 0. App-Specific Context (fill this in before running)

<!-- The more you provide, the more precise the findings. Leave blank and the
agent will infer from the code. -->

- **Next.js version & router:** App Router / Pages Router / mixed
- **Production URL & hosting:**
- **Rendering strategy:** SSG / ISR / SSR / client-heavy pages, per section
- **Target audience & buyer's jobs:** (from `marketing/customer-avatars.md` if available)
- **Downstream metric SEO is judged by:** signups / pipeline / activation — not traffic
- **Topic clusters / priority pages:** (from `cmo/content-seo-strategy.md` if run)
- **Primary competitors in the SERPs:**
- **Search Console access / known issues:** (coverage errors, manual actions, prior noindex incidents)
- **Specific concerns:**

---

## 1. Crawlability & Indexation

- Audit `robots.txt` (or `robots.ts`): is anything that should rank blocked? Is anything crawl-wasteful (search results, infinite params, API routes) left open?
- Audit meta robots / `X-Robots-Tag` across layouts and pages — verify no site-wide or inherited `noindex` leaks onto rankable pages, and that auth/dashboard/thin pages *do* carry `noindex`
- Verify the sitemap: every rankable page present, no 404/redirected/noindexed URLs in it, `lastModified` honest, correctly referenced from robots.txt, split/indexed if large
- Check status-code hygiene: soft 404s (empty states returning 200), redirect chains, 302s that should be 301s, trailing-slash and casing duplicates
- Verify canonical tags: self-referencing on rankable pages, absolute URLs, no canonicals pointing every page at the homepage, consistency with the sitemap and internal links
- Check www/apex and http→https consolidation to a single canonical host
- Confirm dynamic routes (`generateStaticParams`, catch-alls) don't mint unbounded or duplicate URLs

## 2. Rendering & Content Delivery

- Identify pages whose primary content only exists after client-side JS runs — Client Components rendering the money copy, data fetched in `useEffect`. Bots get an empty shell; move content server-side or pre-render
- Verify `generateMetadata` / `metadata` exports exist on every rankable route — Client Component pages silently ship no metadata at all
- Check ISR/SSG revalidation isn't serving long-stale content on time-sensitive pages
- Confirm error and not-found states return correct status codes, not 200s with error copy
- Check pagination, faceted, and parameterized views for crawl traps and duplicate-content spread

## 3. Metadata & SERP Presentation

- Audit `<title>` per page: unique, front-loaded with the page's term, honest length (~50-60 chars), no template stutter ("| Brand | Brand"), written for the buyer's job not the feature name
- Audit meta descriptions: present, unique, compelling enough to win the click against the competing SERP — a description is ad copy, not a summary
- Verify `metadataBase` is set and canonical/OG URLs resolve absolute in production
- Audit Open Graph + Twitter Card tags and images (correct dimensions, per-page where it matters) — social cards are a distribution surface, and LLMs/AI overviews read them too
- Check heading hierarchy: one H1 per page matching intent, logical H2/H3 structure that answers the buyer's actual questions
- Verify hreflang/locale tags if multi-language (or confirm N/A)

## 4. Structured Data

- Audit existing JSON-LD for validity and eligibility (test against current Google rich-result requirements — types gain and lose rich results over time)
- Identify missing high-value schema for the site's page types: `Organization`, `WebSite`, `Product`/`SoftwareApplication` + `Offer`, `Article`/`BlogPosting` with dates and author, `FAQPage` (where genuinely FAQ content), `BreadcrumbList`, `HowTo` where applicable
- Verify structured data matches visible page content (mismatch risks manual action)
- Check that schema is emitted server-side in the initial HTML, not injected client-side

## 5. Internal Linking & Site Architecture

- Map the link graph: are priority pages (by downstream action, per the CMO lens) within ~3 clicks of the homepage and linked from high-authority pages?
- Find orphan pages — in the sitemap but linked from nowhere
- Audit anchor text: descriptive and varied vs "click here"/"learn more" everywhere
- Check topic clusters link hub↔spoke both directions (pillar pages ↔ supporting content)
- Verify breadcrumbs exist on deep content and match `BreadcrumbList` schema
- Check nav/footer links are real `<a>` elements (crawlable), not JS-only handlers

## 6. Performance & Core Web Vitals

- Verify against **current** thresholds (search the web — metrics change; INP replaced FID): LCP ≤ 2.5s, INP ≤ 200ms, CLS ≤ 0.1
- Check image discipline: `next/image` everywhere, correct `sizes`, priority on LCP images, modern formats, explicit dimensions (CLS)
- Audit font loading: `next/font` vs layout-shifting external fonts
- Review bundle weight on rankable pages: `"use client"` creep, heavy libraries shipped to content pages that don't need them, dynamic imports for below-the-fold interactivity
- Check render-blocking third-party scripts (analytics, chat widgets) — `next/script` strategies
- Confirm compression, caching headers, and CDN behavior for static assets

## 7. Content Quality Signals (engineering-adjacent)

- Flag thin, near-duplicate, or doorway-pattern pages (template pages differing only by a swapped keyword) — recommend consolidation per the "differentiate or don't publish" rule
- Check for keyword cannibalization: multiple pages targeting the same term/intent, splitting authority
- Verify E-E-A-T scaffolding is renderable: author attribution on articles, dates (published + honest modified), an about page, contact info, org identity consistent with `Organization` schema
- Check outbound links on content pages cite real sources (and internal links pass authority to money pages)
- Confirm each rankable page has a path to its downstream action — a CTA appropriate to the page's intent stage; content that ranks but leads nowhere is the vanity trap

## 8. Off-Page Readiness & Distribution Hooks

- Verify social cards render correctly for the surfaces where the audience actually is (per the CMO's distribution plan)
- Check RSS/Atom feed exists for the blog if content is published regularly
- Confirm favicon/app icons and `site.webmanifest` are complete (brand presence in SERPs and tabs)
- Check that shareable assets (OG images) are branded and legible at thumbnail size

---

## Remediation (this prompt patches, unlike the security audit)

After the audit report is produced, **implement the fixes** in priority order:

1. **Fix immediately, no approval needed:** indexation blockers (bad robots/noindex/canonical), missing/broken metadata, sitemap errors, incorrect status codes, missing structured data, rendering fixes that move content server-side. These are engineering defects with objectively correct fixes.
2. **Propose before implementing:** anything that changes visible copy (titles/descriptions/headings are outward-facing messaging — draft them, but flag for review), page consolidation/deletion, redirects of live URLs, and anything altering information architecture. **Outward-facing copy is human-gated**, inheriting the gate from `cmo/content-seo-strategy.md`.
3. **Hand off, don't build:** net-new content pages belong to the content engine (`cmo/content-seo-strategy.md`); list them as opportunities with target term, intent stage, and the downstream action they'd drive.

Every fix follows the project's engineering standards: tests for changed metadata/sitemap/robots logic, type-check and lint clean, verified in a production build (`next build`) — metadata bugs frequently only manifest in the build output.

---

## Output Format

Organize all findings into a markdown report grouped by the categories above. For each finding include:

| Field | Description |
|-------|-------------|
| **Severity** | Critical (blocks indexing/ranking of money pages) / High / Medium / Low / Opportunity |
| **File & Line** | Exact file path and line number(s), or URL for production-only findings |
| **Description** | What the issue is and what the crawler actually experiences |
| **Impact** | Effect on discovery, indexing, ranking, click-through, or the downstream action |
| **Remediation** | The fix — and whether it's auto-fix (tier 1), gated (tier 2), or handed to content (tier 3) |

Begin with an executive summary: finding counts per severity, overall indexation health, and the two or three fixes most likely to move the **downstream metric** (not traffic). End with the list of tier-3 content opportunities handed to `cmo/content-seo-strategy.md`.
