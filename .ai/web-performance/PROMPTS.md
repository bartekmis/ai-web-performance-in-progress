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
