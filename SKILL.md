---
name: pagination-independence-architecture
description: "Design category and listing architecture so that a site's indexable value does not depend on paginated pages, using demand-backed indexable subcategories as the alternative to deep pagination. Load this skill when the user asks whether to noindex, canonicalise, block or prune paginated URLs (`?page=`, `?p=`, `/page/2/`); when they mention pagination handling, page-2+ indexation, infinite scroll and crawlability, orphaned product or article pages, \"products only reachable from page 7\", crawl budget wasted on listing pages, thin or duplicate category pages, faceted navigation index bloat, or splitting a large category into subcategories. Also load for \"how do we cut paginated URLs without losing products\" and listing-page taxonomy redesign for large catalogues. Applies to e-commerce catalogues, editorial archives and directories."
version: 1.0.0
license: CC-BY-4.0
metadata:
  author: Eoghan Henn and contributors
  source: https://github.com/rebelytics/pagination-independence-architecture
---

# Pagination-Independence Architecture

This skill makes a large listing-based site independent of its paginated pages before any
pagination pruning happens, by designing demand-backed leaf subcategories that surface every
item on page 1 of at least one listing — turning a defensive index-hygiene job into new ranking
surface.

## Core principle

Before pruning a class of URLs from the index, re-architect so the internal link graph no longer
depends on them. Pagination handling is the last step, not the first. If deep pages are the only
route to a chunk of the catalogue, any noindex, canonical, redirect or crawl restriction applied to them
degrades the discoverability of the items behind them — no matter how correct the directive looks
in isolation.

The agent must therefore refuse to hand over a pagination directive as a standalone
recommendation. It delivers a sequenced plan: audit → subcategory design → coverage verification
→ conditional pruning, rolled out per category.

## Phase 1 — Pagination-independence audit

Establish which items are reachable *only* through pagination.

### Step 0 — define the paginated set from the linked surface, not from a URL pattern

Before configuring any crawl or query, fetch page 1, page 2 and a deep page of one representative
listing per template, and record four things:

- **(a)** The pagination URL pattern *as actually emitted in hrefs* — not the platform's documented
  convention. Verify it with counting probes: a plausible parameter can return zero links while an
  undocumented variant (an infinite-scroll form such as `&p=`) carries the whole set.
- **(b)** Which page URLs the pagination block links to: a full page list, next-only, or next+last.
  This decides crawl depth and what "reachable" means for anything deep in the sequence — see
  *What the crawl cannot tell you* below.
- **(c)** The canonical target of each paginated page. This is evidence in its own right — if every
  paginated page already canonicalises to page 1, the site itself declares them non-canonical, which
  is often the substantive finding the whole audit is chasing, available from one fetch of the head.
- **(d)** Whether a page-1 parameter variant exists (`?p=1`, `?page=1`) and whether anything links to
  it. It commonly returns 200 with content identical to the bare listing URL while appearing nowhere
  in the link graph.

Define the "paginated source" class from (a) and (b). A URL pattern describes what can be
constructed; the link graph describes what exists, and the two diverge in **both** directions — a
matching variant that is never linked would be mis-classified as a legitimate paginated source,
and the linked surface is usually far sparser than the URL space. Every analysis or pruning decision
that turns on classifying paginated sources must derive the class this way. The divergence is cheap
to measure now and expensive to discover after the numbers are quoted.

If the analysis runs through a crawler or link-analysis platform with a page-type **segmentation**
layer, check before building any query that paginated listing URLs are their own segment class.
Where the platform's API has no negation operator, the segmentation *is* the negation operator:
"all sources except pagination" can only be expressed as "sources in [every other class]", which
requires pagination to be a named class. Paginated URLs almost always match their parent listing's
pattern, so they collapse into it unless deliberately carved out by an earlier-ordered rule. This is
a hidden precondition of the whole audit — fixing the segmentation upstream is cheap; working around
a missing class inside the query is not.

### Audit steps

1. Crawl the site with all pagination routes excluded: exclude pagination parameters and path
   patterns, and disable execution of any infinite-scroll or "load more" mechanism, so the crawler
   sees only what a link graph without pagination exposes. Where listings load items client-side,
   confirm whether page 1 renders its items in the initial HTML at all — if it does not, that is a
   separate finding that must be fixed before anything else in this workflow means anything.
   Record the crawl's **depth limit** and whether it **seeds from XML sitemaps**. Both are part of
   the finding and are quoted alongside it.
2. Build a full item inventory independently: XML sitemaps, a full unrestricted crawl, or a
   database or feed export. Prefer at least two sources and reconcile them. Where the question is
   what the crawl could not reach, at least one source must be assembled by a method other than
   traversal — a sitemap, a product feed, a database export, server log files, Search Console, or
   the CMS's own item count.
3. Diff the two. Items present in the inventory but absent from the pagination-free crawl are
   **pagination-orphaned** — provided the crawl finished. Before reading any absence off a
   crawl, read the live queued/waiting count from the crawler's progress state, not a listing's
   rounded completion percentage (a few hundred queued URLs against 100k fetched rounds to
   100%). An item in the crawl's queue was discovered via a link and simply not fetched: it is
   not orphaned, and the queue concentrates exactly the late-found, lightly-linked items the
   diff is hunting. Export the queued URLs and subtract them from the absent set, or state that
   the crawl finished with an empty queue. Recipe:
   `references/measurement-recipes.md` ("Queue guard before any absence diff").
4. **Constrain both endpoints to live URLs before ranking anything.** Filter link sources to
   HTTP 200 — links from redirecting pages are not real links — *and* filter targets to HTTP 200 as
   well. Removed items (404/410), redirects and noindexed targets belong out of the population, not
   at the top of it. A "worst-linked first" ordering surfaces deleted inventory before it surfaces a
   real discovery problem, and reads as urgent precisely because the numbers are extreme; delisted
   items often remain linked from a sale category's deep pagination long after removal. Inspect the
   top few rows of any such ranking manually before quoting the number — the extreme tail is where
   population-definition artefacts concentrate. Filtering can invert the headline: "two items are
   reachable only via pagination" becomes "no live item is", which redirects the analysis from
   *whether* pagination is a sole access path to *how much* of an item's link equity depends on it.
5. Attribute each orphan to its category, and — on multi-market or multilingual sites — to its
   market. Orphan rates commonly differ sharply per market because catalogue depth and taxonomy
   depth differ.
6. Report per category: total items, items on page 1, orphan count, orphan share, page count —
   as a breakdown of the item-level coverage set (see *Measure coverage at item level* below),
   never as a ranking of listings by size.

### What the crawl cannot tell you

A dataset gathered by traversal cannot evidence the absence of pages it was never able to reach,
and its internal consistency actively conceals the gap. Two reasoning errors follow, and both must
be closed before any pruning decision rests on crawl data.

**The unreachable band is the middle, not the end.** Use the link pattern from Step 0 (b) to
work out which pages a depth-limited crawl could actually reach. With next+last pagination, page 1
links to page 2 *and* to page N, so both ends of the sequence sit at shallow depth; depth then
accumulates inward from page 2 forwards and — where a previous-link or `rel=prev` exists — from the
last page backwards. Under a depth limit, the reachable set is the first several pages plus a band
near the end, and the unreachable region is the **middle**. High page numbers routinely record
shallow depths for exactly this reason. Sorting by page number and assuming depth rises
monotonically gets this precisely backwards.

**Missing items may be missing from the crawl, not from the site.** Without sitemap seeding, an item
linked only from an unreachable middle page is absent from the crawl entirely — not merely absent
from its link graph — so every crawl-derived metric agrees the item does not exist and the analysis
returns a confident, self-consistent, wrong answer. Seeding inverts the signature: with it, such
items appear in the crawl with zero inlinks; without it, they are missing altogether. The seeding
setting therefore determines which query finds them.

So never state "no item depends on pagination for discovery" from crawl data alone. Qualify the
conclusion with the depth limit and the seeding configuration, and confirm it by set difference
against the independent inventory from step 2. The comparison, not the crawl, is the instrument.

**The crawl also has a vintage, and the two deliverables it supports age at different rates.**
The structural conclusion this workflow exists to produce — whether any item depends on
pagination as its sole access path — is about the *shape* of the link graph and survives weeks
of drift. The orphan table's per-URL rows are claims about specific pages right now, and decay
continuously from the moment the crawl completes. The same export therefore supports one
deliverable and not the other, and the distinction is invisible unless the vintage is known;
nothing in the workflow forces the question, which is why it belongs in the pre-flight.

- Resolve and record the crawl's **completion date** alongside every figure, and state the
  orphan list's as-of date in the deliverable. A report labelled "Last Crawl" is not a date —
  resolve the tool's actual crawl-date / crawl-name value.
- Ship aggregate and structural conclusions on an older crawl if need be. Do **not** ship the
  per-URL orphan or low-inlink list that way: live-verify a stratified sample first and report
  the observed stale rate, or regenerate from a fresh crawl.

The failure shape: a verified orphan list whose crawl date was never established. A
stratified sample finds a sizeable share of the "orphans" linked today — from prose inside
their category's description block, `index,follow`, self-canonical, no nofollow, exactly the
links the report says do not exist — because the descriptions were written after the crawl
ran. Part of the list is already fixed at delivery, and nobody can say which part.

### Test reachability at anchor level, not by substring

Any manual check of whether an item is linked from a listing page must match the anchor that
carries the link — a pattern including `<a` and `href=` with the item's URL — never a bare item-id
substring. Modern listing templates ship a JSON product payload (dataLayer, hydration state) that
mentions far more item ids than the grid actually links, so a substring test answers "does this id
appear in the HTML", which is not the question. Useful cross-check: count product anchors per page
and compare against the count of id substring matches — a large gap means the substring test is
reading the payload, not the grid. And treat an arithmetically impossible result — an item
"appearing" on more pages of a stable-sorted sequence than the sequence can contain — as a
measurement fault, not a site fault: re-examine the method before hypothesising about the site,
because impossibility is the cheapest error signal available and it is absent from every plausible
wrong answer. The corrected test can invert the diagnosis entirely — from "hidden behind deep
pagination" to "linked from nowhere in its own category", a different problem with a different fix.

### Source classification and measurement traps

Every per-item inlink count rests on a definition of "source", and the definition's failure cases
apply to every item uniformly — they shift the whole distribution by one and never look like
outliers. Apply these rules before quoting any count:

- **Remove self-links first.** Crawler inlink totals commonly include the page itself (breadcrumb
  last node, share links, a language switcher pointing at the same URL), so every item carries a
  phantom +1 and "exactly one non-paginated source" is, for most of its members, zero plus self.
- **Label near-universal structural sources separately** — HTML sitemap page, sitewide menus,
  footer "all products". They make every item look linked. Report **reachability** (any
  same-domain, non-self, non-paginated source) and **editorial linking** (a category body,
  related-item or content link) as two numbers.
- **Multi-domain crawl projects: separate same-domain from cross-domain sources.** Language or
  country switcher and alternate links from sister domains are links between separate sites and
  do not count toward internal pagination-independence. They are also why a multi-domain tool can
  show no item below one non-paginated source while a single-domain crawl, which cannot see them
  at all, shows hundreds.
- **Both sides of any subtraction from one snapshot.** Never subtract a live-fetched count from a
  crawl-stored count on a page set containing rotating listings (sale, new-in, bestseller,
  featured): everything that changed between the two time points becomes a finding, and the
  false negatives are silent. Same crawl for both sides, or exclude the rotating listings from
  both and say so.
- **Apply the crawl-depth bound as a free pre-filter.** A pagination-only item has crawl depth ≥ 3
  (home → page 1 → page N → item). Items at depth ≤ 2 are excluded without any inlink data, and a
  method that flags a depth-≤2 item as pagination-only is broken — reject the method, not the item.
  Check what makes the bound pass before reading it as a property of the architecture: if a
  complete HTML sitemap (or similar all-items page) is linked at depth 1, every item passes
  trivially and the bound proves reachability only. Then measure page-1 coverage before
  concluding that listings carry discovery, and report the HTML sitemap as the sole
  non-paginated source for the items off page 1.
- **Test listing stability before treating a pagination-only set as a fixed remediation list.**
  An item on two pages of one listing within a crawl (once the anchor-level method is confirmed
  sound), or on different pages across crawls, means the sort drifts. Then "pagination-only" is a
  snapshot of a process, and the remedy is structural — the page-1 ceiling, stable related-item
  links, a complete HTML sitemap as backstop — not linking one snapshot's list, because a
  different set drifts onto page 2+ next week.
- **Read an absence off a fetched page only after checking the response is complete.** When
  verifying that a parent listing links a subcategory, filter page or item, confirm a marker that
  must be present (filter panel, product grid, footer) and a size in line with sibling fetches,
  test the link class on several pairs, and cross-check against crawl depth and actual inlink
  sources rather than the inlink count alone. A partial response contains no links to anything.
- **Read paginated URLs' analytics from a dimension that keeps the query string.** GA4's
  "Landing page" dimension strips the query string, so `?p=N` landings fold into the base URL
  and a zero for them is an instrument artefact, not evidence that paginated pages get no
  entries. Use "Landing page + query string", or Search Console page data.

**The cheap test that precedes any aggregate:** pull the actual inlink sources of the single
weakest item the method produced and classify each by hand — self, paginated, cross-domain,
structural, editorial. One read of the right row exposes a phantom source in every row. And when
an early verification turns up something odd but apparently harmless, write down what it would
imply if it were systematic before deciding it is minor; an anomaly noticed and rationalised is
costlier than one never seen. On one mid-sized catalogue, exactly this inverted a "no item depends
on pagination" finding into a material orphan share. Source-class table and the observed
cases: `references/measurement-recipes.md` ("Classifying inlink sources") and
its sibling sections on same-snapshot consistency, listing stability and fetched-page absence.

### Measure coverage at item level

The pass condition is "every item linked from page 1 of at least one listing", so measure it
directly, at item level, as Phase 1's primary metric: fetch page 1 of every listing at one
moment, extract only the product grid (not menu, footer or cross-sell blocks), union the grids,
and diff against the independent inventory from step 2. The residual is the coverage gap. Two
rules travel with it:

- **Validate the extractor against a count the page declares about itself** — an inline size
  variable, a "showing 1–36 of 412" element, a payload field. Extracted grid count must equal
  min(declared size, items-per-page) on every listing; a mismatch means the extractor is reading
  something other than the grid. The same check exposes zero-item hub pages that look like top
  categories and provide no coverage.
- **Do not rank listings by size to find uncovered items.** Listing-size tables double-count
  when facets cross-cut: the largest "leaf" listings in a big catalogue are attribute facets
  over the same items, many already on page 1 of a smaller listing, and cross-cutting facets are
  leaves by URL. The size table misleads most in exactly the catalogues where the problem is
  biggest.

Record, per uncovered item, the earliest page it appears on in any listing — that distribution
sizes the items-per-page lever in Phase 2. Recipe:
`references/measurement-recipes.md` ("Item-level page-1 coverage from validated grid
extraction").

The output of Phase 1 is the item-level coverage gap and its per-category, per-market breakdown,
with each count's source classes, snapshot and queue state recorded. It is the input to every
later decision and the baseline the rollout is measured against.

**Worked example (constructed).** A catalogue category holds 1,200 items at 30 items per listing
page: 40 pages, 30 items linked from page 1, 1,170 pagination-orphaned (97.5%). A directive applied
to pages 2–40 in this state removes the only internal path to 97.5% of that category. The same
category after Phase 2 holds eight leaf subcategories of 80–250 items each; at 30 per page none
are single-page, so the design is revised — either the per-page count rises or the subcategories
split further — until page-1 coverage is met.

## Phase 2 — Design demand-backed leaf subcategories

Subcategories are the alternative to deep pagination. Design them inside the existing main
category structure, not as a parallel taxonomy.

Constraints every candidate subcategory must satisfy:

- **Demand-backed.** Each subcategory maps to real search or user demand — query data, internal
  site-search logs, filter-usage data, or a keyword source. A subcategory nobody searches for is
  index bloat with extra steps. A keyword tool's volume commonly sits on one lead variant of a
  concept: query at least the singular, the plural and the common qualified form and take the
  max before calling a term demand-less, and report which variant carried it.
- **Above the floor.** A minimum item count, typically around six to eight, below which the page
  is too thin to justify indexation. Set the floor explicitly with the user and record it.
- **Below the ceiling.** Small enough that page 1 holds all or nearly all of its items. This is the
  binding constraint: derive the ceiling from the site's items-per-page setting, and **size the
  items-per-page lever before designing any subcategory** — from the earliest-page distribution
  of uncovered items (Phase 1), compute the share each candidate page size brings onto a page 1
  and where the residual concentrates, so design targets the residue and the listings that hold
  it. Price the lever on per-item markup and DOM plus server render time (a staging test at the
  target size); below-the-fold card images are commonly lazy-loaded already, and the cost must
  be measured as a function of item count, never generalised from the first cards, which are
  the above-the-fold exceptions by construction.
- **Covering.** Every item in the catalogue must be linked from page 1 of at least one
  subcategory. Coverage is the test the whole design passes or fails on.
- **Curated.** Where subcategories are generated from facet combinations, only the curated,
  demand-backed combinations become indexable and internally linked. Everything else stays
  non-indexable. This is index governance, and it is part of the deliverable.

### From the coverage gap to a page list

"Items not on page 1" is not the scope of subcategory work. Coverage decides what a subcategory
*could* fix; demand decides where one is *worth having*; the two are different sets, and
conflating them builds zero-demand pages for the wrong reason. Work in this order:

1. **Inventory existing landing-page mechanisms first.** Clean-URL filter landing pages, "shop
   by" blocks or merchandising pages the platform already serves are the vehicle for new
   subcategories before any new mechanism is specified. A clean slug in markup does not mean a
   landing page exists — check each against the crawl's listing set, and verify parent → child
   links under the fetched-page rule from Phase 1.
2. **Pull exact facet membership** from the platform's filter endpoint where it exposes one
   (facet counts say how big a value is, not which uncovered items it lifts), and validate it
   against the panel's counts.
3. **Run a greedy set cover** over the uncovered items with candidates inside the floor–ceiling
   band, and present the coverage-vs-pages curve: it shows the ceiling (usually well below 100%)
   and the long tail of pages needed to reach it.
4. **Gate the candidates on search demand**, under the variant rule above.
5. **Deliver two tracks:** demand-led subcategories (the candidates that pass the gate — usually
   a handful, and the fundable part) and a coverage backstop for the rest (the items-per-page
   lever, stable related-item links, a complete HTML sitemap) so no zero-demand page is built to
   cover an item.

Recipes: `references/subcategory-sizing.md` ("Size the items-per-page lever"), with sibling
sections on pricing the lever, exact facet membership and set cover, and the demand gate.

Also specify, per subcategory: URL, title and heading pattern, where it is linked from (parent
category body, faceted navigation, breadcrumb), and its intended indexation state.

Frame the upside explicitly when presenting: the new subcategories are a fresh ranking surface
addressing mid-tail demand the main category never ranked for. This is offence as well as defence,
and it is usually what makes the work fundable.

## Phase 3 — Verify page-1 coverage

Do not trust the design document. Verify against the built site, per category, before pruning
anything.

1. Re-run the pagination-free crawl from Phase 1 against the environment with the new
   subcategories live.
2. Recompute the orphan table — as a set difference against the independent inventory, under the
   same depth limit and seeding configuration as Phase 1, both restated with the result. A zero
   computed inside the crawl alone is not a pass. The pass condition is zero pagination-orphaned
   items in the categories being signed off, or an explicitly accepted, enumerated residual.
3. Watch for **spillover**: a subcategory that sat just under the page-1 ceiling at design time and
   has since grown past it. Its overflow items are orphaned again the moment pagination is pruned.
   Spillover is the single most likely cause of a regression in this workflow.
4. Confirm the new subcategory links are in the initial HTML and are crawlable links, not
   script-driven handlers — from complete responses, on several parent/child pairs, under the
   fetched-page rule in Phase 1. Recompute the coverage gap on the same-moment page-1 grids, with
   the queue guard and source classification restated.

## Phase 4 — Conditional pruning, rolled out per category

Only categories that passed Phase 3 get pagination handling. This is a per-category rollout, not a
site-wide switch.

Precondition, before any link-removal recommendation — three reads of the template, in order,
then one of the rendered page:

1. **Enumerate every paginated anchor the listing template emits**, not "the pagination control".
   One template commonly carries several: a "load more" anchor, a "load a lot more" variant that
   jumps ahead by a multiplier, and a hidden (`display:none`) no-JS block whose only output is a
   next-page link. Usually only the last is genuinely crawler-only — which is what people tend to
   believe about the first. A removal ticket naming just the element under discussion leaves the
   others live and the change silently fails.
2. **For each anchor, record whether the crawl path and the user-facing control are the same
   element.** A progressively-enhanced "load more" button is commonly an anchor to the next
   paginated URL — JS intercepts the click and appends results, while the `href` is simultaneously
   the no-JS fallback and the crawl path — so "remove the links crawlers use, keep the control users
   use" is one change, not two. A belief about why a link exists ("this is only here for crawlers")
   is a claim about the template, and templates are cheap to inspect and frequently contradict it.
3. **Where they are one element, read the click handler and record whether the JS path depends on
   the `href`.** Same element is not the same as inseparable. The common pattern —
   `preventDefault()` followed by an AJAX append — never reads the `href`, so stripping it leaves
   the control identical for every JS-enabled visitor and the whole cost falls on no-JS visitors
   and crawlers; on a JS-dependent shop that is approximately nobody plus the path being removed on
   purpose. Removal is then cheap. Look one layer further into the script for an existing seam: if
   the next-page URL resolves as `data-href || href` (or similar), the change is renaming an
   attribute and the JS needs no edit at all. Only where the handler *does* read the `href`, or no
   handler is bound and the anchor is the navigation, is the cost real — link removal is then a
   front-end change with UX consequences and must be scoped, estimated and risk-assessed as one,
   not presented as an SEO-only tweak.
4. **Load the listing in a real browser before any claim about what users see or can do.**
   Markup shows what the server sends, not what the user sees: a numbered block can be
   `display:none` while a scroll script loads the next pages and rewrites the address bar. Read
   the pagination block's computed display, scroll to the grid end, count the items, and watch
   the address bar and the network log. For each scroll or AJAX load, record the requested URL
   and whether it differs from the crawlable paginated URL (consequence: the AJAX carve-out
   below). A UX statement is never made from static HTML alone. Recipe:
   `references/pruning-evidence.md` ("Rendered-page check").

The coupling finding and the cost finding come from the same inspection. Stopping after the first
turns a good check into a bad estimate, and an overstated cost argues against a change the client
may have good reason to make — the more damaging error, because it looks like rigour.

**Evidence for pruning — what the paginated URLs do in search today.** Before choosing a lever,
check the paginated URLs' indexability (canonical, robots, hreflang) and pull, per market, the
ranking tool's keywords whose ranking URL matches the pagination parameter and Search Console
page × query for the paginated set. Paginated URLs ranking for head terms or item queries are the
cannibalisation case that justifies pruning. **A paginated URL holding head-term traffic needs a
deliberately chosen redirect target and its own monitoring; a blanket "301 to page 1" rule must
not cover it by default.** Where paginated URLs carry hreflang to sister domains' paginated URLs,
pruning ships on all of those domains together. Recipe: `references/pruning-evidence.md`
("Evidence for pruning").

- Once items are genuinely pagination-independent, a **self-referencing canonical on each paginated
  page** is usually sufficient: the pages remain crawlable, the items no longer depend on them, and
  no signal is misdirected. Treat **canonicalising deep pages to page 1** as a secondary lever, used
  where the client specifically needs consolidation and accepts that it is a hint rather than a
  directive.
- Blocking pagination in robots.txt cuts the crawler's path through those pages entirely. Reserve
  it for cases with a demonstrated crawl-budget problem, and only after coverage has been verified.
- **301-redirecting `?p=N` to the base listing URL is the most aggressive lever of all** — not a
  gentler alternative to a robots block. The ladder orders by how much discoverability is removed,
  not by how forceful the mechanism feels: a robots-blocked page is merely unfetched, whereas a
  redirected one resolves to different content, so any item appearing only on page 2+ becomes
  unreachable the instant it ships, with no grace period and no partial behaviour. The property
  that makes the redirect attractive — a directive where a canonical is a hint — is the property
  that makes it final. It arrives in conversation looking like a tidy-up after link removal, or as
  the fix for the stranding risk of a robots block; it inherits none of that option's caution by
  default and deserves more. The escalation case that justifies it: a canonical-to-page-1
  demonstrably ignored on URLs with long indexation history, after coverage is verified. Include
  the page-1 parameter variant (`?p=1`) in the redirect — a pure duplicate that is typically never
  linked. Two operational requirements travel with it:
  - **Sequencing inverts.** The redirect goes in *before* the internal links are removed, not
    after. Once every fetch resolves to the base URL, removing the links is tidiness rather than a
    precondition, and the window in which URLs are unlinked-but-not-yet-consolidated never opens.
  - **The AJAX carve-out is a requirement of the dev instruction, not a caveat.** A
    progressively-enhanced listing typically fetches its own paginated URL space for partial
    responses — `?p=N&infScroll=1` or similar — so a redirect matching `[?&]p=\d+` alone catches the
    site's own requests and the listing silently serves page 1 forever. Enumerate every parameter
    the listing's scripts append (one template has been observed using two different such
    parameters, one per load-more control) and exempt each by name in the ticket. Where the
    rendered-page check (read 4) shows a scroll load requesting the plain paginated URL with no
    marker, there is nothing to exempt: the dev instruction first adds a marker parameter to the
    script, and the redirect exempts that marker.
- Before finalising the directive, **check current search engine guidance on pagination handling
  and rel next/prev**, and verify it against the vendor's dated changelog or announcement rather
  than the documentation page alone — documentation pages are edited silently, changelogs carry
  dates. Record the date and source of what was checked in the deliverable.
- Sequence the rollout: one or two categories first, monitor indexation and organic coverage of
  their items for a full crawl-and-index cycle, then extend. Define in advance what a regression
  looks like and what rolls it back.

## Phase 5 — Governance

Hand over the rules that keep the architecture true after the project ends.

- The page-1 ceiling is an ongoing constraint. Specify a recurring check that flags subcategories
  which have grown past it, and the action when one does (split, or raise items-per-page).
- New items must land in at least one demand-backed subcategory at creation. Specify who enforces
  this and where.
- The curated facet-combination allow-list needs an owner and a review cadence. Without one it
  either ossifies or leaks.
- Re-run the Phase 1 item-level coverage diff periodically as the standing health metric: the
  uncovered count should stay at the accepted residual. On drifting listings this is the only
  measure that holds — a fixed list of items does not.

## Anti-patterns

- **Directive first.** Recommending noindex, canonical-to-page-1, a robots.txt block or a redirect
  on pagination before the orphan audit exists. This is the failure this skill exists to prevent.
- **Ranking the redirect below the robots block.** A 301 to page 1 is proposed as the tidy-up
  after link removal or as the safe fix for a robots block's stranding risk, and priced beside
  "canonical, but it works". Order levers by what becomes unreachable the moment they ship: a
  redirect removes more discoverability than any of them, and a redirect that does not exempt the
  listing's own AJAX parameters by name breaks the listing.
- **Defining the paginated set by regex.** Classifying sources from a URL pattern rather than from
  the links the templates emit. It admits parameter variants nothing links to and misses linked
  variants the pattern never anticipated, and both errors survive into every downstream number.
- **Asserting the separation between crawl path and control without reading the template — in
  either direction.** A belief about the purpose of a piece of markup is not evidence about the
  markup. Assuming the load-more link exists "only for crawlers" and is removable for free is one
  error, and it survives longest among the people who built the thing; assuming that because it is
  one element the removal must be expensive is the other, and it looks like rigour. Enumerate the
  anchors, read the handler, then price it.
- **Testing whether an item is linked by grepping for its id.** An id substring hits the JSON
  product payload, not the link; only an anchor-level match (`<a` + `href=`) answers "is this
  item linked from here". An impossible result (one item on more pages than a stable sort
  allows) is the instrument failing, not the site.
- **Shipping a per-URL orphan list without its vintage.** An undated list implicitly claims to
  describe the present. Structural conclusions survive an ageing crawl; per-URL remediation
  lists do not, and the stale rate is measurable on a sample before delivery.
- **Reading a crawl's silence as evidence.** Treating "not in the crawl data" as "not on the site"
  without stating the depth limit and seeding configuration and checking a non-traversal source.
  A depth-limited crawl of next+last pagination is blind to the middle of every long sequence.
- **Ranking by link deficiency without filtering dead targets.** The top of an unfiltered
  "worst-linked" list is mostly removed pages, so the finding is an artefact of the population
  definition rather than a fact about the site.
- **Reading an absence off an unfinished crawl or a partial response.** A listing's rounded 100%
  with URLs still queued, or one truncated fetch of a parent page, both produce a clean,
  plausible, wrong "not linked". Check the queue and the response before the diff.
- **Counting a self-link or a sister-domain switcher link as a non-paginated source.** Both apply
  to every item uniformly, so they never surface as outliers; "exactly one source" is then "zero
  plus self". Classify the weakest item's sources by hand before trusting the aggregate.
- **Joining a live fetch against a crawl snapshot on a rotating listing.** Every item that moved
  into or out of the sale listing between the two time points becomes a finding. Both sides from
  one snapshot, and the depth bound before any of it.
- **Ranking listings by size to find uncovered items.** Cross-cutting facets double-count the
  same items; measure item-level page-1 coverage directly with a validated grid extractor.
- **Pricing the items-per-page lever from the first cards, or treating "items not on page 1" as
  the scope of subcategory work.** The first cards are the above-the-fold exceptions — measure
  as a function of item count. Demand decides where a subcategory is worth having; the uncovered
  remainder gets a coverage backstop, not zero-demand pages.
- **Noindexing paginated pages that are still the only route to items.** Beyond the coverage
  problem, a noindexed page's onward links carry less weight over time — the orphaning is
  progressive and easy to miss.
- **Subcategories invented from the catalogue tree alone.** Splitting by whatever attribute the
  product data happens to carry, with no demand check, produces pages nobody searches for and a
  larger index to govern.
- **Auto-generating every facet combination.** Turning the whole facet matrix into indexable pages
  replaces a pagination problem with a combinatorial one. Curate.
- **Site-wide rollout on one category's evidence.** Coverage is per category; so is the rollout.
- **Ignoring per-market variance.** Signing off a market with a shallow catalogue and applying the
  same directive to a deep one re-orphans thousands of items.
- **Treating design sign-off as verification.** The design document is a plan. Only a re-crawl is
  evidence.
- **Dropping the ranking-surface framing.** Presented purely as index hygiene, the work reads as
  cost with no upside and stalls.

## Pre-flight verification

Before delivering, the agent runs this against its own output:

- [ ] The paginated-source class was derived from the links the templates actually emit, with the
      emitted pattern, the linked page set (full list / next-only / next+last), each paginated page's
      canonical target and the status of any page-1 parameter variant recorded.
- [ ] Where a segmentation layer drives the analysis, pagination is its own segment class.
- [ ] A pagination-free crawl was actually run (or its absence is flagged as a blocking gap), not
      assumed from sitemap data alone.
- [ ] The crawl's depth limit and sitemap-seeding setting are stated, the unreachable band of the
      sequence is reasoned about from the link pattern, and no "nothing depends on pagination"
      conclusion rests on crawl data alone.
- [ ] Both endpoints of any link ranking are constrained to live 200 URLs, and the top rows of any
      "worst-linked" ordering were inspected before the number was quoted.
- [ ] The item inventory came from at least two reconciled sources, at least one of them not
      assembled by traversal.
- [ ] The crawl's queued/waiting count was read from its live progress state before any absence
      diff; the deliverable states that the queue was empty or how many queued URLs were
      subtracted from the absent set.
- [ ] Every per-item source count excludes self-links, labels structural sources separately
      (reachability and editorial linking reported as two numbers), counts same-domain sources
      only in a multi-domain project, takes both sides of any subtraction from one snapshot, and
      applies the depth-≥3 bound; the weakest item's sources were classified by hand.
- [ ] A depth bound passed via a depth-1 HTML sitemap is reported as reachability only, with the
      sitemap named as sole non-paginated source off page 1; analytics for paginated URLs keep
      the query string (never GA4's "Landing page").
- [ ] Listing stability was tested before any pagination-only set was presented as a fixed
      remediation list, and where listings drift the remedy is structural.
- [ ] Item-level page-1 coverage was measured from same-moment page-1 grids with the extractor
      validated against a declared per-listing count, and the per-category, per-market table is
      a breakdown of that set, with counts and shares, not a size ranking of listings.
- [ ] The crawl's completion date is resolved and recorded, and the orphan table carries an
      as-of date. Any per-URL list shipped for item-by-item action was live-verified on a
      stratified sample with the observed stale rate reported, or regenerated from a fresh
      crawl; structural conclusions may rest on the older crawl and say so.
- [ ] Every proposed subcategory has a stated demand signal, read as the max over singular, plural
      and qualified variants with the carrying variant named.
- [ ] Every proposed subcategory is checked against both the item floor and the page-1 ceiling, with
      the site's items-per-page value stated.
- [ ] The items-per-page lever was sized from the earliest-page distribution of uncovered items
      before subcategory design, and priced as a function of item count (markup, DOM, render
      time), not from the first cards.
- [ ] Existing landing-page mechanisms were inventoried, each candidate's existence checked
      against the crawl's listing set, and any parent → child absence claim rests on a complete
      response verified by marker and size on several pairs.
- [ ] The design shows a coverage-vs-pages curve from exact facet membership and delivers two
      tracks: demand-led subcategories and a coverage backstop for the remainder.
- [ ] Coverage is asserted explicitly: every item is linked from page 1 of at least one
      subcategory, or the residual is enumerated and accepted.
- [ ] Any item-level reachability check matched anchors (`<a` + `href=`), not id substrings, with
      the anchors-vs-substring-count cross-check applied where the two could diverge.
- [ ] Every paginated anchor the listing template emits was enumerated (not only the control under
      discussion), and for each the deliverable records whether the crawl path and the user-facing
      control are the same element and whether the click handler reads the `href`; removal is
      priced from that read — cheap where the JS path ignores the `href` or a `data-href` seam
      already exists, a scoped front-end change with UX consequences where it does not.
- [ ] Every UX statement rests on a real-browser load, not static HTML, with each scroll/AJAX
      request URL recorded against the crawlable paginated URL.
- [ ] Indexability, ranking-tool keywords on the pagination parameter and Search Console page ×
      query were checked per market before any lever; a paginated URL holding head-term traffic
      has its own redirect target and monitoring; hreflang-linked sister domains prune together.
- [ ] Pagination handling is presented as conditional on verified coverage and staged per category,
      never as a standalone directive.
- [ ] Self-referencing canonical is presented as the default post-independence state, with
      canonical-to-page-1, robots blocking and a 301 to the base listing URL as secondary, justified
      levers in that order of aggression. Any redirect recommendation names its escalation case,
      sequences the redirect before link removal, includes `?p=1`, and exempts by name every
      parameter the listing's own scripts append — or, where a script requests the plain
      paginated URL, adds a marker parameter to it first.
- [ ] Current vendor guidance on pagination was checked, with the date and the changelog or
      announcement source recorded — not the documentation page alone.
- [ ] Spillover risk is called out with a named recurring check.
- [ ] The ranking-surface and index-governance upside is stated, not just the defensive rationale.
- [ ] Rollout order, monitoring window and rollback condition are specified.
- [ ] No claim about search engine behaviour is stated as timeless fact where it should be a check
      instruction.

## Feedback on this skill

If the user has methodology feedback, offer to draft a report for
https://github.com/rebelytics/pagination-independence-architecture; before drafting, search that
repository's open and closed issues and pull requests for the same problem,
and read its CONTRIBUTING file for how the maintainer takes reports. If the
problem is the agent not following the skill's rules, acknowledge and
correct it instead.

Pagination-Independence Architecture: Eoghan Henn and contributors | CC BY 4.0
