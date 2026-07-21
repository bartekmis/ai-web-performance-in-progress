# Stage 40: Cache and CDN - what is cached, where, and what still hits the origin

> Can be run STANDALONE or from the index (.ai/web-performance/audit/00-index.md).
> Behaviour: this is a DO stage, not an interview. You run the tools yourself and
> write the results - you do NOT ask me for the data. The ONE thing you must NOT
> invent is the domain/URLs: read them from site-profile.md. If missing/ambiguous,
> ASK me and WAIT - do not guess a hostname.
> Input:  site-profile.md > ## Project (Public URL) + ## Page types + ## Network.
> Output: site-profile.md > ## Cache & CDN + findings.md (cache map + change list).
> Runs AFTER Stage 20 (network). Idempotent: if ## Cache & CDN exists, update it.

SCOPE: this stage owns the CACHE / EDGE layer only - is there a CDN in front, what
is cached and for how long, HIT vs MISS, and (the point) how much of the measured
speed is real vs merely a cache masking a slow origin. This stage does NOT fix the
origin. If a request bypasses cache and TTFB balloons, that is a SERVER problem:
record the origin TTFB and hand it to Stage 50 (backend triage). Do NOT try to fix
it here.

WHY THIS STAGE (say once, then run): a full-page/edge cache can make a site look fast
while the origin is still slow. The moment cache misses - a query param, a new URL, a
purge, a logged-in user - the real origin TTFB comes back. Before you profile the
backend you must know which numbers are cached and which are the true origin, so you
optimise the right thing.

## 40.0 Derive the targets (read the profile - do not invent)
- Read site-profile.md > ## Project (Public URL) and ## Page types (one URL per type).
- Read ## Network - note whether Stage 20 saw a CDN/CNAME in front. This stage
  re-checks it live (the CDN may have been added AFTER Stage 20 ran, e.g. Cloudflare).
- HOST = the URL's hostname. If ## Project has no Public URL, ASK me and WAIT.

SPEED RULES: cap EVERY curl with `--max-time 20` (the origin can be slow). Prefer
`-sI` (headers only) so you do not download the whole body just to read cache headers.

## 40.1 Is there a CDN / edge in front? (Do)
- `curl --max-time 20 -sI https://{HOST}/` and read the response headers for edge
  fingerprints. Any of these = a CDN/edge is serving the response:
  - Cloudflare: `cf-ray`, `cf-cache-status`, `server: cloudflare`.
  - Vercel: `x-vercel-cache`, `server: Vercel`, `x-vercel-id`.
  - Fastly / Varnish: `x-served-by`, `x-cache`, `via`, `x-varnish`.
  - CloudFront: `x-cache: Hit/Miss from cloudfront`, `via: ...cloudfront`.
  - Netlify: `x-nf-request-id`; Akamai: `x-akamai-*`.
- Record which edge (or "none - direct origin"). Cross-check ## Network (CNAME/IP): a
  Cloudflare/Vercel edge but the IP still resolves to the raw origin = DNS not proxied.

## 40.2 Is the HTML document cached at the edge? (Do)
- Request the SAME page-type URL TWICE and read the edge cache-status header each time
  (`cf-cache-status` / `x-vercel-cache` / `x-cache`):
  `curl --max-time 20 -sI https://{URL} | grep -iE 'cf-cache-status|x-vercel-cache|x-cache|age|cache-control|expires|vary'`
  - First uncached hit = `MISS` / `DYNAMIC`; second = `HIT` / `Age > 0` -> HTML is
    edge-cached. `DYNAMIC` / `BYPASS` / `no-store` / `private` on both -> HTML is NOT
    cached at the edge (every visit hits the origin).
- Read the document's own `Cache-Control` (`max-age` / `s-maxage` / `no-store` /
  `private`), `Age`, `Expires`, and `Vary`. A wide `Vary` (e.g. `Vary: Cookie`) or a
  set-cookie on the HTML usually DEFEATS full-page cache - note it.
- Do this for EACH page type from ## Page types (a template may be cached on one type
  and dynamic on another).

## 40.3 Cache-bust probe - does cache mask a slow origin? (Do - the key check)
- Take one representative URL and append a junk query param the cache has never seen:
  `curl --max-time 20 -so /dev/null -w 'status=%{http_code} cache=%{header_cf-cache-status} ttfb=%{time_starttransfer}\n' 'https://{URL}?__perfbust=1'`
  (if `%{header_...}` is unsupported in your curl, re-request with `-sI` and read the
  status header manually).
- Compare against the CACHED TTFB from 40.2 (the HIT). Two outcomes:
  - Param busts cache (`MISS`/`DYNAMIC`) and TTFB jumps back to seconds -> the fast
    number was the CACHE; the true origin TTFB is the busted one. Record BOTH:
    `cached TTFB` vs `origin TTFB`, and the gap.
  - Param still `HIT` (cache ignores the query string) -> good, note the cache key
    ignores params; the origin TTFB must be measured differently in Stage 50.
- This is the 16.07 (Cloudflare) lesson made measurable: full-page cache pudruje TTFB,
  it does not remove it. Query params (utm, ad clicks, newsletters) are everyday cache
  misses, so the origin TTFB is what real returning-from-campaign users pay.

## 40.4 Static assets - are JS/CSS/images/fonts on the CDN with a long TTL? (Do)
- From ## Page types (or a quick `curl -sL --max-time 20 {URL}` of one page), pick a
  few static assets (a hashed JS/CSS bundle, the LCP image, a font).
- For each: `curl --max-time 20 -sI {ASSET_URL} | grep -iE 'cf-cache-status|x-vercel-cache|x-cache|cache-control|age|expires'`.
- Verdict per asset:
  - Hashed/immutable build assets SHOULD have `Cache-Control: public, max-age=31536000, immutable` and `HIT` on repeat. Short/absent max-age or a `MISS` on repeat = misconfigured static cache -> flag it.
  - Images/fonts should be long-lived and edge-cached too.
- Note any static asset served straight from the origin (no edge HIT) - that is load
  the CDN should be absorbing.

## 40.5 Save results + change list (Do)
- Save to site-profile.md > ## Cache & CDN (create the section if absent):
  - Edge/CDN in front: which one (or none).
  - HTML cacheability per page type: cached (HIT/Age) vs dynamic (MISS/no-store/private) + the Cache-Control / Vary that decides it.
  - Cache-bust result: cached TTFB vs ORIGIN TTFB + the gap (hand the origin TTFB to Stage 50).
  - Static assets: cached long+immutable? any origin-bound assets?
- Append to findings.md a dated entry (source, raw headers/numbers, observation,
  recommendations), same format as earlier stages.

## Change list (end every run with this)
Numbered, prioritised (highest-impact first). Typical items:
1. HTML not edge-cached but safe to cache -> add a full-page / edge cache rule (mask, not fix, per 40.3 caveat).
2. Cache defeated by `Vary: Cookie` / set-cookie on HTML -> strip the cookie for anonymous traffic so the cache can key.
3. Static assets not immutable / not edge-cached -> set `max-age=31536000, immutable` on hashed assets, put them behind the CDN.
4. Cache key includes query params (busts on every utm) -> ignore known tracking params in the cache key.
Each line: the change + one-line why. ALWAYS end with the handoff line:
"ORIGIN TTFB = <n>s (cache-bypassed) -> owned by Stage 50 (backend triage). Cache
masks it; it does not remove it."

## At the end
Check off in site-profile.md > ## Progress: [x] Stage 40 - cache & CDN. State the one
thing that matters most: is the fast number real, or is a cache hiding a slow origin?
