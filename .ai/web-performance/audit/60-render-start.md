# Stage 60: Render start - find what delays the first paint, then PROVE it by experiment

> Can be run STANDALONE or from the index (.ai/web-performance/audit/00-index.md).
> Behaviour: DO stage. Unlike Stages 10-40 you do not read a value, and unlike Stage 50
> you do not stop at routing - you form falsifiable hypotheses and settle them with a
> controlled A/B measurement. Read domain/URLs/stack from site-profile.md; if missing,
> ASK me and WAIT - do not guess.
> Input:  site-profile.md > ## Project, ## Page types, ## Backend (Stage 50 class)
>         + a WebPageTest JSON export in .ai/web-performance/wpt/mobile/
> Output: site-profile.md > ## Render start + findings.md, ending with a verdict table
>         and a stack-routed fix list. Runs AFTER Stage 50.
> Idempotent: if ## Render start exists, update it, do not create a second one.

SCOPE AND HARD BOUNDARY (read first - IMPORTANT):
This stage answers ONE question: what delays Start Render, and what is the proof?

The boundary you learned in Stage 50 was never "the audit does not act" - it was
"the audit does not touch the project". That still holds here, and it is why the
experiment runs on a THROWAWAY LOCAL COPY of the page HTML. You never modify the live
site, never modify the repo, never deploy anything to test a hypothesis. The fix itself
is a separate step, after the verdict.

Second boundary, new in this stage: this is the first stage that can be CONFIDENTLY
WRONG. Stages 10-40 either read a value or they do not. Stage 50 ends in a class that a
human verifies with a tool. Here you produce a number that looks like a result, and a
sloppy measurement produces a number that looks like a result but is not one. So:
- a difference smaller than the run-to-run spread is INCONCLUSIVE, never a win;
- a Lighthouse or DevTools "estimated savings" is a PREDICTION, never report it as a
  measurement;
- if you cannot run the experiment (page behind auth/VPN, no browser available), say so
  and stop at hypotheses. Do not substitute an estimate for evidence.

## 60.0 Read what earlier stages already found (Do, no new measurement yet)
- From site-profile.md > ## Backend: the bottleneck class from Stage 50.
- GATE: if TTFB is the wall (class A/C/D, origin TTFB in seconds), say so and send me
  back to Stage 50. There is no point shaving the first paint while the HTML arrives
  after 3 s. Continue only when TTFB is acceptable but Start Render is not.
- From findings.md: Start Render / FCP / LCP per page type from Stage 10.
- From site-profile.md > ## Preconnect and ## Cache & CDN: which external origins are
  already known, and whether HTML is edge-cached.

## 60.1 Facts from the WebPageTest JSON (Do)
- Ask for (only if absent): a WPT JSON export for the slow page type, dropped into
  .ai/web-performance/wpt/mobile/. Export -> JSON on the WPT result page, whole file.
- The file is ~1 MB. Do NOT read it into context. Write a throwaway script that prints:
  - metrics: TTFB, Start Render, FCP, LCP, TBT, Speed Index, domInteractive,
    domContentLoaded, fullyLoaded, bytesIn, request count, unique host count;
  - RUN CONSISTENCY: the same metrics for every run in the file. If the runs disagree
    wildly, say so - you are about to reason on top of noise;
  - every request that started BEFORE Start Render, with type, priority, protocol,
    status, host;
  - the render-blocking lists (confirmed blocking + the "possibly render blocking" list);
  - EVERY consoleLog entry, with its timestamp converted to ms from navigation start;
  - network gaps longer than 500 ms, and for each: what was still in flight during it;
  - TBT per origin, fonts (family, font-display, loaded vs declared), LCP candidates
    over time with the element that won;
  - responses that returned zero bytes.
- Write the raw numbers to findings.md. Facts only, no interpretation yet.

## 60.2 The "what is NOT here" pass (Do - the habit this stage teaches)
- Cross the console errors and the render-blocking list against the request list.
- Anything that is render-blocking or errored but has NO normal row in the waterfall is
  a prime suspect: a resource that never arrives cannot be drawn as a bar, so it is
  invisible to the eye and expensive to the browser.
- Typical shapes: a host that does not resolve, a 404 on a stylesheet, an origin that
  accepts the connection and never answers, a CDN that was decommissioned.
- Note explicitly: the COST of such a resource is set by the DNS resolver and the
  timeout, not by the page. It can be seconds on one network and milliseconds on
  another. Do not promise a fixed gain for removing it.

## 60.3 Resource-by-resource interrogation (Do - the completeness pass)
Hypotheses catch the big offenders. This step catches the long tail, and it is the part
a human reviewer will actually recognise as "an audit". Do it BEFORE forming hypotheses -
the hypotheses should fall out of this table, not the other way round.

Take EVERY resource that started before Start Render / FCP and answer five questions:

| # | question | where the answer comes from |
|---|---|---|
| 1 | What is it and who added it? | `initiator` + `initiator_type` in the WPT JSON: parser-inserted vs injected by another script (and by WHICH one). Name the owner: framework, theme, page builder, CMP, analytics, ads, captcha, chat, fonts. |
| 2 | Is it needed ON THIS PAGE TYPE? | a captcha on a page with no form, a slider library on a page with no slider, a second tag manager container, a library the site stopped using two redesigns ago |
| 3 | Is it needed BEFORE the first paint? | only the CSS for above-the-fold content, the fonts actually used up there, and the LCP image are. Almost nothing else is. |
| 4 | Is it loaded the right way? | sync / `defer` / `async` / `type=module` / preloaded / render-blocking. Compare against what the resource actually does. |
| 5 | How big is it, and how much CPU does it burn? | `b_in` for bytes and `cpu_t` / `cpu_eval_script` for main-thread cost. A 3 kB script that evaluates for 400 ms is worse than a 200 kB image. |

Then assign each resource ONE verdict from this ladder, in this order of preference:

1. **DELETE** - nothing on this page uses it. The cheapest resource is the one you never request.
2. **DEFER / ASYNC** - it is used, but nothing before the first paint depends on it.
3. **MOVE** - it must stay parser-inserted, but it does not belong in `<head>`.
4. **ON DEMAND** - it is needed, but only after an interaction or when its section is
   reached: a captcha on form focus, a chat widget on click, a map on scroll into view,
   anything below the fold via `IntersectionObserver` or `requestIdleCallback`.
5. **SELF-HOST** - it must load early and cannot be deferred, so at least take the
   third-party origin (DNS + TCP + TLS, and someone else's uptime) off the critical path.
6. **KEEP** - genuinely required for the first paint. Say why.

Precision about the ladder (this is where people get it wrong):
- A `<script src>` in `<head>` with NO attribute blocks the parser at that point, so it
  blocks the first paint.
- The same script moved to the end of `<body>` no longer blocks the paint of what is
  above it, but it still blocks DOMContentLoaded and delays interactivity.
- `defer` is usually the better move than relocating: it stops blocking the parser
  wherever it sits, keeps execution order, and runs before DOMContentLoaded. Once a
  script has `defer`, its position in the document no longer matters for blocking.
- `async` suits independent, order-insensitive trackers, but it executes the moment it
  arrives, so it can still land mid-parse and delay the paint.
- So the usual answer to "should I move this head script into the body?" is: give it
  `defer` first, and only relocate if something forces it to stay parser-blocking.

### The step in the waterfall
If the waterfall shows a clear STEP - a batch of requests that all start noticeably later
than the ones above them - do not walk past it. A step is a DISCOVERY BOUNDARY: everything
above it was found by the preload scanner while parsing `<head>`, everything below it was
only discovered once the parser got past whatever was holding it.

So the step asks a specific question: what was blocking the parser at that exact moment?
Usually one of these:
- a parser-blocking `<script>` in `<head>` that had to download and execute first,
- a render-blocking stylesheet the browser was waiting on,
- a resource that never answered (see 60.2), which is the expensive version,
- the end of `<head>` itself, with the rest simply living in `<body>`.

Whatever it is, the resources ABOVE the step are your critical path and the ones worth
interrogating hardest. Record where the step falls and what caused it - it is usually the
single most informative feature of the whole chart.

Write the table to findings.md. Any resource whose verdict is not KEEP is a candidate for
60.4; the three that promise the largest gain become the hypotheses you actually measure.
The rest go into the fix list as low-risk cleanups that do not need an experiment each.

## 60.4 Hypotheses (Do - then STOP and show me)
- State AT MOST 3, ranked by expected gain in Start Render. For each:
  - mechanism: why this delays the first paint (blocking CSS / parser-blocking script /
    a resource nobody is waiting on but everybody queues behind);
  - expected gain in ms, and where that number comes from;
  - THE FALSIFYING EDIT: the single change to the HTML that would prove it wrong.
- Show me the three hypotheses and wait. Do not propose fixes yet, and do not start
  measuring until I agree they are the right three.

## 60.5 The experiment (Do - this is the evidence)
Build the variants:
- Download the page HTML once. Inject `<base href="{PAGE_URL}">` into `<head>` so every
  relative path still resolves against production - without it the copy 404s on its own
  assets and you are measuring a different page.
- Produce one file per hypothesis, each differing from the baseline by EXACTLY ONE edit,
  plus the unmodified baseline, plus optionally one "all fixes" variant.
- BEFORE measuring, verify each variant actually differs from the baseline and that
  your edit matched something. A pattern that matched nothing produces a file identical
  to the baseline, and identical files always report "no difference" - which is not the
  same as "no effect". If it did not match, fix the pattern; never lower the bar.
- Serve the files locally (any static server).

Measure them - drive the browser through the chrome-devtools MCP server:
- Call `emulate` ONCE (mobile viewport ~412x765, "Fast 4G", CPU throttling 4x) and never
  change it again for the rest of the stage. Changing the profile mid-experiment
  invalidates every comparison made before it.
- Every single measurement is TWO navigations, in this order:
  1. `navigate_page` type=`url` -> the variant's local URL,
  2. `navigate_page` type=`reload` with `ignoreCache: true` -> THIS is the measured load.
  `ignoreCache` is honoured only on a reload. On a url navigation it is silently
  ignored (measured: 62% of resources still served from cache, TTFB 1 ms, FCP 720 ms;
  the same page reloaded with ignoreCache: 3% cached, TTFB 24 ms, FCP 1776 ms). Nothing
  warns you - you just get a number roughly half the real one.
- Why cache-cold is not optional: with a warm cache the blocking resources are served
  from cache, their cost drops to zero, and every variant converges to the same number.
  An experiment that cannot see the effect is not evidence that the effect is absent.
- Load once as a WARM-UP and throw that result away.
- Then rotate the variants - A, B, C, A, B, C - for at least 3 rounds. Never in blocks:
  a block bakes the warming curve into whichever variant went first.
- After each load, `evaluate_script` to collect FCP, LCP and domContentLoaded from the
  Performance API, and in the SAME script PROVE the load was cold: count resources with
  `transferSize === 0` and a non-zero `decodedBodySize`. Above roughly 20% of all
  resources the load came from cache - the measurement is void, discard it and repeat.
- Budget note: 3 MCP calls per measurement, so 4 variants x 3 rounds is over 40 steps.
  If time is short, run the full protocol on ONE hypothesis rather than a sloppy
  protocol on three.

## 60.6 Verdict (Do - the deliverable)
- Compare MEDIANS, never single runs. Report the spread (max - min) within each variant.
- The bar per metric is the larger of the two spreads being compared. Verdicts:
  - SUPPORTED - improvement beyond the bar,
  - REGRESSION - the change made it worse beyond the bar,
  - INCONCLUSIVE - the difference is inside the noise in THIS environment.
- Table: hypothesis | mechanism | verdict | measured delta (FCP / LCP / DCL) | caveat.
- Write it to findings.md with the raw per-run numbers, not just the medians.

## 60.7 OUTPUT - "Render-start fixes", ROUTED BY STACK (Do)
Only for SUPPORTED hypotheses, plus every non-KEEP verdict from 60.3 and any
broken resource from 60.2 (see the note below).
For each: which file to edit, what to change, and what to re-measure. Do NOT edit yet.

Next.js:
- Third-party tags live in `pages/_document.tsx` or `app/layout.tsx` - a sync `<script>`
  there blocks the parser on EVERY route, including the ones that never use it.
  Move to `next/script` with `strategy="lazyOnload"` or `"worker"`, or load it on the
  interaction that needs it (a form field focus for a captcha, a click for a chat widget).
- Google Fonts `<link>` -> `next/font` (self-hosted, `display: swap`, no extra origin).
  Then delete the now-useless `preconnect` to those origins.
- LCP image: `priority` / `fetchPriority="high"` AND no `loading="lazy"` on the same
  element - the two together are contradictory instructions.

WordPress:
- Prefer an mu-plugin over editing the theme, so the change survives updates:
  `style_loader_tag` / `script_loader_tag` filters to add `defer`/`media` tricks,
  `wp_dequeue_style` / `wp_dequeue_script` for assets a page does not use.
- Hardcoded tags in `header.php` (or the page builder's "custom code" box) are the usual
  home of a dead CDN link - grep the theme and the builder settings, not just the plugins.
- Page builders enqueue globally by default; dequeue per page type where the widget is
  not used.

Astro:
- Head tags live in `src/layouts/*.astro`. Astro ships no JS by default, so a blocking
  third-party tag here is almost always hand-added - remove or defer it.
- Use `client:idle` / `client:visible` instead of `client:load` for anything below the
  fold; use `<script>` without `is:inline` so Astro can process and defer it.

Any stack - the three moves, in this order:
1. DELETE what nothing uses (dead hosts, unused libraries, duplicate tag managers).
2. DEFER what is used but not needed for the first paint.
3. SELF-HOST what must stay, to remove a third-party origin from the critical path.

Note on broken resources: if 60.2 found a dead host or a 404 and the experiment came
back INCONCLUSIVE, still remove it - but label it a CORRECTNESS fix, not a performance
fix, and record that its cost depends on the resolver. It measured near zero here and
measured seconds on the WebPageTest agent. Both numbers are true.

## 60.8 Known traps (verified - do not rediscover them the expensive way)
Every one of these produces a measurement that LOOKS fine and is wrong.

- `ignoreCache` works on `reload`, not on a `url` navigation. See 60.5 for the numbers.
- You cannot remove a tag at runtime. A script injected at document start does delete
  the `<link>` from the DOM, but the preload scanner already sent the request - verified:
  zero matching elements left in the DOM, resource still present in Resource Timing.
  This is why variants are built by editing the HTML, not by scripting the live page.
- A stylesheet injected by JavaScript does NOT block rendering unless it carries
  `blocking="render"`. Chrome only render-blocks parser-inserted stylesheets. So you
  cannot reproduce a blocking resource by adding a link from the console.
- The console can come back EMPTY after a traced reload even when the same errors sit in
  the WebPageTest JSON. Trust the network list (a request with status 0 or protocol
  "unknown") over the console.
- `<base href>` in the local copy is mandatory. Without it every relative path 404s and
  you are measuring a page with no CSS and no JS.
- Check the attribute before claiming a script blocks the parser. A `defer` script blocks
  DOMContentLoaded, not the first paint - a real effect, but a different one.
- The first load after a change of page or profile is always the slowest. If you compare
  a first load of one variant against a later load of another, you will "prove" a gain of
  several seconds that does not exist.

## Save results
- site-profile.md > ## Render start: Start Render / FCP / LCP baseline, the render-blocking
  list, any dead or errored resources, the three hypotheses with verdicts, and the routed
  fix list.
- findings.md: dated entry with per-run numbers, medians, spreads, and the reasoning.

## STOP line (end every run with this, verbatim intent)
"Render-start triage done. Proven: <hypotheses with deltas>. Not proven: <hypotheses>.
Next: apply the routed fixes in <file>, then RE-RUN WEBPAGETEST with the same profile.
This experiment measured with warm DNS and warm connections; only a WPT run starts cold
on every load, the way a real first-time visitor does. The local delta is the mechanism,
the WPT delta is the gain."

## At the end
Check off in site-profile.md > ## Progress: [x] Stage 60 - render start. Report the
verdict table and the single next action - nothing more.
