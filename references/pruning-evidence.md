# Phase 4 — pruning preconditions and evidence

Reference file for the `pagination-independence-architecture` skill (CC BY 4.0 — Eoghan Henn and contributors). Long-form mechanics behind two Phase 4 rules in `SKILL.md`: the rendered-page read
of the precondition, and "Evidence for pruning". Each section is a self-contained
recipe. Load the section the core points at; the file is not meant to be read end
to end.

## Contents

- Rendered-page check
- Evidence for pruning

## Rendered-page check

The first three Phase 4 reads (enumerate the anchors, crawl path vs control, does
the handler read the `href`) are reads of the markup. Markup shows what the server
sends; the user sees what the scripts make of it. A statement about user behaviour
("there is no load more", "the numbered links are the users' only navigation",
"removing the links breaks browsing") needs the rendered page, and a script's
network requests are part of the template.

Recipe, on one representative listing per template:

1. **Load page 1 in a real browser** with JavaScript enabled, at a normal viewport.
2. **Read the pagination block's computed style**, not its markup
   (`getComputedStyle(el).display`). A block that is `display:none` for every
   visitor is not the users' navigation, whatever it looks like in source.
3. **Scroll to the end of the grid** and keep scrolling until nothing more loads.
   Count the items before and after each load.
4. **Watch the address bar.** A scroll loader that pushes `?p=2` into the history
   makes paginated URLs user-facing: they get shared, bookmarked and landed on,
   which matters for any redirect.
5. **Watch the network log.** For each scroll or AJAX load, record the requested
   URL and compare it with the crawlable paginated URL.
   - It carries a marker parameter (`&ajax=1`, `&infScroll=1` or similar): any
     redirect exempts that marker by name.
   - It is identical to the crawlable URL: a redirect on the pagination pattern
     catches the site's own loads, and scrolling breaks. The dev instruction adds
     a marker parameter to the script before the redirect ships; an exemption
     alone has nothing to match.
6. **Read where the script gets its next URL.** A scroll loader commonly builds it
   from the hidden pagination link's `href`, which puts the "crawler-only" block on
   the JavaScript path (Phase 4 read 3).

The failure shape: a static fetch finds no load-more markup and a numbered block,
concludes that the numbered links are the users' only navigation, and prices link
removal as a UX break. In the browser the block is hidden, items load on scroll,
and the real constraint (the scroll loader requests the plain paginated URL, so a
redirect would break it) is never seen.

## Evidence for pruning

What justifies pruning differs per site. Establish what the paginated URLs do in
search before choosing a lever, per market.

1. **Indexability.** From the head of a sample of paginated URLs per template and
   market, record the canonical target (self or page 1), robots meta and
   `X-Robots-Tag`, and hreflang. Indexable, self-canonical paginated URLs can rank,
   and can compete with their own page 1.
2. **Ranking-tool pull.** One domain-keyword report per market, filtered to
   keywords whose ranking URL matches the pagination parameter as emitted (a URL
   regex such as `[?&]p=[0-9]`). Paginated URLs ranking for a category head term
   or for item queries are the cannibalisation case. For the head terms, also read
   the ranking-URL history: a ranking URL alternating between the homepage, page 1
   and page 2 is the same signal over time.
3. **Search Console page × query for the paginated set.** Filter the performance
   report on page containing the pagination parameter, break it down by page over
   a long window (twelve months or more), then by query for the top pages. Clicks
   commonly concentrate: one paginated URL can take most of the paginated clicks,
   for generic head terms that are not even its category's own term, and can
   out-click its own page 1 in recent weeks. This is stronger evidence than
   ranking positions, because it is clicks.
4. **Analytics.** Use a dimension that keeps the query string (GA4 "Landing page
   + query string"). GA4's "Landing page" dimension folds `?p=N` into the base URL,
   so its zero for paginated landings is an instrument artefact.

Consequences for the directive:

- **A paginated URL holding head-term traffic gets a deliberately chosen redirect
  target** (the page that should own those queries, which need not be its page 1)
  and its own monitoring line in the rollout plan. A blanket "301 to page 1" rule
  must not cover it by default.
- **Where paginated URLs carry hreflang to sister domains' paginated URLs, prune
  on all of those domains together.** Pruning one side leaves the others'
  hreflang pointing at redirects or removed pages.

The failure shape: a zero from an analytics dimension that strips the query string,
plus the assumption that nobody lands on page 2, produces a blanket redirect that
sends a market's head-term traffic from a well-ranking paginated URL to a page 1
that never ranked for those terms.
