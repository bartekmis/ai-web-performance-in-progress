# Stage 30: Preconnect - warming connections to external domains

> Can be run STANDALONE or from the index (.ai/web-performance/audit/00-index.md).
> Behaviour (IMPORTANT): question-driven audit. Every "Ask for" = ask me and wait.
> "Do" items (curl, Chrome DevTools MCP snippets, JSON parsing) you perform yourself,
> then write to site-profile.md > ## Network (### Preconnect) and findings.md.
> Runs AFTER Stage 20. Reuses ## Page types and ## Network from site-profile.md.
> Input: ## Page types (representative URLs) from site-profile.md.
> Output: a capped (MAX 4) list of preconnect tags to add + hints to remove/downgrade.
> Idempotent: if ### Preconnect exists, update it, do not create a second one.

WHY THIS STAGE (say once, then run): every EXTERNAL origin the page pulls from (fonts,
LCP image host, CDN, GraphQL/API, analytics) costs a fresh DNS + TCP + TLS handshake the
first time. `preconnect` opens that connection early so the critical request does not sit
waiting for the handshake. But it is not free: each preconnect holds an open connection,
and past ~4 they compete and hurt. So the job is to preconnect the FEW earliest-critical
origins and leave the rest alone (dns-prefetch at most, or - better - stop loading them in
the critical path).

START: read ## Page types from site-profile.md. If missing, run Stage 10 first (or ask me
for at least one representative URL) and wait.

SCOPE & SPEED (IMPORTANT): run this on ONE representative page - the most important /
heaviest template (usually the homepage). Do NOT loop over every page type; a full page
load per type multiplies the wait for little gain. Only spot-check a SECOND page if you
have concrete reason to think it pulls DIFFERENT third-party origins (e.g. an article
template with an embed the homepage lacks) - then compare and note the delta.

## 30.1 Collect the cross-origins the page actually uses (Do)
Prefer REAL runtime data over parsing HTML - a page connects to origins that never appear
as static `<link>`s (injected by scripts, e.g. a tag manager fanning out to 20 vendors).

Primary (runtime, webperf snippet skill): load the ONE representative URL in the browser
and use the `webperf-loading` skill via Chrome DevTools MCP:
- `mcp__chrome-devtools__new_page` / `navigate_page` to the URL (wait for load).
- Run `Resource-Hints.js` and `Resource-Hints-Validation.js` with
  `mcp__chrome-devtools__evaluate_script`. They report: existing hints, hints that are
  UNUSED (dead), and cross-origins the page connects to that WOULD benefit from a hint.
- If you need the raw ranking, also read `performance.getEntriesByType('resource')`:
  per external origin capture request count, earliest `startTime`, bytes, initiatorType,
  and `renderBlockingStatus`. Sort by earliest startTime.

Fallback (static, no browser): `curl -sL --max-time 8 {URL}` and extract external hosts
referenced in `<head>` / early markup (fonts, CSS/JS hosts, LCP `<img>`/preload src, API
endpoints). Note in findings that script-injected origins are invisible to this method.

- Save to: findings.md (cross-origin table for the representative page).
- Done when: the representative page has a list of external origins with earliest-start + criticality.

## 30.2 Read the hints already present (Do)
List existing `rel="preconnect"` and `rel="dns-prefetch"` and their `crossorigin`.
Cross-check against 30.1:
- A preconnect/dns-prefetch whose origin serves NO request = dead hint -> remove it.
- A font preconnect WITHOUT `crossorigin` = broken (browser opens a 2nd connection) -> fix.

- Save to: findings.md (existing hints + dead-hint flags).
- Done when: current hint state recorded.

## 30.3 Rank and CAP at 4 (Do - the core rule)
Rank external origins by: (1) how EARLY the first request fires, (2) how CRITICAL it is
(fonts and the LCP image/media host first; render-blocking CSS/JS host next; API needed
above the fold next; analytics/marketing LAST).

HARD LIMIT: recommend AT MOST 4 preconnects (Lighthouse guidance). Then:
- Keep the top <=4 earliest-critical origins as `preconnect`.
- Everything else that still fires early -> `dns-prefetch` (cheap, DNS only, no cap worry).
- State EXPLICITLY which origins you dropped from preconnect and why.
- crossorigin rule: fonts and any CORS/credentialed fetch (e.g. GraphQL) need
  `crossorigin` on the preconnect. A plain image/script host does not - do not add it
  blindly (a mismatched crossorigin opens a second connection).

Bigger-picture check: if the page has MANY (e.g. >8) external marketing/analytics origins,
the real win is not preconnecting them - it is NOT loading them in the critical path
(defer / load on interaction / remove). Say this in findings; preconnect only masks the
handshake, it does not remove the third-party cost.

- Save to: findings.md (ranked table + the drop list).
- Done when: a final set of <=4 preconnect origins is chosen with reasons.

## 30.4 The change list (Do - output)
Produce the exact tags to add and to remove, e.g.:
```html
<!-- add to <head>, earliest-critical first, MAX 4 preconnects -->
<link rel="preconnect" href="https://IMAGE-CDN-HOST">
<link rel="preconnect" href="https://FONT-HOST" crossorigin>
<!-- everything else that fires early: DNS only -->
<link rel="dns-prefetch" href="https://ANALYTICS-HOST">
<!-- remove: dead/unused hints -->
```
For WordPress note the filter `wp_resource_hints`; for Next.js the `<Head>` / metadata;
for Astro a `<link>` in the layout head.

- Save to: site-profile.md > ## Network (### Preconnect) + findings.md.
- Done when: I have copy-paste tags (<=4 preconnects) + a remove list + the "defer these
  third-parties instead" note.

## At the end
Check off in site-profile.md > ## Progress: [x] Stage 30. Tell me the single most
valuable preconnect first (usually the LCP image/font host).
