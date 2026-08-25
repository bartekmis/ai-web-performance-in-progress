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

## Media
Filled by Stage 90 (audit/90-images-and-video.md). Leave <TODO> until run.
- Viewport the numbers were measured on: <TODO e.g. 412x765x2.6 mobile, Fast 4G, CPU 4x>
- Assets on the audited page type (asset / element type / origin / intrinsic px / rendered
  px x DPR / format / transfer / loading / fetchpriority / in server HTML): <TODO one row
  per asset>

### LCP element (latency problem - its own verdict)
- What it is, and at which viewport: <TODO element + viewport>
- Delivered as: <TODO img in server HTML / injected by script / css-background / carousel slide>
- Phase split (median of 3 + spread): TTFB <TODO> / load delay <TODO> / load time <TODO> /
  render delay <TODO>
- Dominant phase -> problem class: <TODO discovery / transfer / render (route back to 60/80)>
- Lazy-loaded? Priority granted? Cross-origin and preconnected?: <TODO / TODO / TODO>
- Priority verdict: <TODO FIX NOW / FIX LATER / NO ACTION> because <TODO>
- Highest-value single change: <TODO>

### Everything else (bandwidth problem - its own verdict)
- Total image transfer for this page type: <TODO n KB>
- Oversized images (intrinsic / rendered x DPR ratio > 2): <TODO n of n>
- Worst 3 by wasted pixels + cause (no srcset / wrong sizes / wrong variants): <TODO>
- currentSrc vs src disagreement (what my users actually downloaded): <TODO>
- Optimisation moment in use: <TODO upload time / delivery time / both / neither>
- Format coverage (modern format served, with fallback): <TODO>
- CSS backgrounds below the fold (no native lazy): <TODO list / none>
- Missing width/height or aspect-ratio (ROUTE to CLS stage, do not measure here): <TODO list>
- SVG embedding (inline / img / sprite) + document weight from inline SVG: <TODO n KB>
- base64 / data URI payload in the document: <TODO n KB / none>
- Video: preload value, poster, GIFs to replace, third-party embeds + player cost: <TODO / none>
- Priority verdict: <TODO FIX NOW / FIX LATER / NO ACTION> because <TODO vs LCP/FCP>
- Per-asset decisions (resize / convert / compress / defer / preconnect): <TODO routed by stack>
- Experiment result: <TODO SUPPORTED / REGRESSION / INCONCLUSIVE + before/after>

## Fonts
Filled by Stage 100 (audit/100-fonts.md). Leave <TODO> until run.
- Viewport the numbers were measured on: <TODO e.g. 412x765x2.6 mobile, Fast 4G, CPU 4x>
- Font files loaded on the audited page type (file / family-weight-style / origin / format /
  transfer / unicode-range / font-display / preloaded? / used above the fold? / elements):
  <TODO one row per FILE, not per declaration>
- @font-face declarations in my CSS vs faces actually loaded: <TODO n vs n>
- Faces loaded but used by zero elements: <TODO list, or none>

### The chain (latency - where the cost really is)
- Chain per file (A inline+preload / B inline / C own stylesheet / D third-party stylesheet /
  E via @import / F injected by script) + hops before the request: <TODO one row per file>
- Deepest chain on this page type: <TODO letter, file, n hops>
- Files whose request starts BEFORE Start Render: <TODO list> (competing with the critical path)
- Files whose request starts AFTER Start Render: <TODO list> (repaint + shift, not a slow start)

### The budget (this is the number the stage exists to produce)
- Is the LCP element text or an image (from Stage 90)?: <TODO text in <face> / image>
- Critical face - renders the LCP element / primary above-fold copy: <TODO family + weight>
- PRELOAD BUDGET for this page type: <TODO N = 0 / 1 / 2> because <TODO>
- Currently preloaded files, KEEP or REMOVE against the budget: <TODO per file>
- Preload correctness (as=font, type, crossorigin, URL identical to @font-face): <TODO / TODO>

### The trade-off (there is no free option)
- font-display in effect per face, and whether it was chosen or inherited: <TODO>
- Cost this project is choosing: <TODO invisible text (block) / layout shift (swap) /
  font may not appear (optional)>
- Layout shift attributable to the font swap (value + timestamp vs font load): <TODO>
- Metric-matched fallback (size-adjust / ascent-override / descent-override) present?: <TODO>

### Bytes (second-order lever)
- Total font transfer for this page type / number of faces: <TODO n KB / n>
- Weights and styles downloaded vs actually used: <TODO>
- Variable font compared against the static set (actual file sizes): <TODO / not applicable>
- Subsetting and unicode-range in use, character coverage vs my content: <TODO>
- Licence permits modification/subsetting?: <TODO yes / no / free font>
- Subset pass on the heaviest face (Wakamai Fondue -> Transfonter): <TODO file, what was
  dropped, BEFORE n KB -> AFTER n KB, rendering verified incl. Polish diacritics: yes/no>
- Formats served (WOFF2 or a finding): <TODO>
- Icon font: glyphs available vs glyphs used: <TODO / none>

### Origin and cache
- First-party or third-party font origin: <TODO> (preconnected? see Stage 30: <TODO>)
- Cloudflare Fonts in use as a mitigation: <TODO yes / no / not applicable>
- Cache TTL on font files: <TODO>

- Inventory source: <TODO WebPerf Snippets skill / snippet from the site / hand-rolled>
- Where I departed from the tool's preload advice, and why: <TODO>
- Priority verdict: <TODO FIX NOW / FIX LATER / NO ACTION> because <TODO vs LCP/FCP/TTFB>
- Per-file policy (preload / eager / defer / drop + font-display): <TODO routed by stack>
- Experiment result (baseline vs preload-critical-only vs preload-all): <TODO SUPPORTED /
  REGRESSION / INCONCLUSIVE + FCP, LCP and CLS per variant>

## Consent / CMP
Filled by Stage 110 (audit/110-consent-cmp.md). Leave <TODO> until run.
- Viewport the numbers were measured on: <TODO e.g. 412x765x2.6 mobile, Fast 4G, CPU 4x>
- Cookies/storage cleared before every run (banner only shows to a first-time visitor): <TODO yes>
- CMP present / vendor: <TODO name, or none - then the stage stops here>
- Script origin (first- or third-party) / transfer / loading attribute: <TODO / TODO / TODO>
- Position in the document and how it arrives (in HTML / tag inside GTM / injected): <TODO>
- Tag manager present (container id) / consent framework (Consent Mode v2, TCF, none): <TODO / TODO>
- Two independent entry points (tag manager AND hardcoded script)?: <TODO yes - ordering is now
  a hand-maintained invariant / no>

### Failure 1 - render start (LOADING)
- Render-blocking?: <TODO yes / no> - chain and hops before the request: <TODO>
- Start Render / FCP with the CMP origin BLOCKED vs baseline: <TODO / TODO> (this is the
  measured ceiling on any loading fix, not an estimate)
- Verdict: <TODO FIX NOW / FIX LATER / NO ACTION>

### Failure 2 - LCP (CLASSIFICATION)
- LCP element as a first-time visitor (banner present): <TODO element + time>
- LCP element without the banner (from Stage 90): <TODO element + time>
- Is the banner (or a node inside it) the LCP element?: <TODO yes / no / it changed the element>
- Banner area on screen at this viewport: <TODO>
- Verdict: <TODO FIX NOW / FIX LATER / NO ACTION>

### Failure 3 - INP (SCHEDULING)
- Consent-click interaction duration (input delay / processing / presentation): <TODO>
- What the click starts: <TODO n requests, n KB, n ms main thread>
- Time from click to banner gone from screen: <TODO>
- Is the cascade the CMP itself or tags firing on the consent event?: <TODO>
- Verdict: <TODO FIX NOW / FIX LATER / NO ACTION>

### The consent gate (outranks every performance finding here)
- Non-essential cookies/storage written BEFORE consent today: <TODO list, or none>
- Same list after the proposed loading change: <TODO list> - gate <TODO PASSED / FAILED / UNPROVEN>
- With consent denied: analytics tags stay silent?: <TODO> - anonymised Consent Mode pings
  still sent?: <TODO yes / no / vendor does not support>

### Decisions
- Loading policy chosen: <TODO leave synchronous / defer / relocate into the tag manager>
- Latency paid back after relocation (preconnect / preload to the vendor origin): <TODO>
- Time-to-banner-visible, before vs after: <TODO / TODO>
- Off-screen policy applied?: <TODO yes - selector <TODO> / no - banner is not the LCP element>
- Honesty test (LCP improved AND time-to-banner-visible unchanged?): <TODO PASSED / REGRESSION>
- Layout shift caused by the banner (routed to the CLS stage): <TODO>
- Experiment result (baseline / defer / relocated / off-screen): <TODO SUPPORTED / REGRESSION /
  INCONCLUSIVE + Start Render, LCP with winning element, CLS and time-to-banner-visible per variant>

## INP / interactions
Filled by Stage 120 (audit/120-inp-interactions.md). Leave <TODO> until run.
- Viewport and throttling the lab numbers were measured on: <TODO e.g. 412x765x2.6 mobile, Fast 4G, CPU 4x>
- Is the CPU throttling representative of the device class RUM reports?: <TODO>

### The target (chosen from the field, never from the lab)
- Source of the target: <TODO RUM tool name / UNCONFIRMED IN THE FIELD - no RUM installed>
- Page type and URL: <TODO>
- Element (selector or label) and interaction type (pointer / keyboard): <TODO>
- How often users hit it: <TODO share of visitors, or unknown>
- When in the visit it happens (early, inside the load window, or later): <TODO>
- Why this one and not the others: <TODO frequency vs severity>

### Numbers
- Field INP (p75) and worst observed interaction: <TODO / TODO>
- Field subparts - input delay / processing time / presentation delay: <TODO / TODO / TODO>
- Attributed script domain reported by RUM: <TODO my domain / vendor / none>
- Lab reproduction, total duration: <TODO>
- Lab subparts - input delay / processing / presentation: <TODO / TODO / TODO>
- Did the lab reproduce the field result?: <TODO yes / no - and what state was missing>

### Attribution and route
- Whose code runs in that frame: <TODO my bundle / framework / tag manager / other vendor>
- Source location from the LOCAL run (file and line, not a bundle offset): <TODO>
- Dominant subpart: <TODO input delay / processing time / presentation delay>
- Owner implied by the subpart: <TODO this stage / Stage 60 or 80 (loading) / Stage 70 (recalc)>

### Decision
- Any work that should simply be deleted?: <TODO what, or none - this is the only fix here that
  genuinely reduces work>
- Named visual change that must land first: <TODO>
- Scheduling mechanism: <TODO setTimeout / scheduler.yield with a yieldToMain fallback / web
  worker / optimistic UI / none needed>
- What is now scheduled after the paint: <TODO>
- If a tag cascade: fixed in the container or in code?: <TODO> - events verified to still
  arrive (MPA and SPA)?: <TODO>
- Experiment result: <TODO SUPPORTED IN LAB / REGRESSION / INCONCLUSIVE> - medians and spread:
  <TODO>
- Time from click to visual change, before vs after: <TODO / TODO>
- Field confirmation due (re-read RUM after the change is live): <TODO date> - result: <TODO>

## JS runtime
Filled by Stage 130 (audit/130-js-runtime.md). Leave <TODO> until run.
- Page type these numbers came from: <TODO>
- Viewport and throttling: <TODO e.g. 412x765x2.6 mobile, Fast 4G, CPU 4x>

### The tail (measured with no scrolling and no clicking)
- Total Blocking Time: <TODO>
- Long tasks in the tail (duration and file): <TODO>
- How long the main thread stayed busy after Document Complete: <TODO>
- Last long task relative to LCP: <TODO>

### Owners
- Chunk / file: <TODO> - what it contains: <TODO libraries>
- What visually broke when the request was blocked: <TODO>
- Source location from the LOCAL run (not a bundle offset): <TODO>
- Unrelated libraries sharing one chunk?: <TODO what, or no>

### Triggers
- Trigger per component: <TODO module scope / DOMContentLoaded / observer / framework directive
  / hydration / repeating (scroll, rAF, timer)>
- Lazy appearance WITHOUT lazy code (appears on scroll, executes at startup): <TODO which, or
  none - and how it was proved>
- Does the user see this component on this page type?: <TODO share of visits that scroll to it>

### Hydration and re-renders (framework stacks; note if skipped and why)
- Hydration cost measured: <TODO / skipped - not a component-hydrated stack>
- What could be excluded from hydration (islands, suspense, static sections): <TODO>
- Re-renders observed with a highlighter, and which of them had to happen: <TODO>
- Deterministic repo scan run?: <TODO tool and headline result>
- Memoising compiler in play?: <TODO yes/no> - measured effect: <TODO>

### Decision
- Per candidate: <TODO delete / defer the code / defer the work / defer the rendering
  (content-visibility - rendering only, not JS) / stop the repetition / NO ACTION>
- Build-config change needed (chunk splitting)?: <TODO>
- Experiment result: <TODO IMPROVED / REGRESSION / INCONCLUSIVE> - medians and spread: <TODO>
- Four costs before vs after (bytes / parse+compile / execute / last long task): <TODO>
- Did a load-time cost move into an interaction-time cost?: <TODO checked how>

## Navigation
Filled by Stage 140 (audit/140-navigation-bfcache.md). Leave <TODO> until run.

### Field sizing (navigation types)
- Source: <TODO CrUX via which tool / UNCONFIRMED IN THE FIELD - no field data>
- navigate / navigate cache / reload: <TODO / TODO / TODO>
- back-forward: <TODO> vs back-forward cache: <TODO> - the gap: <TODO>
- prerender / restore: <TODO / TODO>
- Split by device (back/forward is mostly mobile): <TODO>

### bfcache
- Served from bfcache?: <TODO yes / no> - per page type: <TODO>
- Blockers named by the browser: <TODO verbatim>
- Attributed to: <TODO my code (file) / vendor script (which)>

### Analytics on restore
- What happens on a back/forward restore today: <TODO no event / duplicated event / correct>
- pageshow path present?: <TODO>
- How it was tested: <TODO>

### Policy
- What the project prefetches by default, and how many requests fire before any click: <TODO>
- Prefetch policy chosen: <TODO leave as is / on hover / key routes only>
- Speculation Rules present already?: <TODO none / platform default / custom>
- Policy chosen: <TODO prefetch or prerender, which routes, eagerness> - or <TODO none - SPA
  routing / NO ACTION>
- Consent and analytics verified for prerendered pages: <TODO>
- Field confirmation due: <TODO date> - result: <TODO>

## Layout stability
Filled by Stage 150 (audit/150-layout-stability.md). Leave <TODO> until run.

### Field sizing
- Source: <TODO CrUX via which tool / RUM / UNCONFIRMED IN THE FIELD - no field data>
- CLS p75 mobile: <TODO> - desktop: <TODO>
- Per page type: <TODO>
- Lab CLS on the same page: <TODO> - gap against the field: <TODO>

### Shifts during load
- Per shift: <TODO element (selector) | score | above/below the fold | timestamp>
- Unstyled-then-styled or absent-then-inserted?: <TODO per shift>
- Emulation used: <TODO device, network, CPU>

### Shifts during use (the pass the lab cannot run)
- Waiting without touching anything: <TODO what moved, when>
- On scroll: <TODO>
- On interaction (hadRecentInput true): <TODO what moved, and whether it lands under the user>
- On page transition: <TODO>

### Attribution
- Sizing present in the HTML the server sent?: <TODO per shift, checked in view-source>
- Technique that caused it: <TODO async CSS (60) / lazy media (90) / font swap (100) / consent
  (110) / layout decided in JS (130) / other>
- Owner: <TODO my template / theme / plugin or page builder / third-party tag / not identified>
- Recurs across page types?: <TODO>

### Verdicts
- Metric verdict: <TODO>
- UX verdict (shifts that score badly but serve the user, and shifts that score clean but
  disorient): <TODO>
- Automated clients affected (AI agents, e2e tests, scrapers)?: <TODO>

### Reservations decided
- Per shift: <TODO fix | file | stage that owns it>
- Routed back to another stage: <TODO which shifts, which stage>
- Local Overrides test result: <TODO IMPROVED / REGRESSION / INCONCLUSIVE> - medians and spread:
  <TODO>
- Use-phase pass re-run after the fix?: <TODO result>
- Field confirmation due: <TODO date> - result: <TODO>

## Progress
- [ ] Stage 10 - profile and baseline
- [ ] Stage 20 - network: DNS / TLS / HTTP
- [ ] Stage 30 - preconnect (external domains)
- [ ] Stage 40 - cache & CDN (edge)
- [ ] Stage 50 - backend triage (TTFB)
- [ ] Stage 60 - render start (first paint)
- [ ] Stage 70 - DOM size (recalc cost)
- [ ] Stage 80 - scripts and third party
- [ ] Stage 90 - images and video
- [ ] Stage 100 - fonts
- [ ] Stage 110 - consent / CMP
- [ ] Stage 120 - INP / interactions
- [ ] Stage 130 - JS runtime after load
- [ ] Stage 140 - navigation and bfcache
- [ ] Stage 150 - layout stability (CLS)
