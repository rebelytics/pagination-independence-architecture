# Phase 2 — sizing recipes

Reference file for the `pagination-independence-architecture` skill (CC BY 4.0 — Eoghan Henn and contributors). Long-form mechanics behind the Phase 2 rules in `SKILL.md`: how to size the cheap
lever before designing the expensive one, how to price it, and how to turn an
uncovered-item set into a short list of demand-backed pages plus a backstop. Load
the section the core points at.

## Contents

- Inventory existing landing-page mechanisms
- Size the items-per-page lever
- Price the items-per-page lever
- Exact facet membership and greedy set cover
- Demand gate and the two tracks

## Inventory existing landing-page mechanisms

Before specifying a new mechanism for subcategories, find the one the platform
already runs. Many shops already serve clean-URL filter landing pages
(`/category/attribute-value`) from layered navigation, a "shop by" block or a
merchandising tool — an existing vehicle that a new subcategory can ride without
template work. Record, per mechanism: URL pattern, whether the page is indexable,
whether the parent listing links it in the initial HTML, and whether it is in the
crawl's listing set.

**A clean slug in markup is not a landing page.** A filter value carrying a
URL-friendly slug in a data attribute or a "SEO-enabled" flag can still resolve to a
404. Check existence against the crawl's listing set (or fetch it), not against the
markup.

**Verifying that a parent links its filter or subcategory page** is an absence
claim and is subject to the fetched-page controls in
`references/measurement-recipes.md` — a complete response, several pairs,
cross-checked against crawl depth and sources.

## Size the items-per-page lever

The core names raising items-per-page as a cheap lever alongside splitting. Size it
before designing any subcategory, so that design targets the residue the cheap lever
leaves.

Input: for every item on no listing's page 1, the earliest page it appears on in any
listing (recorded during the item-level coverage measurement). Then, for each
candidate page size P — the platform's selectable values, or a few plausible ones —
compute the share of uncovered items whose earliest position is ≤ P:

| Items per page | Uncovered items brought onto a page 1 | Residual |
|---|---|---|
| current (e.g. 36) | 0 | all |
| 48 | … | … |
| 72 | … | … |
| 96 | … | … |

Two things to read off the table:

- The share the lever covers at each size, so the recommendation is a number, not a
  hunch.
- **Where the residual concentrates.** It is usually a handful of large listings, and
  those — not "every category" — are what subcategory work has to target.

What sizing typically shows: raising the default items-per-page covers most of the
gap on its own, leaving a residual concentrated in a few listings — which turns a
catalogue-wide redesign into a targeted one an order of magnitude smaller.

## Price the items-per-page lever

The cost of a larger page is **per-item markup and DOM** (KB of HTML per card,
gzipped and raw, and DOM node count at the target size) and **server render time**,
which needs a staging test at the target size, not an estimate. Below-the-fold card
images are commonly lazy-loaded already, so extra cards add little to the image
request count at load.

Measure as a function of item count: fetch the same listing at several sizes (3, 10,
20, 36, the target) and fit eager requests, bytes and DOM nodes against card count.
The regression separates the fixed template cost (the intercept) from the per-item
cost (the slope), which is what a page-size decision needs. Never generalise from the
first cards: behaviour that depends on position — above vs below the fold, featured
vs rest — makes the first items the least representative sample available, because
the first items are where the special treatment lives.

The failure shape: the first cards on a listing show plain `src=` images with no
lazy marker, which gets reported as "card images are not lazy-loaded" and makes a
larger page size sound like hundreds of extra eager image requests. A regression of
eager image requests over listings of different card counts shows the count barely
moving — the template plus the first few cards — with every card beyond them already
lazy, so the real marginal cost is a few KB of markup per card. The wrong sample also
produces the wrong recommendation — "lazy-load the cards first" — for something
already done.

## Exact facet membership and greedy set cover

Facet *counts* from the filter panel say how big a facet value is, not which of the
uncovered items it would lift onto a page 1. To size subcategory design you need
membership.

1. **Exact membership from the platform's own filter endpoint**, where it is
   exposed. Layered-navigation implementations commonly answer an AJAX request with
   the full item list for a facet value within a category (raise the page-size
   parameter to return everything). Validate the returned lists against the panel's
   displayed counts — exact on nearly every value, off by a handful on the rest, is
   normal; systematic disagreement means the endpoint applies a filter the panel does
   not.
2. **Candidate set:** every (category, facet value) whose membership size sits
   between the item floor and the page-1 ceiling.
3. **Greedy set cover** over the uncovered items: repeatedly pick the candidate that
   covers the most still-uncovered items, until no candidate adds anything. Plot the
   result as a **coverage-vs-pages curve**: uncovered items covered after 10, 25, 50,
   100 pages, and at exhaustion.

The curve answers "what would subcategories for every uncovered item look like": a
ceiling, usually well below 100%, reached only with a long tail of pages. Items with
no facet value in the floor–ceiling band cannot be covered by facet subcategories at
all and need the backstop.

The typical curve: the first few pages cover a modest share of the uncovered set,
coverage grows with diminishing returns, and even the full candidate list leaves a
substantial remainder for the backstop.

## Demand gate and the two tracks

Coverage decides what a subcategory *could* fix; demand decides where one is *worth
having*. These are different sets, and the framing error to guard against is
treating "items not on page 1" as the scope of subcategory work. Most high-gain set
cover picks in a large catalogue are brand × attribute combinations that nobody
searches for; building them adds the index bloat the work exists to remove.

**Demand gate.** For each candidate, check search demand in the target market
(keyword tool, Search Console, site-search logs). Before calling a term demand-less,
query at least the singular, the plural and the common qualified form, and take the
max as the concept's demand: keyword tools commonly put a cluster's volume on one lead
variant and leave the siblings near the floor, and which form leads is not
predictable. A floor reading on one string means "this string is not the lead", not
"nobody searches this". Report which variant carried the volume. (Tool-specific
detail lives in the keyword tool's own companion skill.)

**Two tracks in the deliverable:**

1. **Demand-led subcategories** — the candidates that pass the gate, each with its
   demand figure, its membership size, and the uncovered items it lifts. Usually a
   handful of pages covering a modest share of the gap, and the fundable part of the
   work.
2. **Coverage backstop for the rest** — the items-per-page lever, stable related-item
   links, or a complete HTML sitemap, so that every remaining uncovered item has a
   non-paginated path without a zero-demand page being built for it.

The typical result: only a small fraction of the set cover candidates is
demand-backed, they cover a minority of the uncovered items, and the rest is left to
the backstop.
