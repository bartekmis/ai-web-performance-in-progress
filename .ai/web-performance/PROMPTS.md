# Audit prompts

Four cards: three ways to run + one to extend. Copy-paste into your AI assistant
(Claude Code / Codex / Antigravity...) started in the project folder (where
.ai/web-performance/ lives).

## 1. Full audit from the start
```
Read .ai/web-performance/audit/00-index.md and run the audit for my project.
Follow the "Execution rules" from the index. Start from the first stage.
IMPORTANT: ask me for every piece of data (URL, page types, URLs) and wait for my
answers. Do NOT derive them yourself from code, MCP or the network - you must ask me.
```

## 2. Resume the audit
```
Read .ai/web-performance/audit/00-index.md and resume the audit.
Run the first unchecked stage per the "## Progress" section in
.ai/web-performance/site-profile.md.
```

## 3. Single stage (standalone)
```
Run .ai/web-performance/audit/10-profile-and-baseline.md for my project.
Take data from .ai/web-performance/site-profile.md; ask me for anything missing, one
at a time, and wait. Do NOT derive data yourself from code/MCP - you must ask me.
```
(swap the filename for any stage in the audit/ folder)

## 3b. Stage 60 standalone: render start, end to end
Paste after workshop 07. Runs the whole loop: facts -> hypotheses -> experiment ->
verdict -> fix. Needs the chrome-devtools MCP server and a WebPageTest JSON in
.ai/web-performance/wpt/mobile/.
```
Run .ai/web-performance/audit/60-render-start.md for my project.

Inputs: the WebPageTest JSON in .ai/web-performance/wpt/mobile/, my page URL and stack
from site-profile.md, the chrome-devtools MCP server, and my source code.
Read what earlier stages found first; if Stage 50 says TTFB is the wall, tell me and stop.

Work in stages and show me each one before moving on. Do not propose a single fix
before the experiment is done.

1. FACTS. Write a throwaway script to pull from the JSON (do not read 1 MB into
   context): metrics, run-to-run consistency, every request that started before Start
   Render, both render-blocking lists, every consoleLog entry with its timestamp in ms,
   network gaps over 500 ms with what was in flight, TBT per origin, fonts, LCP
   candidates, zero-byte responses. Facts only.

2. WHAT IS MISSING. Cross the console errors and the render-blocking list against the
   request list. Anything render-blocking or errored with no normal row in the waterfall
   is a prime suspect: a resource that never arrives cannot be drawn as a bar. Note that
   its cost is set by the resolver and the timeout, not by my page, so it can be seconds
   on one network and milliseconds on another.

3. INTERROGATE EVERY RESOURCE that started before Start Render. One row each, five
   questions: what is it and WHO added it (parser, or which script injected it); is it
   needed on THIS page type; is it needed BEFORE the first paint; is it loaded the right
   way (sync / defer / async / preload / render-blocking); how many bytes and how much
   main-thread CPU does it cost. Then give each row ONE verdict, preferring the earlier
   option: DELETE (nothing uses it) > DEFER or ASYNC > MOVE out of <head> > ON DEMAND
   (on interaction, on scroll into view, on idle - anything below the fold) > SELF-HOST
   > KEEP (say why it must load before the first paint).
   Be precise about the ladder: a sync <script> in <head> blocks the parser and therefore
   the paint; the same script at the end of <body> stops blocking the paint above it but
   still blocks DOMContentLoaded; defer stops the blocking wherever the tag sits and keeps
   execution order, so "add defer" usually beats "move it to the body".
   If the waterfall shows a STEP - a batch of requests starting noticeably later than the
   ones above - say what caused it. A step is a discovery boundary: everything above was
   found by the preload scanner in <head>, everything below only after the parser got past
   a blocker. Name the blocker: a parser-blocking script, a render-blocking stylesheet, a
   resource that never answered, or simply the end of <head>.
   Give me this table before any hypothesis.

4. HYPOTHESES. Take the three rows from the table that promise the largest gain. For
   each: the mechanism, the expected gain and where that number comes from, and THE
   SINGLE HTML EDIT THAT WOULD PROVE IT WRONG. Everything else from the table goes to the
   fix list as a low-risk cleanup that does not need its own experiment.
   Show me these three and wait for my OK.

5. BUILD VARIANTS - in the shell, not in the browser. Download the page HTML, inject
   <base href="MY_URL"> so subresources still come from production, and produce one file
   per hypothesis differing by exactly ONE edit, plus an unmodified baseline. Serve them
   locally. Before measuring anything, confirm every variant really differs from the
   baseline and that each edit matched something - a pattern that matched nothing yields
   a file identical to the baseline, and identical files always report "no difference",
   which is not the same as "no effect".

6. MEASURE - everything through the chrome-devtools MCP server.
   Call emulate ONCE (mobile 412x765x2.6, Fast 4G, CPU 4x) and never change it again.
   Every measurement is TWO navigations:
     navigate_page type=url     -> the variant URL
     navigate_page type=reload with ignoreCache=true   -> this is the measured load
   ignoreCache is honoured only on reload; on a url navigation it is silently ignored
   and you get a page served ~60% from cache and an FCP about half the real one.
   Warm up once and discard it, then rotate A,B,C,A,B,C for at least 3 rounds.
   After each load, evaluate_script for FCP, LCP and domContentLoaded, and in the same
   script count resources with transferSize 0 and a non-zero decoded size. Above ~20%
   the load was not cold: discard it and repeat.

7. VERDICT. Compare medians, never single runs, and state the spread within each
   variant. The bar per metric is the larger of the two spreads. Allowed verdicts:
   SUPPORTED, REGRESSION, INCONCLUSIVE. A DevTools or Lighthouse estimate is a
   prediction - never present it as a measurement. Give me one table:
   hypothesis | mechanism | verdict | delta FCP/LCP/DCL | caveat.

8. FIX. Only for SUPPORTED hypotheses, at the source, one change at a time, with the
   measured delta in a comment so nobody re-adds it later. A resource that is broken but
   measured INCONCLUSIVE still gets removed - label it a correctness fix, not a
   performance fix. Run my project's type check or build.

9. REPORT. What you proved with numbers, what you failed to prove and why, what you
   changed file by file. Last line, always: re-run WebPageTest with the same profile,
   because this loop measures with warm DNS and warm connections and only WPT starts
   cold on every run, the way a real first-time visitor does.
```

## 3c. Stage 70 standalone: DOM size and recalculation cost
Paste after workshop 09 (30.07). Needs the chrome-devtools MCP server. No WebPageTest JSON
required - the browser supplies both the cost and the culprit.
```
Run .ai/web-performance/audit/70-dom-size.md for MY project.

Inputs: MY page URL and stack from site-profile.md, what Stage 60 found in ## Render start,
the chrome-devtools MCP server, and MY SOURCE CODE - the repo you are running in.
Show me each step before moving on.

Measure MY page, the URL from site-profile.md. Do not substitute any example or demo site.
If the URL, page types or stack are missing from the profile, ask me and wait.
Any figure quoted inside the stage file was measured on an unrelated reference page and is
there to calibrate the method - never copy it into my findings and never treat it as a
target. My numbers will be different.
The deliverable is not "your DOM is deep" - it is: this chain / duplicate / list in MY
codebase, in THIS file, costs THIS much, here is the change.

READ THIS FIRST, it decides whether the run is any good: node count is NOT the criterion.
The old Lighthouse rule (warn ~800 nodes, error ~1400) is gone. Chrome now flags a LARGE
LAYOUT OR STYLE RECALCULATION, roughly 40 ms and up. A page with 5000 nodes and no heavy
recalculation is fine. Never tell me "you have N elements, that is too many".

1. MEASURE. Call emulate ONCE (mobile 412x765x2.6, Fast 4G, CPU 4x) and never change it -
   which elements count as hidden or below the fold depends on the viewport. Navigate,
   then performance_start_trace with reload and autoStop. If the trace lists a DOMSize
   insight, call performance_analyze_insight on it and give me verbatim: total elements,
   DOM depth and the element that STARTS the deepest chain, most children, and every
   "Layout update: Duration N ms, with X of Y nodes needing layout".
   Run the trace 3 times. The structural statistics will be identical every run; the
   duration will not. Anything under ~5 ms, or under the observed spread, is noise.
   If there is no DOMSize insight, say so - that is a result, not a failure - do step 3
   anyway and stop before the experiment.

2. ATTRIBUTE. If the trace lists a ForcedReflow insight, analyze it too. It names the
   function and the script URL:line that forced synchronous layout, with per-frame ms.
   Tell me whether that is my code or a third-party tag, because the fix differs. Explain
   in one sentence the mechanism: JS asked for geometry after styles were invalidated, so
   the browser had to lay out on the spot - and the deeper and wider the tree, the more
   that costs. If ForcedReflow is absent, say the recalculation came from ordinary
   rendering and the fix is structural only.

3. INTERROGATE THE STRUCTURE with evaluate_script on the same viewport (the exact script is
   in step 70.3 of the stage file - scope it to document.body, skip SVG internals). Then
   give me a table of what it found, and for each row the move:
   - a large display:none subtree with a -mobile / -desktop twin = the navigation is built
     twice and CSS hides one copy -> one structure, styled by CSS;
   - a big share of elements inside a cookie modal or off-canvas panel nobody opened
     -> render on interaction;
   - a long single-purpose chain at the deepest path (it should match the element the
     DOMSize insight named - if it does not, trust the insight) -> flatten the wrappers;
   - many removable wrappers and ~40%+ div/span -> fragments and semantic tags;
   - half or more of the tree below the fold -> islands / lazy sections / content-visibility;
   - a uniform list or carousel with a big subtree -> pagination or virtualisation.

3b. MAP EVERY FINDING TO MY SOURCE - without this it is an analysis of a web page, not of
   my project. Take the class and id tokens from step 3 and search MY repo for them. Strip
   the build hash first: CSS Modules and styled-components append suffixes, so a rendered
   `navigation_dropdown__label__2tt1k` lives in the repo as `navigation_dropdown__label` or
   as a class inside a *.module.css. Give me one row per finding: finding | token searched |
   file:line in my repo | owner. Owner is one of: my component/template (I can edit it - the
   fix list goes here); my theme (say whether the change survives an update); a dependency I
   configure, e.g. a page builder or consent plugin (the markup is generated, so the fix is
   a SETTING, not a source edit - name the setting); a third-party script (the markup is not
   in my repo at all - check it is also absent from the server HTML with curl - so the only
   levers are loading strategy, configuration or dropping the vendor).
   On WordPress search the active theme AND plugins AND the page builder's saved content
   (builder markup lives in post meta, not in files) before concluding it is not mine.
   If a token is nowhere to be found, say "source not identified" and stop guessing.
   Do not write a fix for any finding whose owner you have not established.

4. PRIORITISE - and this is the step I care about most. Put the measured cost next to LCP,
   FCP and Start Render from Stage 60 and tell me honestly: FIX NOW, FIX LATER, or NO
   ACTION. A 50 ms layout on a page with LCP 4.5 s is a rounding error and I would rather
   hear that than get a fix list. Add: this measured the LOAD, so it is a lower bound - the
   same tree is laid out again on every interaction and that lands in INP.
   Show me the verdict and wait.

5. PROVE IT (only if we agreed on FIX NOW). Build local variants as HTML FILES - do not
   try to mutate the page at runtime. navigate_page's initScript does NOT survive the
   reload that performance_start_trace performs, so a runtime removal silently never
   happens and you would "prove" my fix does nothing. Download the HTML, inject
   <base href>, one edit per variant plus an unmodified baseline, verify each edit actually
   matched something, serve locally, then re-trace each variant 3 times rotated A,B,A,B.
   Compare three numbers and tell me which cost each one measures:
   - Total elements (parse, style, memory) - moves when you delete anything;
   - "X of Y nodes needing layout" - moves ONLY for elements that are actually laid out;
   - the duration in ms - noisy, judge it against the spread.
   Know this before concluding: deleting a display:none subtree drops Total elements and
   leaves layout untouched - measured, 69 elements removed, 408 -> 339, layout 322 of 322
   both times, 51 vs 52 ms. Hidden elements are parsed and styled but not laid out. That
   is a real win in parse/style/memory, just not a layout win - say which one it is.
   Also: compare variant against the LOCAL baseline only. The same page served locally
   reported 408 elements against 624 live, because third-party scripts behave differently
   off-domain. Verdicts: SUPPORTED, REGRESSION, INCONCLUSIVE, per metric.

6. FIX in MY code, using the owner column from step 3b - routed by my stack, one change at
   a time, at the source file you named, then re-measure with the same emulation and report
   before/after as median + node count. For findings owned by a dependency or a vendor
   script, give me the setting or the loading change instead of an edit, and say so. Anything you make lazy must still be
   what a crawler sees - do not hide indexable content behind an interaction.
```

## 4. Extend: new stage (builder prompt)
```
We're creating a new audit stage as a SEPARATE file in .ai/web-performance/audit/.
Topic: <e.g. images / fonts / INP>.

1. Create NN-name.md (number in steps of 10, after the last stage) in the same format
   as existing stages: header "> Can be run STANDALONE...", Input/Output, steps
   (Ask for / Do / Save to / Done when), and a Progress update at the end.
2. Add the stage to .ai/web-performance/audit/00-index.md under "Stage order" and a new
   row to "## Progress" in .ai/web-performance/site-profile.md.
3. Write it as an instruction that, WHEN RUN, interviews me: asks for missing data one
   at a time, reads site-profile.md, writes to .ai/web-performance/, then acts.

Rules: don't touch other stage files, stay tool-agnostic, the stage must be
self-contained (run standalone). Show a draft, save only after my OK.
```

## 5. Build the network stage: DNS / TLS / HTTP (builder prompt)
Paste after workshop 05. Produces audit/20-network-dns-tls-http.md.
```
Build a NEW audit stage as a SEPARATE file: .ai/web-performance/audit/20-network-dns-tls-http.md.
Follow the format of the existing stages and the card 4 rules above (standalone,
question-driven, wire it into 00-index.md "Stage order" and site-profile.md "## Progress").

Topic: the network layer BEFORE the first byte - DNS, TLS/SSL, HTTP protocol.
Input it must read from site-profile.md: ## Project (Public URL) and ## Page types.
It must NOT invent the domain - read it from the profile, or ASK me and wait.

SCOPE & SPEED (build this in): DNS/TLS/HTTP/SSL Labs are HOST properties, not per-page -
run every check ONCE PER HOSTNAME (if all page types share one host, run the stage ONCE).
Cap every `curl` with `--max-time 8`; run independent checks in parallel.

When RUN, the stage must (these are "Do" items - it runs the tools itself, then writes
results to site-profile.md > ## Network and findings.md):
- Derive HOST + APEX from my URL; if page types live on other hostnames, test each host.
- DNS: `dig NS {APEX}` -> classify the provider and decide anycast yes/no (Cloudflare,
  Route53, Google, NS1, DNSMadeEasy = anycast; tld.pl/nazwa.pl/home.pl/ovh/registrar NS
  = probably not). `dig A/AAAA/CNAME` for TTL, IPv6, CDN alias. Measure DNS latency with
  `dig {HOST} +noall +stats` (Query time) run twice for cold/warm - NOT a `curl` GET loop
  (curl waits for the whole page, 50s+ on a slow server). If not anycast or cold >100ms ->
  recommend moving DNS to a global-anycast provider (Cloudflare DNS, free).
- TLS: `openssl s_client -tls1_3` then fall back to `-tls1_2`; record the HIGHEST version
  that negotiates + the cipher. TLS 1.2 only -> recommend enabling TLS 1.3 (saves ~1 RTT).
- HTTP: `curl --http2` and `--http3` + `alt-svc` header -> record h2/h3/1.1. 1.1 only ->
  enable HTTP/2; h2-but-no-h3 with a CDN -> enable HTTP/3.
- HSTS + http->https redirect + full connection timing (`curl -w DNS/Connect/TLS/TTFB`).
  Note: a huge TTFB is a SERVER problem - record it and hand it to a later stage, this
  stage owns DNS/TLS/HTTP only.
- SSL Labs: run ONCE per host (not per URL). Prefer the API - CACHE FIRST (instant), fresh
  scan only if needed:
    cached:  "https://api.ssllabs.com/api/v3/analyze?host={HOST}&fromCache=on&maxAge=24"  (READY in <1s if a recent scan exists)
    if not READY: "...&startNew=on&all=done" then poll "...&all=done" (1-2 min; kick it off in the BACKGROUND at the start and collect at the end).
  From the READY JSON take grade, protocols, alpnProtocols, hstsPolicy, sessionResumption,
  supportsRc4, forwardSecrecy. If you have a browser MCP instead, open
  https://www.ssllabs.com/analyze.html?viaform=on&d={HOST}&hideResults=on
  wait ~2 min and read the grade + warning bullets. No API and no browser -> ASK me to
  open that URL and paste the grade + bullets. Write SSL_LABS_GRADE, the confirmed TLS
  version, HSTS verdict and session-resumption (SSL Labs is authoritative here - curl
  cannot judge resumption) into ## Network, and append the recommendations to findings.md.
- End with a "### Change list": a numbered, prioritised TODO of exactly what I must change
  in my project (DNS provider, TLS 1.3, HTTP/2-3, HSTS, resumption), ordered bad -> info.

The GOAL of running this stage: I walk away knowing precisely which DNS provider / TLS
version / HTTP protocol to switch to. Show me the draft file first, save only after my OK.
```

## 6. Build the preconnect / connection-warming stage (builder prompt)
Adds the next step: warming connections to external domains. Produces audit/30-preconnect.md.
```
Build a NEW audit stage as a SEPARATE file: .ai/web-performance/audit/30-preconnect.md.
Same rules as card 4 (standalone, question-driven, wire into 00-index.md + site-profile.md).
It runs AFTER stage 20 and reuses ## Network + ## Page types from site-profile.md.

Topic: connection warming to EXTERNAL (cross-origin) domains via preconnect / dns-prefetch.
Every third-party origin the page pulls from (fonts, LCP image host, analytics, GraphQL/
API, CDN) pays a fresh DNS+TCP+TLS handshake. `preconnect` opens that connection early so
the critical request does not wait for it.

SCOPE & SPEED (build this in): run on ONE representative page (the most important / heaviest
template, usually the homepage) - do NOT loop over every page type (a full page load per type
multiplies the wait). Spot-check a second page only if it clearly pulls DIFFERENT third-parties.

When RUN, the stage must (Do items):
- On the ONE representative page, collect the cross-origin origins it uses:
  (a) from static HTML: `curl -sL --max-time 8 {URL}` and extract external hosts referenced
      in <head> and early markup (fonts, CSS/JS hosts, LCP <img>/preload src, API endpoints);
  (b) from the REAL waterfall at runtime, using the webperf snippet skill: run the
      `webperf-loading` skill's Resource-Hints.js and Resource-Hints-Validation.js via
      Chrome DevTools MCP (mcp__chrome-devtools__evaluate_script), which lists every
      cross-origin the page actually connects to, flags existing hints that are UNUSED,
      and flags cross-origins that WOULD benefit from a preconnect. Prefer runtime data
      over HTML guessing.
- Read the hints already present (rel="preconnect" / "dns-prefetch") and their crossorigin.
- Rank cross-origins by how early + how critical the first request to them is (fonts and
  the LCP image host first; late analytics last).
- HARD LIMIT: recommend AT MOST 4 preconnects (Lighthouse guidance - each preconnect holds
  an open connection and beyond ~4 they compete and hurt). If more than 4 origins qualify,
  keep the top 4 by earliest-critical-request and put the rest on `dns-prefetch` (cheap,
  DNS only). State explicitly which origins you dropped from preconnect and why.
- Font preconnects MUST carry crossorigin (anonymous) or the browser opens a second
  connection - flag any missing crossorigin. Also flag existing preconnects that no
  request uses (dead hints - remove them).
- Output a concrete change list: the exact <link rel="preconnect" ...> / dns-prefetch tags
  to add (with crossorigin where needed) and any hints to remove, capped at 4 preconnects.

Write results to site-profile.md > ## Network (### Preconnect) and findings.md. Show me
the draft file first, save only after my OK.
```

## 7. Build the cache & CDN stage (builder prompt)
Adds the edge/cache layer: is the fast number real, or is a cache hiding a slow origin.
Produces audit/40-cache-cdn.md.
```
Build a NEW audit stage as a SEPARATE file: .ai/web-performance/audit/40-cache-cdn.md.
Same rules as card 4 (standalone, wire into 00-index.md + site-profile.md). This is a DO
stage (run the tools yourself, do not interview) - the ONE thing not to invent is the
domain/URLs: read them from site-profile.md, or ASK and wait. Runs AFTER stage 20.

Topic: the CACHE / EDGE layer only - is there a CDN in front, what is cached and for how
long, HIT vs MISS, and (the point) how much of the measured speed is real vs a cache
masking a slow origin. Do NOT fix the origin here; a cache-bypassed TTFB in seconds is a
SERVER problem - record it and hand it to stage 50.

SPEED: cap every curl with `--max-time 20` (the origin can be slow); prefer `-sI`.

When RUN, the stage must (Do items):
- Detect the edge/CDN in front: `curl -sI https://{HOST}/` and read edge fingerprints
  (cf-ray/cf-cache-status = Cloudflare, x-vercel-cache = Vercel, x-served-by/x-cache/via =
  Fastly/Varnish, x-cache ...cloudfront = CloudFront). Record which, or "none - direct origin".
- HTML cacheability per page type: request the same URL TWICE, read the edge cache-status
  header (cf-cache-status / x-vercel-cache / x-cache) - MISS then HIT = edge-cached;
  DYNAMIC/BYPASS/no-store/private on both = not cached. Read Cache-Control / Age / Vary; a
  wide Vary (e.g. Cookie) or set-cookie on the HTML defeats full-page cache - flag it.
- Cache-bust probe (the key check): append a junk query param the cache never saw
  (`?__perfbust=1`) and re-measure TTFB. If it busts cache and TTFB jumps to seconds, the
  fast number was the CACHE - record cached TTFB vs ORIGIN TTFB and the gap (utm/ad-click
  params are everyday cache misses, so the origin TTFB is what returning-from-campaign
  users pay). This is the "cache pudruje TTFB, nie usuwa" lesson, made measurable.
- Static assets: sample a hashed JS/CSS bundle, the LCP image, a font - check they are
  edge-cached (HIT on repeat) with long immutable Cache-Control (max-age=31536000,
  immutable). Flag short/absent max-age or MISS-on-repeat, and any asset served straight
  from origin.
- Change list + ALWAYS end with the handoff line: "ORIGIN TTFB = <n>s (cache-bypassed) ->
  owned by stage 50 (backend triage). Cache masks it; it does not remove it."

Write results to site-profile.md > ## Cache & CDN and findings.md. Show me the draft file
first, save only after my OK.
```

## 8. Build the backend triage stage (builder prompt)
The TTFB endgame: isolate the real origin TTFB, classify it, and hand off routed next
steps - WITHOUT running the profiler. Produces audit/50-backend-triage.md.
```
Build a NEW audit stage as a SEPARATE file: .ai/web-performance/audit/50-backend-triage.md.
Same rules as card 4 (standalone, wire into 00-index.md + site-profile.md). DO stage with a
couple of "Ask for" items. Runs AFTER stage 40; reads the origin TTFB from ## Cache & CDN
and the TTFB baselines from findings.md.

HARD BOUNDARY (build this in): the stage answers three questions from OUTSIDE the server -
(1) the real origin TTFB with cache removed, (2) which request class is slow (static vs
dynamic, one route vs all), (3) what CLASS of backend problem it is - and then STOPS with a
routed next-steps list. It does NOT open Query Monitor / Code Profiler / Sentry / any APM
and does NOT change code. That deep-dive is a SEPARATE human step. Keep "measure + route"
apart from "profile + fix".

When RUN, the stage must (Do items):
- Isolate the ORIGIN TTFB: force a cache MISS/bypass (junk param, or a known-uncacheable
  route, or the origin IP if recorded in the profile - never guess it) and measure TTFB a
  few times, cold + warm. Record min/typical and whether cold >> warm.
- Read the `Server-Timing` response header. If present, parse the spans (db/render/cache).
  If absent, add "instrument Server-Timing" to next steps (cheapest backend observability).
- TTFB by request class: compare a STATIC/prebuilt route (robots.txt/favicon/static file =
  the server floor) vs a DYNAMIC/app route vs any API/data endpoint. Ask me (one at a time)
  which routes are static vs per-request and where the data comes from, only if unclear.
  Static floor fast but app routes slow -> cost is in render/data fetch; even the static
  floor slow -> cost is infra/host-level.
- Classify the bottleneck CLASS: A slow every request incl. warm (compute/DB/external fetch
  on render), B cold >> warm (cold start / cache-miss regeneration), C TTFB scales with
  content (N+1 / unbounded query/fetch), D even static floor slow (infra/host sizing,
  PHP-FPM saturation), E origin actually fine (revisit stages 40/20).
- OUTPUT a "backend deep-dive: next steps" list ROUTED BY STACK - which tool to open and
  what to look for, tied to the class:
  - WordPress: Query Monitor (per-request DB time, N+1, slow hooks, blocking HTTP API
    calls); Code Profiler / Xdebug / Blackfire (function-level flamegraph); wow-audit-check.php
    drop-in (server/infra floor: opcache, PHP-FPM vs RAM, MySQL buffer pool + missing indexes
    on postmeta/options, autoload/transient bloat, object-cache type); Novamira MCP (AI runs
    WP-CLI / DB queries / diagnostic PHP inside WordPress). The drop-in and Novamira execute
    actions -> they belong to the deep-dive, route to them, do not run them in this stage.
  - Next.js: `next build` (which routes are Static/SSG/ISR/SSR); the data fetch on the
    render/regeneration path (external CMS e.g. WordPress GraphQL gates the TTFB - follow it
    into the WordPress tools); tracing via Sentry / OpenTelemetry / Vercel Observability.
  - Astro: static (prerender) vs SSR adapter; slow Astro.fetch/endpoints on render; cold start.
  - Any other stack / no APM: add Server-Timing + log the slowest span, or use whatever APM
    the host offers. The routing follows the CLASS, not the tool brand.
- End with the STOP line: bottleneck class + the single tool to open next - "the AI audit
  stops at routing; the deep profile + fix happen with that tool (a separate step)."

Write results to site-profile.md > ## Backend and findings.md. Show me the draft file
first, save only after my OK.
```

## 9. Build the DOM-size stage (builder prompt)
Paste after workshop 09 (30.07). Produces audit/70-dom-size.md - use this if you would
rather have your AI generate the stage than copy the ready one.
```
Build a NEW audit stage as a SEPARATE file: .ai/web-performance/audit/70-dom-size.md.
Same rules as card 4 (standalone, wire into 00-index.md "Stage order" and site-profile.md
"## Progress"). DO stage, runs AFTER stage 60, reads ## Render start from site-profile.md.

Topic: does the SIZE AND SHAPE of the DOM cost measurable main-thread time on this page,
which part of the structure is responsible, and which code forces the recalculation.

Build these two boundaries into the file, they are the point of the stage:
- Node count is NOT the criterion. The old Lighthouse thresholds (~800 warn, ~1400 error)
  are gone; Chrome flags a large layout or style recalculation, roughly 40 ms and up. The
  stage must never report a node count as a problem on its own.
- The stage must be allowed to end in "real, but not now". It compares its own measured
  cost against LCP/FCP from stage 60 and outputs FIX NOW / FIX LATER / NO ACTION.

When RUN, the stage must (Do items, all through the chrome-devtools MCP server):
- emulate ONCE (mobile 412x765x2.6, Fast 4G, CPU 4x) and state the viewport next to every
  number - hidden subtrees and below-the-fold counts are viewport-dependent.
- performance_start_trace (reload, autoStop) x3, then performance_analyze_insight on the
  DOMSize insight. Record verbatim: total elements, DOM depth AND the element that starts
  the deepest chain, most children, and each "Layout update: Duration N ms, with X of Y
  nodes needing layout". Structural stats are identical across runs, the duration is not -
  treat sub-5 ms differences as noise. A missing DOMSize insight is a RESULT, not a failure.
- performance_analyze_insight on ForcedReflow when present: it names the function and
  script URL:line that forced synchronous layout, with per-frame ms. That is the owner.
  Distinguish first-party bundle from third-party tag - the fix differs.
- evaluate_script for the structure: scope to document.body, skip SVG internals, and report
  deepest path, elements over 12 deep, div/span share, removable single-child wrappers,
  display:none subtrees with their element counts, share below the fold, biggest subtrees,
  and uniform repeated lists. Map each signal to a move: duplicated mobile/desktop tree ->
  one structure styled by CSS; unopened modal -> render on interaction; deep chain ->
  flatten wrappers; divitis -> fragments and semantic tags; below-fold bulk -> islands or
  content-visibility; uniform list -> pagination or virtualisation.
- Prove any fix by re-tracing variants (local copies with <base href>, one edit each,
  rotated A,B,A,B, medians) and compare BOTH the duration and the "X of Y nodes" count -
  the node count is stable, so it is the honest signal when milliseconds sit in the noise.
- MAP EVERY FINDING BACK TO THE SOURCE before proposing anything - this is what makes it an
  audit of a PROJECT and not of a web page. Strip build hashes from the rendered class names
  (CSS Modules/styled-components suffixes), search the repo for the stable part, and output
  finding | token | file:line | owner, where owner is: my component/template, my theme, a
  dependency I configure (fix = a setting, not an edit), or a third-party script (markup is
  not in the repo - verify it is absent from the server HTML too - so the lever is loading
  strategy or configuration). Unattributed findings are reported as "source not identified",
  never guessed. On WordPress also search plugins and the page builder's saved post meta.
- Route fixes by stack (Next.js/React, WordPress + page builders, Astro), each pointing at
  the file from the mapping step, and add the SEO caveat: what is rendered lazily must still
  be what a crawler sees.
- State at the top of the generated file that it runs against MY project - the URL from
  site-profile.md, never a demo - and that any figure quoted in it as calibration was
  measured elsewhere and must not be copied into findings or used as a threshold.

Known traps to write into the file: viewport changes the answer; scope the script to body
or <head> pollutes the "hidden" count; SVG internals inflate element counts and are not
divitis; getComputedStyle/getBoundingClientRect in a loop force layout themselves; CSS
Selector Stats is a DevTools UI feature and is NOT available through MCP - do not claim to
have run it; selectors are matched right to left, which is why one careless wildcard rule
can drag the whole tree into a recalculation.

Show me the draft file first, save only after my OK.
```
