# Site profile

This file holds the audit DATA and STATE. The AI fills it in while running stages
(asking you, writing here). You may also fill it in manually.

Note: the fields below are filled ONLY from the user's answers (or from "Do" step
outputs). The AI must not auto-derive them from code, MCP or the network.

## Project
- Public URL: <TODO>
- Stack: <TODO e.g. Next.js / WordPress / Astro>
- Code local: <TODO yes/no, path>
- Hosting / server: <TODO>
- Deploy: <TODO e.g. CI/CD>

## Page types
| Type | Representative URL |
|------|--------------------|
| <TODO> | <TODO> |

## Measurement
- WebPageTest baseline: <TODO status>
- CrUX (public field): <TODO status / no data>
- RUM: <TODO which / none>
- RUM access / MCP: <TODO>
- RUM data (p75 per URL): <TODO status>

## Network
Filled by Stage 20 (audit/20-network-dns-tls-http.md). Leave <TODO> until run.

- Host / Apex: <TODO>
- DNS provider / nameservers: <TODO>
- DNS anycast: <TODO yes / probably not>
- DNS lookup cold / warm: <TODO ms / ms>
- A record TTL / IPv6 (AAAA): <TODO>
- TLS version / cipher: <TODO e.g. TLS 1.3>
- HTTP protocol: <TODO 1.1 / h2 / h3>
- HSTS: <TODO present max-age / absent>
- HTTP->HTTPS redirect: <TODO>
- SSL Labs grade / resumption: <TODO>

### Change list
- <TODO filled by Stage 20.7: what to change in the project>

### Preconnect
Filled by Stage 30 (audit/30-preconnect.md). Leave <TODO> until run.
- External origins used (count): <TODO>
- Preconnect (max 4, earliest-critical): <TODO>
- Dns-prefetch / dropped: <TODO>
- Dead hints to remove: <TODO>

## Cache & CDN
Filled by Stage 40 (audit/40-cache-cdn.md). Leave <TODO> until run.
- Edge / CDN in front: <TODO which / none>
- HTML cacheable per page type: <TODO cached (HIT/Age) / dynamic (MISS/no-store)>
- Cache-bust: cached TTFB vs ORIGIN TTFB: <TODO n / n + gap>
- Static assets long+immutable & edge-cached: <TODO yes / origin-bound assets>

## Backend
Filled by Stage 50 (audit/50-backend-triage.md). Leave <TODO> until run.
- Origin TTFB (cache-bypassed, cold / warm): <TODO n / n>
- Static-floor TTFB vs app-route TTFB: <TODO n / n>
- Server-Timing present: <TODO yes (spans) / no - add it>
- Bottleneck class (A compute/DB, B cold-start, C N+1/scale, D infra, E false-alarm): <TODO>
- Next step (tool to open): <TODO routed by stack>

## Render start
Filled by Stage 60 (audit/60-render-start.md). Leave <TODO> until run.
- Start Render / FCP / LCP (lab, slow page type): <TODO n / n / n>
- Render-blocking resources: <TODO list>
- Resources before first paint - verdict count (DELETE / DEFER / MOVE / ON DEMAND /
  SELF-HOST / KEEP): <TODO n / n / n / n / n / n>
- Waterfall step at: <TODO ms - and what was blocking the parser there>
- Dead or errored resources (not visible in the waterfall): <TODO list / none>
- Hypothesis 1 / 2 / 3: <TODO> -> verdict <SUPPORTED / REGRESSION / INCONCLUSIVE>
- Measured delta of the winning change (FCP / LCP / DCL, medians): <TODO>
- Fixes to apply (file + change): <TODO routed by stack>
- Re-tested on WebPageTest, same profile: <TODO date + before/after>

## DOM size
Filled by Stage 70 (audit/70-dom-size.md). Leave <TODO> until run.
- Viewport the numbers were measured on: <TODO e.g. 412x765x2.6 mobile, Fast 4G, CPU 4x>
- DOMSize insight present: <TODO yes / no - no large recalculation on this load>
- Total elements / DOM depth / most children: <TODO n / n / n>
- Element starting the deepest chain: <TODO tag + class, from the insight>
- Largest layout or style recalculation (median of 3 runs + spread): <TODO n ms +/- n, X of Y nodes>
- Forced reflow attributed to: <TODO script URL:line, or 'none - ordinary rendering'>
- Structure findings (duplicated tree / hidden panel / deep chain / wrappers / below-fold /
  repeated list): <TODO top 2-3 with element counts>
- Priority verdict: <TODO FIX NOW / FIX LATER / NO ACTION> because <TODO vs LCP/FCP>
- Fixes to apply (file + change): <TODO routed by stack>
- Re-measured after the fix (median + node count): <TODO before/after>

## Scripts
Filled by Stage 80 (audit/80-scripts-and-third-party.md). Leave <TODO> until run.
- Viewport the numbers were measured on: <TODO e.g. 412x765x2.6 mobile, Fast 4G, CPU 4x>
- Scripts on the audited page type (script / origin / 1st-3rd party / mode / position /
  transfer / in server HTML): <TODO one row per script>
- Main-thread time, first party vs third party (median of 3 runs + spread): <TODO n ms / n ms +/- n>
- Heaviest by MEASURED time (not by size), and when it runs: <TODO script, n ms, before/after FCP>
- Does the size order disagree with the time order: <TODO yes/no - state it if yes>
- SPOF - third party in head without async/defer: <TODO none, or vendor + Start Render impact from the WPT SPOF test>
- Tag manager cascade (container -> tags -> injected scripts): <TODO levels + timing, or 'no tag manager'>
- Container config reviewed (exported JSON): <TODO tags unused / duplicated / interaction-triggered>
- Interaction-triggered tags (route to INP stage): <TODO list, or none>
- Coverage split - dead / needed later / needed now: <TODO n KB / n KB / n KB>
- Priority verdict: <TODO FIX NOW / FIX LATER / NO ACTION> because <TODO vs LCP/FCP>
- Per-script decisions (remove / defer / on demand / keep + preconnect): <TODO routed by stack>
- Experiment result (blocked request, then shipped strategy): <TODO SUPPORTED / REGRESSION / INCONCLUSIVE + before/after>

## Progress
- [ ] Stage 10 - profile and baseline
- [ ] Stage 20 - network: DNS / TLS / HTTP
- [ ] Stage 30 - preconnect (external domains)
- [ ] Stage 40 - cache & CDN (edge)
- [ ] Stage 50 - backend triage (TTFB)
- [ ] Stage 60 - render start (first paint)
- [ ] Stage 70 - DOM size (recalc cost)
- [ ] Stage 80 - scripts and third party
