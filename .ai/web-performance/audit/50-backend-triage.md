# Stage 50: Backend / server triage - isolate origin TTFB, classify it, hand off next steps

> Can be run STANDALONE or from the index (.ai/web-performance/audit/00-index.md).
> Behaviour: DO stage with a couple of "Ask for" items. You measure from the OUTSIDE
> and CLASSIFY; you do NOT run the deep profiler here. Read domain/URLs/stack from
> site-profile.md; if missing/ambiguous, ASK me and WAIT - do not guess.
> Input:  site-profile.md > ## Project (URL + stack), ## Page types, ## Network,
>         ## Cache & CDN + findings.md (TTFB baselines from Stage 10, origin TTFB from Stage 40).
> Output: site-profile.md > ## Backend + findings.md, ending with a stack-routed
>         "BACKEND DEEP-DIVE - NEXT STEPS" list. Runs AFTER Stage 40.
> Idempotent: if ## Backend exists, update it, do not create a second one.

SCOPE AND HARD BOUNDARY (read first - IMPORTANT):
This stage's job is to answer THREE questions from outside the server:
  (1) What is the REAL origin TTFB, with cache removed?
  (2) Which request class is slow (dynamic vs static, one route vs all)?
  (3) What CLASS of backend problem is it (compute/DB/external-fetch/cold-start/infra)?
and then STOP with a routed next-steps list. This stage does NOT open Query Monitor,
Code Profiler, Sentry traces or any APM, and does NOT change code. That deep profiling
and the fixes are a SEPARATE step done by a human with the tool this stage points to
(see the Task 2 brief). Keeping "measure + route" apart from "profile + fix" is
deliberate - do not merge them into this stage.

## 50.0 Read what earlier stages already found (Do, no new measurement yet)
- From findings.md: lab TTFB per page type (Stage 10.3) and field TTFB (Stage 10.6).
- From site-profile.md > ## Cache & CDN: the ORIGIN TTFB (cache-bypassed, Stage 40.3)
  and whether HTML is edge-cached.
- From site-profile.md > ## Project: the STACK (this drives the routing in 50.5) and
  whether code is local (path).
- If TTFB is already known to be fine (< ~0.6s origin) on every type, say so and skip
  to 50.5 with "no backend bottleneck found - nothing to route." Otherwise continue.

## 50.1 Isolate the ORIGIN TTFB - remove the cache (Do)
- Goal: the server's true think-time, uncontaminated by edge cache.
- Force a cache MISS/bypass and measure TTFB several times (cold + warm):
  `for i in 1 2 3; do curl --max-time 30 -so /dev/null -w 'ttfb=%{time_starttransfer} total=%{time_total}\n' "https://{URL}?__perfbust=$i"; done`
  (or hit a known-uncacheable route, or set a Cloudflare "Bypass cache" rule for the test).
- If you can reach the origin directly (origin IP / an unproxied hostname from
  ## Network), measure it too - that removes the edge round-trip entirely. Do NOT
  guess the origin IP; only use one recorded in site-profile.md or that I give you.
- Record: origin TTFB (min / typical), and whether run 1 (cold) is much worse than
  runs 2-3 (warm). Cold >> warm is a signal (see 50.4).

## 50.2 Server-Timing - does the server already tell you the breakdown? (Do)
- `curl --max-time 30 -sI "https://{URL}?__perfbust=1" | grep -i '^server-timing'`
- If present, parse the spans (e.g. `db;dur=820, render;dur=140, cache;dur=5`) - this
  often points straight at the culprit layer with zero profiling.
- If ABSENT: record "no Server-Timing" and add to the next-steps list "instrument
  Server-Timing" - it is the cheapest backend observability you can add and it makes
  every future audit faster. Do not treat its absence as a finding about speed, only
  about visibility.

## 50.3 TTFB by request class - narrow WHERE the cost is (Do + Ask if unclear)
- Goal: separate a slow FRAMEWORK/render path from a slow WHOLE SERVER.
- Compare origin TTFB across classes, using the page types from ## Page types plus:
  - a STATIC/prebuilt asset served by the same origin (e.g. `/robots.txt`,
    `/favicon.ico`, a static HTML file) - this is the server's floor with ~no app logic;
  - a DYNAMIC/app route (the page types);
  - if there is an API/data endpoint the front-end calls (REST/GraphQL), measure it too.
- Ask for (ONLY if you cannot tell from the code/profile): which routes are
  static/prebuilt vs rendered-per-request, and where the page data comes from
  (same server, or an external API/CMS). Ask one at a time, wait, write to ## Backend.
- Read the signal:
  - Static floor is fast (ms) but app routes are slow -> the cost is in APP RENDER /
    DATA FETCH, not the box. Route to the framework/DB/external-fetch tools (50.5).
  - EVEN the static floor is slow (seconds) -> the cost is INFRA/host-level (PHP-FPM
    saturation, swap, noisy neighbour, DB connection, undersized droplet). Route to
    host/infra checks, not app code.

## 50.4 Classify the bottleneck CLASS (Do - decide, then route)
From 50.1-50.3, pick the closest class (it drives 50.5):
- A. Slow on EVERY request incl. warm, app routes only -> synchronous work per render:
  DB queries, or a blocking external fetch (CMS/GraphQL/REST) on the render path.
- B. Cold >> warm, warm is fine -> COLD START / cache-miss regeneration (serverless
  cold start, ISR/on-demand rebuild, opcache/JIT cold, empty object cache). The fix is
  keeping it warm / precomputing, not shaving the fast path.
- C. TTFB SCALES with content/size (bigger listing = slower) -> unbounded work:
  N+1 queries, fetching-all-then-filtering, missing pagination/index.
- D. Even the static floor is slow -> INFRA/host: undersized server, PHP-FPM/worker
  saturation, DB on the same box starved, swapping.
- E. Origin TTFB is actually fine; the slow field number came from cache misses on a
  thin cache or from network -> revisit Stage 40 / Stage 20, not the backend.
Write the chosen class + the evidence to ## Backend.

## 50.5 OUTPUT - "Backend deep-dive: next steps", ROUTED BY STACK (Do - the deliverable)
This is what the stage produces. Pick the block(s) for the stack in ## Project. For
each, give: WHICH tool to open, WHAT to look for (tied to the class from 50.4), and
what to bring back. End with the explicit STOP line. Do NOT run these tools now.

WordPress (PHP):
- Query Monitor (plugin) - per-request panel: total DB query time + the slowest
  queries, number of queries (N+1 -> class C), slow hooks, and "HTTP API Calls" (a
  blocking wp_remote_get to an external API on render -> class A). Load the slow URL
  logged-in-as-admin and read the panel. Best for class A/C (the per-request path).
- Code Profiler / a function-level profiler (Code Profiler, Xdebug, Blackfire,
  Tideways) - flamegraph of where PHP time goes: a plugin, a theme function, an
  external call. Reach for it when Query Monitor points at "slow" but not "which line".
- wow-audit-check.php (drop-in from the session repo:
  github.com/bartekmis/wordpress-full-stack-performance-audit) - one randomised PHP
  file to the WP root, hit over HTTP, returns HTML + a JSON blob you (or the AI) can
  parse. Covers the SERVER/INFRA floor Query Monitor cannot see: PHP/opcache, PHP-FPM
  worker count vs RAM, CPU/RAM/load benchmark, MySQL buffer pool + slow-query log +
  missing indexes on postmeta/options, autoload/transient/revision bloat, object-cache
  type. Best first look for class D (infra) and class B (opcache/object cache). After:
  delete the file and verify it 404s. NOTE: it uploads a file = an ACTION, so it belongs
  to the deep-dive (Task 2), not to this read-only audit stage - route to it, don't run it here.
- Novamira MCP (novamira.ai, WordPress-only) - an MCP that gives the AI direct in-WP
  access: run WP-CLI, query the DB, inspect options/autoload, execute diagnostic PHP in
  the WordPress process. This is the CHANNEL the AI uses in the deep-dive to actually
  run the checks above and act on them (Task 2). It executes code, so - like the drop-in
  - it is out of scope for this read-only audit stage; name it as a next step, do not use it here.
- Object cache (Redis/Memcached) + opcache status - the fix side for class B (cold/warm
  gap) and to cut repeated query cost. wow-audit-check reports whether they are even on.

Next.js:
- First: WHICH rendering mode is the slow route actually using? `next build` output
  lists each route as Static / SSG / ISR / SSR (`getServerSideProps` = per-request).
  A route you THINK is static (ISR/`getStaticProps` + `revalidate`) but that measures
  seconds of TTFB is regenerating or blocking on data on the render path - that is the
  finding (class A/B).
- Data fetch on the render/regeneration path - if the page pulls from an external CMS
  (e.g. a WordPress GraphQL / `WORDPRESS_API_URL` endpoint), the Next.js TTFB is gated
  by THAT backend. Trace the awaits: sequential fetches = a waterfall (class A);
  fetch-all-then-slice = class C. The frontend stack is fast; the data source is the wall.
  -> follow the trail into the WordPress block above for the CMS itself.
- Tracing/APM - if Sentry is wired in (`@sentry/nextjs`, `sentry.*.config.ts`), use its
  Performance/spans to see the slow server span; or OpenTelemetry
  (`experimental.instrumentationHook` / `instrumentation.ts`), Vercel Observability, or
  New Relic. Read the slowest span; do not add a second APM if one is already present.
- Server-Timing / a manual `console.time` around the data fetch in the route - cheapest
  local confirmation of which await dominates.

Astro:
- Is the page static (prerendered) or SSR (an adapter + `export const prerender =
  false`)? A static page should have a near-floor TTFB; a slow one means it is SSR or
  hitting a slow endpoint on render. Move slow-but-static-eligible pages to prerender/ISR.
- Astro API endpoints / `Astro.fetch` during render -> same as Next.js: trace the
  external fetch (often the same CMS/DB) and route to that backend's tools.
- Adapter/host runtime (Node/serverless) cold start -> class B.

Any other stack / no APM installed:
- Add request-level `Server-Timing` (50.2) and log the slowest span, OR reach for
  whatever APM the host offers (New Relic, Datadog, Dynatrace, Sentry, your host's
  built-in profiler). If none: a one-request timing log around the data fetch + DB call
  is enough to place the cost in a layer.
- The routing does not depend on the tool brand - it depends on the CLASS from 50.4:
  class A/C -> query/fetch profiler; class B -> warm-up/cache/cold-start; class D ->
  host/infra sizing.

## Save results
- site-profile.md > ## Backend: origin TTFB (cache-bypassed), static-floor vs app-route
  TTFB, Server-Timing present?, chosen bottleneck CLASS + evidence, and the routed
  next-steps list (which tool, what to look for).
- findings.md: dated entry with the raw numbers and the classification reasoning.

## STOP line (end every run with this, verbatim intent)
"Backend triage done. Bottleneck class = <A/B/C/D/E>. Next: OPEN <the routed tool> and
find the root cause there - that is a SEPARATE step (see Task 2), not part of this
audit. The AI audit stops at routing; the deep profile + fix happen with the tool
above."

## At the end
Check off in site-profile.md > ## Progress: [x] Stage 50 - backend triage. Report the
class and the single tool to open next - nothing more.
