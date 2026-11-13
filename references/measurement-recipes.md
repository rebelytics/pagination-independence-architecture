# Phase 1 — measurement recipes

Reference file for the `pagination-independence-architecture` skill (CC BY 4.0 — Eoghan Henn and contributors). Long-form mechanics behind the Phase 1 rules in `SKILL.md`. Each section is a
self-contained recipe: what to run, what the result looks like when the method is
broken, and how the failure was observed. Load the section the core points at;
the file is not meant to be read end to end.

## Contents

- Queue guard before any absence diff
- Same-snapshot consistency and the depth bound
- Classifying inlink sources
- Listing stability test
- Item-level page-1 coverage from validated grid extraction
- Reading an absence off a fetched page

## Queue guard before any absence diff

A crawler's URL exports contain *fetched* URLs; its queue contains
*discovered-but-unfetched* ones. Every absence-based conclusion — orphan
detection, "not linked", "not reachable", a sitemap diff — treats queued URLs as
missing, when discovery is precisely what it is asking about. And the queue is not
a random sample: it is the tail of discovery, so it concentrates exactly the
late-found, lightly-linked URLs an orphan analysis is hunting for.

Recipe:

1. Read the crawl's **live progress state** — the count of waiting or queued URLs
   and whether the crawl is paused — not a crawl listing's completion percentage.
   A rounded percentage conceals a non-empty remainder: a few hundred queued URLs
   against a six-figure fetched count round to 100.0%.
2. If the queue is non-zero, export the queued URLs (a small export, safe even
   when the bulk exports time out) and **subtract them from the "absent" set**.
   Better: resume and let the crawl finish, then re-run the diff.
3. State in the deliverable either that the crawl finished with an empty queue, or
   how many queued URLs were removed from the absent set.

The failure shape: on a large crawl, a sitemap-vs-crawl diff reports a handful of
products reached by no link. Every one of them sits in the crawl's queue — discovered
via a link and simply not yet fetched. They are the opposite of orphans and the true
count is zero; thinly linked items (promotions linked from one or two places) are
exactly the ones a crawler queues last.

An absence claim over an unfinished process is a claim about the process, not the
world.

## Same-snapshot consistency and the depth bound

When the full inlink graph is too large to export, a tempting shortcut is to compute
a per-item count two ways and subtract: the crawl's stored total inlink count minus
the number of paginated pages linking the item, the latter obtained by live-fetching
the paginated pages. Equal counts would mean "linked only from pagination".

Do not do this on any page set containing rotating listings — sale, new-in,
bestseller, featured, "popular". A difference between two measurements is only a
finding if both were taken of the same state. Churn *into* a rotating listing
between the crawl and the fetch inflates the live count and manufactures false
"only paginated" positives; churn *out* deflates it and hides a true positive. The
false positives are visible; the false negatives are silent, so a clean run does not
establish absence.

Rules:

- **Both sides from one snapshot**, or exclude the rotating listings from both
  sides — and say which was done. Where the inlink export exceeds what the tool's
  API can return, route it through the crawler's desktop export rather than
  approximating it with a live join.
- **A one-sided sanity inequality catches only the gross tail.** "Live count
  greater than crawl total" is impossible if both measure the same thing, and it
  fires on the extreme cases; inflation that lands *exactly* on the total, or stays
  below it, passes. A control that certifies everything moderate is where the
  plausible wrong answers live.
- **Apply the crawl-depth bound as a free pre-filter.** A pagination-only item sits
  at crawl depth ≥ 3 (home → page 1 → page N → item), because paginated pages hang
  off page 1 and cannot be at depth 1. Any item at depth ≤ 2 provably has a
  non-paginated path and is excluded without any inlink data. A method that flags
  a depth-≤2 item as pagination-only is broken; reject the method, not the item.
  This is a consistency check against a *different field of the same snapshot*, and
  it is stronger than any cross-time control.

The failure shape: a live-vs-snapshot subtraction flags a few products, and every flag
is false — each item has several genuine non-paginated sources, most of them in a
rotating section (a sale, a promotion) that changed between the crawl and the live
fetch. The inequality control catches some; others are inflated to exactly the total
and pass it. All of them sit at a shallow crawl depth, so the depth bound would have
rejected them at no cost.

## Classifying inlink sources

A count of "non-paginated sources" is only as good as the definition of a source,
and the definition's failure cases are the ones that apply to every item uniformly:
they shift the whole distribution by one and never look like outliers. Before
counting an item's non-paginated inlinks, classify each source:

| Class | Examples | Counts toward pagination-independence? |
|---|---|---|
| **Self** | breadcrumb last node, share links, language switcher linking to the same URL | No — remove first. Crawler `Unique Inlinks` figures commonly include the page itself. |
| **Paginated** | page 2+ of any listing, per the Step 0 class | No — this is the dependency being measured. |
| **Cross-domain** | language/country switcher or alternate links from sister domains in a multi-domain crawl project | No — to a search engine these are links between separate sites. They aid discovery but are not internal linking of the target domain, and they are not the safety net that makes pruning safe. A single-domain crawl cannot see them at all, which is one reason two tools disagree. |
| **Structural, near-universal** | HTML sitemap page, footer "all products", sitewide menus, "all articles" index | Reachability, yes; editorial linking, no. Label separately. |
| **Editorial** | category body links, related items, blog posts, page-1 grid of a listing | Yes — this is the count that matters. |

Report **reachability** (any non-self, same-domain, non-paginated source exists)
and **editorial linking** (an editorial source exists) as two numbers. Structural
sources that reach nearly every item make every item look linked; an item whose
only non-paginated source is the HTML sitemap is reachable but not linked in any
sense that survives a sitemap template change.

**The cheap test that makes this safe.** Before trusting any aggregate — and
especially before interpreting a threshold near zero ("exactly one non-paginated
source") — pull the actual inlink sources of the single weakest item the method
produced and classify each by hand. One read of the right row exposes a phantom
source in every row.

**Write down what an anomaly would imply if it were systematic.** When an early
verification turns up something odd but apparently harmless — a page listing
itself among its own sources — record what the analysis would look like if that
applied to every item, before deciding it is minor. An anomaly noticed and
rationalised is costlier than one never seen, because the noticing produces a
feeling of diligence that stops the follow-up.

The failure shape: the "exactly one non-paginated source" bucket holds a sizeable
slice of the catalogue and is reported as fragile but safe, supporting a redirect of
paginated URLs — and the one source is the page itself. A stratified sample finds
most of the bucket with no real non-paginated source, so the directive the finding
endorses would orphan a measurable share of the catalogue. A self-reference noticed
on one item and dismissed as minor is the early sign; a suspiciously clean result on
another content type (zero pagination-only, zero fragile) is the prompt to check, and
the weakest item's real sources turn out to be the HTML sitemap, the page itself
many times over, and a few paginated listings.

## Listing stability test

"Pagination-only" is not a fixed property of an item; it is a snapshot of which items
happened to sit on page 2+ when the crawl passed. Where the default sort is
merchandiser position, stock-based visibility, recency or popularity, the set drifts
between crawls and even within one.

Before treating a pagination-only set as a remediation list, test stability:

1. **Within a crawl:** is any item recorded on more than one page of one listing?
   Once the anchor-level method has been confirmed sound (a substring test can also
   produce this), that is evidence the order moved while the crawl ran.
2. **Across crawls:** does the same item sit on different pages of the same listing
   in two crawls, or on page 1 in one tool and page 3 in another?
3. **Live spot-check:** fetch pages 1–N of the listing for a few flagged items and
   record where each sits now.

Where listings drift, linking the items on one snapshot's list is not a remedy — a
different set drifts onto page 2+ next week. The durable remedy is structural: every
item on page 1 of at least one listing (the page-1 ceiling, Phase 2), stable
related-item links, or at minimum a complete HTML sitemap as a reachability backstop.
The list is a sample of a process; the remedy has to target the process.

The failure shape: one item is on page 1 of its category in one tool's crawl, on
later pages in another's, and on none of the first pages in a live check the same
morning — and both crawls record it on two paginated pages at once, which a stable
sort cannot produce.

## Item-level page-1 coverage from validated grid extraction

The pass condition of the whole workflow is "every item is linked from page 1 of at
least one listing". Measure it directly, at item level, rather than through a table
of listings ranked by size.

Why the size table misleads: the largest "leaf" listings in a big catalogue are
commonly cross-cutting facets over the same items (attribute listings, "popular",
"new"), so listing slots across N listings represent far fewer distinct items, many
of them already on page 1 of a smaller listing. A size table double-counts in
exactly the catalogues where the problem is biggest, and a URL-based leaf/parent test
does not help, because cross-cutting facets are leaves by URL.

Recipe:

1. **Enumerate every listing URL** (from the crawl's listing segment, including facet
   landing pages that are real pages — see the slug trap in
   `references/subcategory-sizing.md`).
2. **Fetch page 1 of every listing at one moment.** A run that spans hours on a
   rotating catalogue measures several states at once.
3. **Extract only the product grid** — anchors inside the grid's card container —
   and exclude menu, footer, "recently viewed" and cross-sell blocks. This is the
   anchor-level rule from the core applied at scale.
4. **Validate the extractor against a count the page declares about itself.** Most
   listing templates expose their exact size somewhere — an inline JS variable, a
   "showing 1–36 of 412" element, a JSON payload field. For every listing, extracted
   grid count must equal min(declared size, items-per-page). A mismatch on any
   listing means the extractor is reading something other than the grid. The same
   check surfaces zero-item hub pages that look like top categories but provide no
   coverage.
5. **Union the grids and diff against the independent inventory** from audit step 2.
   The residual — items on no listing's page 1 — is the coverage gap, measured
   directly.
6. **Record, per uncovered item, the earliest page it appears on** in any listing.
   This distribution is the input to sizing the items-per-page lever in Phase 2.

What a sound run looks like: the grid-extraction method's share of items on no
page 1 agrees with an independent earlier figure, the extractor validates exactly on
every listing, and zero-product hub pages come out as their own class. A size-banded
table produced first counts listing slots, not items — cross-cutting listings put
the same item in many slots, so its total exceeds the catalogue and answers a
different question.

## Reading an absence off a fetched page

An absence read off a single response is only as good as the response. Before
concluding that a page does not link to something — a parent category not linking
its filter landing page, a listing not linking an item — confirm the fetched page is
a complete, normal response:

- **A marker that must be present** is present: the filter panel container, the
  product grid, the footer, a closing `</html>`.
- **The size is in line with sibling fetches** of the same template. A response at
  roughly half the normal size, with a whole panel missing, is a partial or variant
  response (bot handling, an A/B variant, a truncated transfer), not evidence.
- **Repeat across several fetches and, where relevant, user agents**, and test the
  link class on **several parent/child pairs** before generalising from one.
- **Cross-check against the crawl** — depth and actual inlink sources, not the inlink
  count alone. A child at crawl depth 2 under a depth-1 parent is consistent with the
  link existing; a low inlink count says how many pages link it, not whether the
  parent does.

The failure shape: one fetch of a parent category comes back at a fraction of the
template's normal size with no filter panel at all; a grep for the child URL finds
nothing, and "the filter pages are not linked from their parents" goes into a draft
ticket. Fresh fetches across several user agents all carry the link, as do the other
parent/child pairs, and the crawl showed the child at a shallow depth all along.
