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

## 3d. Stage 80 standalone: scripts and third party
Paste after workshop 11 (6.08). Needs the chrome-devtools MCP server, plus WebPageTest for
the SPOF check. If you use a tag manager, export its container config first - you will be
asked for it.
```
Run .ai/web-performance/audit/80-scripts-and-third-party.md for MY project.

Inputs: MY page URL and stack from site-profile.md, what Stage 60 found in ## Render start
and Stage 70 in ## DOM size, the chrome-devtools MCP server, and MY SOURCE CODE - the repo
you are running in. Show me each step before moving on.

Measure MY page, the URL from site-profile.md. Do not substitute any example or demo site.
If the URL, page types or stack are missing from the profile, ask me and wait.
The deliverable is not "you have too much JavaScript" - it is: THIS script, loaded THIS
way, from THIS file in my repo or from THIS vendor, costs THIS much main-thread time at
THIS point in the load, and here is the change to when it runs.

READ THIS FIRST, it decides whether the run is any good: weight is NOT the criterion.
A script and an image of the same size are not the same cost - the image decodes largely
off the main thread, the script must be transferred, parsed, compiled and executed, and
two of those four happen on the main thread. Never flag a script because of its transfer
size. The question is always: when does it execute, and what waits behind it?

1. INVENTORY. Call emulate ONCE (mobile 412x765x2.6, Fast 4G, CPU 4x) and never change it.
   CPU throttling is not optional here - without it, parse and compile are nearly invisible
   and you will conclude that JavaScript is free. Navigate, then performance_start_trace
   with reload and autoStop. Also curl the page and keep the raw server HTML.
   Give me one row per script: script | origin | 1st/3rd party | mode | position |
   transfer | in server HTML? Mode is sync / async / defer / module / module+async /
   injected, read from the actual tag in the server HTML - not from what the framework
   claims. "injected" means the tag is not in the server HTML at all; name its injector if
   the initiator chain shows it.

2. ATTRIBUTE. From the trace, give me main-thread time per script: parse/compile, execute,
   and whether it ran before or after First Paint. If a ThirdParties insight exists, call
   performance_analyze_insight on it and give me per-vendor main-thread time. Cross-check
   against the ForcedReflow caller Stage 70 already named - a script that both executes
   long and forces layout is the first candidate regardless of size.
   Run the trace 3 times and give me the spread; that is my noise floor.
   Then sort the table by MEASURED TIME, not by size, and tell me explicitly whether the
   two orders disagree. They usually do, and that is the point of this stage.

3. SPOF. Flag every third-party script in head WITHOUT async and WITHOUT defer - if the
   vendor is slow or down, the parser stops and my page may never render. Report these
   above any timing result. Then prove it: run a WebPageTest test with that domain
   blackholed (the SPOF tab - a different test from DevTools request blocking) and tell me
   what happened to Start Render.
   third-party-web is fine for prioritising and for arguing with stakeholders, but it is a
   population median - never write a number from it into my findings as if I had measured it.

4. COVERAGE - and read this before you conclude anything. Coverage reports bytes not
   executed UP TO THE MOMENT OF MEASUREMENT. It does not report dead code. A modal handler,
   a form validator, a carousel below the fold will all look ~100% unused because nobody
   has clicked anything yet. Deleting them does not optimise my page, it breaks it.
   Split every unused figure into three buckets and never give me the raw total as a
   saving: dead (not reachable on this page type - prove it by searching my repo), needed
   later (only after an interaction or scroll - this is the bucket we act on, by loading it
   later), needed now (stays). A Lighthouse "reduce unused JavaScript" number is a
   prediction, not a measurement - cite it as a hypothesis to test, never as a result.

5. TAG MANAGER, if I have one. Map the cascade: container fetched from a vendor origin ->
   it executes -> its tags fire -> tags inject further scripts with their own origins and
   connection costs -> those finally execute. Each level is serialised behind the previous
   one. Give me the timing per level. Scripts at the injected level will not be in my repo
   - say so instead of proposing an edit I cannot make.
   Then ASK ME to export the container config (Admin > Export Container, JSON) and WAIT.
   Analyse it: which tags exist, which triggers, which are duplicated, which never fired in
   the trace. Do not guess the container contents from the page.
   List every tag on an INTERACTION trigger (click, submit, phone/e-mail link, menu open).
   Those run inside the same task as the user's click, before the browser can paint - they
   never appear in a load trace. Tell me the load figure is a LOWER BOUND and route that
   list to the INP stage. Do not measure INP here.

6. PRIORITISE. Put script main-thread time (first vs third party) next to LCP, FCP and
   Start Render from Stage 60 and the Stage 70 verdict, and tell me honestly: FIX NOW,
   FIX LATER or NO ACTION. A SPOF finding means FIX NOW regardless of milliseconds.
   Show me the verdict and wait.

7. PROVE IT (only if we agreed on FIX NOW). Round 1: block the request (DevTools block by
   URL or domain, or the WPT equivalent), 3 rounds rotated A,B,A,B, discard the warm-up,
   compare medians against the spread.
   AND CHECK THE PAGE STILL WORKS while blocked. Blocking a script the layout depends on
   shows a beautiful improvement in a page that is now broken - content missing, half the
   DOM never built. A number from a broken render is not a result.
   Round 2, only if round 1 was SUPPORTED: test what I would actually ship, which is
   usually not deletion but a different loading strategy - sync to defer, a tag pushed
   behind first paint, a widget loaded on interaction. Build it as local HTML variants
   (download the HTML, inject <base href>, one edit per variant plus an unmodified
   baseline). Do NOT mutate the page at runtime: initScript does not survive the reload
   performance_start_trace performs, so you would prove my fix does nothing.
   Compare local variants against the LOCAL baseline only - a local copy loads a different
   set of third-party scripts than production.
   Verdicts: SUPPORTED, REGRESSION, INCONCLUSIVE.

8. DECIDE, one line per script, in this order: is it needed on this page at all (if not,
   remove - and prefer detecting the component over a per-URL exception list, which breaks
   the moment I edit a page); does it render something above the fold (stays early, defer,
   in head, after the CSS); does it have dependencies or require order (defer, NEVER async -
   async between dependent scripts is not a trade-off, it is a bug); is it needed only after
   an interaction or scroll (load on demand); is it third party (the levers are loading
   strategy, configuration or dropping the vendor - not a source edit; and warm the
   connection with preconnect even when you defer the script, but never to an origin we
   fetch nothing from).
   Route the fixes by my stack: WordPress - wp_enqueue_script strategy and
   wp_dequeue_script rather than editing plugins; Next.js - next/script afterInteractive or
   lazyOnload, next/dynamic, and check for barrel files defeating code splitting; Astro -
   client:visible / client:idle, and revisit every hydrated island.

9. FIX one thing at a time in the file you named, re-measure on the same emulation, report
   before/after. Last line, always: this measured the LOAD - tags firing on interaction
   cost extra and show up in INP, not here.
```

## 3e. Stage 90 standalone: images and video
Paste after workshop 12 (7.08). Needs the chrome-devtools MCP server. No WebPageTest
required, but a WPT run at the same profile is a useful cross-check on the LCP element.
```
Run .ai/web-performance/audit/90-images-and-video.md for MY project.

Inputs: MY page URL and stack from site-profile.md, what Stage 60 found in ## Render start,
Stage 30 in ## Preconnect, Stage 40 in ## Cache & CDN and Stage 80 in ## Scripts, the
chrome-devtools MCP server, and MY SOURCE CODE - the repo you are running in. Show me each
step before moving on.

Measure MY page, the URL from site-profile.md. Do not substitute any example or demo site.
If the URL, page types or stack are missing from the profile, ask me and wait.
The deliverable is not "your images are too heavy" - it is: THIS image, at THIS intrinsic
size, rendered at THIS size, in THIS format, discovered at THIS moment, from THIS file in
my repo, and here is the one change that matters most for it.

READ THIS FIRST, it decides whether the run is any good: the images on my page are TWO
populations, not one, and they are fixed by opposite means.
- The LCP element is a LATENCY problem: its cost is how late the browser learns it exists
  and what priority it then gets. Compression helps a little; discoverability and priority
  help a lot; lazy-loading it is a regression.
- Every other image is a BANDWIDTH problem: dimension, format, compression, deferral - and
  giving them the LCP treatment (eager, high priority, preloaded) makes my page WORSE,
  because they then compete with the one image that decides the metric.
Give me TWO separate verdicts at the end. One verdict for "the images" means you skipped
the analysis.
Second rule: the levers have a fixed order - dimension, then format, then compression, then
loading strategy. A 3000 px file in a 300 px box is a 10x error; re-encoding it is maybe a
30% one. If you propose a format change for an image that is grossly oversized, you skipped
a step - tell me so.

1. INVENTORY. Call emulate ONCE (mobile 412x765x2.6, Fast 4G, CPU 4x) and never change it.
   Viewport matters more here than anywhere earlier: srcset picks a different file per
   width and per DPR, so a desktop run audits files my mobile users never download. State
   the viewport next to every number.
   Navigate, then performance_start_trace with reload and autoStop. Also curl the page and
   keep the raw server HTML - an image in the DOM but not in the server HTML was injected
   by a script, cannot be found by the preload scanner, and usually cannot be fixed in my
   template.
   Then use evaluate_script to collect, per <img>: currentSrc (what srcset ACTUALLY picked,
   not the src attribute), naturalWidth/naturalHeight, the rendered box from
   getBoundingClientRect, devicePixelRatio, whether width and height attributes are both
   present, and the loading / fetchpriority / decoding / sizes / srcset attributes. Do the
   same for CSS backgrounds (getComputedStyle backgroundImage), <video> and <iframe> embeds
   - backgrounds will not appear in the img list and they are the ones most likely to be
   the problem.
   Give me one row per asset: asset | element type | origin | intrinsic px | rendered px x
   DPR | format | transfer | loading | fetchpriority | in server HTML?

2. THE LCP ELEMENT, on its own. Identify it from the trace, not by eye - if an LCP insight
   exists (LCPBreakdown / LCPDiscovery), call performance_analyze_insight and give me the
   phase split: TTFB, load delay, load time, render delay, median of 3 with the spread.
   Re-confirm it at MY mobile viewport; the desktop answer is frequently a different
   element.
   Then classify how it is delivered, because the fix follows entirely from this: <img> in
   the server HTML (the scanner finds it early), <img> injected by a script (the scanner
   never sees it - discovery waits for that script), CSS background-image (the URL is
   inside a stylesheet, so the browser needs the CSS, the CSSOM, rule matching and layout
   before it knows the image exists - this is the most common reason an LCP image starts
   late, and no amount of compression fixes it), or a carousel slide (add the carousel's
   JavaScript to the chain).
   Read the phase split and tell me which problem this is: load delay dominates = DISCOVERY
   (make it findable, plus priority); load time dominates = TRANSFER (dimension and format);
   render delay dominates = not an image problem, route it back to Stage 60 or 80.
   Check on this one element: is it loading="lazy" (a direct regression - report it as the
   finding), what priority did it get in the network list, and is it cross-origin without a
   preconnect from Stage 30?

3. DIMENSION - the first lever, for everything EXCEPT the LCP element. Compute intrinsic px
   divided by rendered px x DPR. A ratio above 2 in width means at least 4x the pixels
   needed, since area scales with the square. Sort by WASTED PIXELS, not by file size.
   For each oversized image tell me WHY, because the fix differs: no srcset at all; srcset
   present but sizes wrong or missing (without sizes the browser assumes 100vw and a
   one-third-width column gets a file three times too wide - this is the most common cause
   and it is invisible unless you compare currentSrc against the rendered box); or the
   variants that exist do not match the layout.
   Note where sizes="auto" would help - it only works on images that are also
   loading="lazy", so it is a fix for the long tail, never for the hero.

4. FORMAT AND COMPRESSION - second and third levers, only for images already near the right
   dimension. Record the current format, whether a modern one is served at all, and whether
   it is negotiated per browser (picture + source type, or an image CDN) or hardcoded with
   no fallback. Do not quote a generic percentage saving - the figure varies enormously by
   image content; measure my actual files if you propose a change.
   Then ASK ME which optimisation moment my project uses and WAIT: upload time (converted
   and resized when the file enters the system - cheap at runtime, but only applies to media
   uploaded after the change, so existing files need regeneration) or delivery time (one
   master, an image CDN transforms per request and caches at the edge - retroactive and
   flexible, but adds a third-party origin with its own connection cost, and the first
   request for each variant is slow). Usually both is the right answer.
   And say it plainly: an image CDN WITHOUT correct srcset and sizes does almost nothing -
   it serves one large transformed file to every device.

5. LOADING STRATEGY. Above the fold but not the LCP: eager, ordinary priority - do NOT mark
   these high. Priority is zero-sum and every high-priority image competes with the LCP
   image, the CSS and the JavaScript. Flag any page marking several images high.
   Below the fold: loading="lazy", decoding="async" as hygiene (a hint with a modest effect
   - never a headline finding).
   Hidden until an interaction (mega-menu, modal, closed tab): frequently loaded eagerly
   because the markup is merely hidden - prefer not putting it in the document at all.
   CSS backgrounds below the fold: there is no native lazy loading for background-image.
   Either convert to <img> (which also gets alt text and native lazy) or apply the
   background from an IntersectionObserver. Say which fits the element.
   CSS background AS the LCP element: make it an <img> if the design allows; if it cannot
   move, preload it with fetchpriority="high" and use imagesrcset/imagesizes so the
   preloaded file matches what the CSS will request, otherwise we download two. Tell me
   honestly that a preloaded background still typically paints later than the same file as
   an <img>, because rendering still waits on the CSSOM and layout.
   iframe embeds: loading="lazy" is supported but browsers apply their own heuristics and
   often load them early - verify in the waterfall instead of assuming.
   Missing width/height: record every one, and check the computed styles too (CSS can
   discard the reserved box). This is a CLS finding - route it, do not measure it here.

6. SVG AND BASE64. Per SVG: inline, external <img>, or sprite. State the trade-off for MY
   page - inline costs no request and paints immediately but adds its weight to EVERY HTML
   response and cannot be cached separately; external is cacheable at one request; a sprite
   is one request for many icons but a project-wide sprite can be large enough to compete
   with the CSS and JavaScript in the initial burst. Give me the document weight
   contributed by inline SVG - if my HTML is unusually large, this is frequently why.
   Run the SVGs through SVGO and give me before/after; design-tool exports routinely carry
   several times the markup they need.
   Flag every data URI. base64 inflates the payload by roughly a third, moves the bytes
   into a document that is usually not cached, and forfeits caching of the asset - and the
   "fewer requests" argument it rests on is largely obsolete over HTTP/2 and HTTP/3. Give
   me the total base64 payload in the document.

7. VIDEO, if any. Self-hosted <video>: duration, transfer, resolution and the preload value
   - preload="auto" on a below-the-fold video buffers it during the initial page load,
   alongside the CSS and the JavaScript. Is a poster set? Read from the trace what the LCP
   candidate actually is on a page with a hero video; do not assert it.
   Flag every animated GIF: it is a sequence of full frames with no modern compression, and
   the same clip as MP4 or WebM is typically an order of magnitude smaller. The replacement
   is <video autoplay muted loop playsinline> - muted is what makes autoplay permitted at
   all, playsinline is what stops iOS going fullscreen.
   Per-device variants: there is no native responsive selection for video. Report the
   current state; do not build the JavaScript solution here.
   YouTube / Vimeo: these are not one file, they are a player - frequently megabytes of
   JavaScript and CSS before anything plays. Give me the transfer and main-thread cost, and
   propose a facade (our own thumbnail plus play button, real embed inserted on click;
   lite-youtube-embed or @next/third-parties do this ready-made). Say plainly that this
   changes nothing for someone who plays the video and saves everything for the majority
   who do not.

8. PRIORITISE - TWO verdicts, separately, and then WAIT.
   A) The LCP element: what it is, how delivered, at which viewport, the phase split with
   the spread, which phase dominates, whether it is lazy/low-priority/cross-origin, then
   FIX NOW / FIX LATER / NO ACTION plus the single highest-value change.
   B) Everything else: total image transfer, how much is wasted pixels, the worst 3 with
   causes, SVG and base64 payload, video findings, and how all of it compares to what Stage
   60 and Stage 80 found - is media a visible share of this page's problem, or noise next
   to a 2-second TTFB? Then FIX NOW / FIX LATER / NO ACTION.
   The two can disagree and frequently do. Say so.

9. PROVE IT (only if we agreed on FIX NOW). Normally the LCP element's discovery or
   priority, since that is where the leverage is. Build local HTML variants (download the
   page, inject <base href>, one edit per variant plus an unmodified baseline), serve
   locally, same emulation, 3 rounds rotated A,B,A,B, discard the warm-up, compare medians
   against the larger spread. Do NOT mutate the page at runtime - initScript does not
   survive the reload performance_start_trace performs. Compare local against LOCAL
   baseline only.
   AND CHECK WHAT YOU ARE MEASURING: removing an image improves LCP by making the page
   emptier. If the variant does not paint the same content, the number is not a result.
   Expect asymmetry and report it honestly - a discovery fix on the LCP element often moves
   LCP visibly, while resizing twenty below-the-fold images moves no headline metric at all
   and simply saves transfer. Both are legitimate; do not inflate the second into an LCP win.

10. DECIDE, one line per asset, levers in order: resize to what the layout needs with real
    srcset and sizes; convert with a working fallback; compress to the lowest quality that
    still looks right on MY images; defer everything below the fold while keeping the LCP
    element eager, discoverable and high priority; warm the connection for an external image
    origin, and reconsider serving the LCP image from my own origin so the first visit does
    not pay DNS + TCP + TLS before the most important byte on the page.
    Route by my stack: WordPress - add_image_size plus thumbnail regeneration (adding the
    size alone does nothing for existing media), and check what the page builder emits,
    including its "exclude the first N images from lazy loading" setting, which counts DOM
    order and can be spent on a logo and two icons before it reaches the real hero - verify
    against the element the trace named. Next.js - the sizes prop must describe the real
    layout or the whole benefit is lost, the priority prop emits a preload so it belongs on
    the LCP element only, and at scale on-the-fly optimisation is CPU and memory on MY
    server (connect that to Stage 50 if the origin is under pressure). Astro - Image/Picture
    from astro:assets with explicit widths, formats and sizes, and check whether remote
    images are being passed through unprocessed.

11. FIX one thing at a time in the file you named, re-measure on the same emulation, report
    before/after. Last line, always: these numbers are for <viewport> - srcset picks
    different files at other widths, and missing width/height was recorded, not measured,
    because that belongs to the CLS stage.
```

## 3f. Stage 100 standalone: fonts
Paste after workshop 13 (11.08). Needs the chrome-devtools MCP server. Run Stage 90 first
if you can - whether the LCP element is text or an image decides this whole stage.
```
Run .ai/web-performance/audit/100-fonts.md for MY project.

Inputs: MY page URL and stack from site-profile.md, what Stage 90 found in ## Media (is the
LCP element text or an image?), Stage 60 in ## Render start, Stage 30 in ## Preconnect,
Stage 40 in ## Cache & CDN and Stage 80 in ## Scripts, the chrome-devtools MCP server, and
MY SOURCE CODE - the repo you are running in. Show me each step before moving on.

Measure MY page, the URL from site-profile.md. Do not substitute any example or demo site.
If the URL, page types or stack are missing from the profile, ask me and wait.
The deliverable is not "self-host your fonts and use font-display: swap" - it is: THIS font
file, at THIS weight, discovered after THIS many hops, needed (or not) by THIS element above
the fold, from THIS declaration in my repo, and the one policy that applies to it.

READ THIS FIRST, it decides whether the run is any good.
A font is not a file, it is the END OF A CHAIN: HTML -> CSS -> possibly another CSS via
@import -> CSSOM -> a selector matched to a real element -> only now does the file enter the
network queue. The cost lives in the hops, not in the kilobytes. And @font-face declarations
are NOT downloads: fifteen declarations can be zero requests.
Second, and this is the rule that makes or breaks the stage: PRELOAD IS ZERO-SUM. It does
not make a font faster in isolation, it moves that font ahead of the CSS, the JavaScript and
the LCP image on the same connection. You must output a NUMBER - how many font files may be
preloaded on this page type - and it is normally 0 or 1, occasionally 2. If you propose more
than two, justify it with a measurement or withdraw it.
Third: while a font is in flight the page must either hide text (block/auto: up to ~3 s of
invisible text) or show a fallback and then shift (swap). There is no free option. Tell me
which cost this project is CHOOSING - do not recommend swap by reflex - and if it is swap,
metric matching is the price of that choice, not an extra.

1. INVENTORY. Call emulate ONCE (mobile 412x765x2.6, Fast 4G, CPU 4x) and never change it.
   Navigate, then performance_start_trace with reload and autoStop. Also curl the page and
   keep the raw server HTML.
   Collect TWO things with evaluate_script, because the gap between them is the finding:
   (a) what the browser actually loaded - iterate document.fonts and keep the faces with
   status "loaded", with family, weight, style, unicodeRange and display; (b) what the page
   above the fold actually renders in - walk the elements inside the initial viewport that
   contain text, read the computed font-family, weight and style, and count elements per
   face. The WebPerf Snippets snippet fonts-preloaded-loaded-and-used-above-the-fold does
   exactly this cross-reference including preload status and font-display; you may paste it
   through evaluate_script instead. Say where the numbers came from.
   Cross-reference the trace so every face maps to a real file, transfer size and origin.
   Give me one row per FILE, not per declaration: file | family/weight/style | origin |
   format | transfer | unicode-range | font-display | preloaded? | used above the fold? |
   elements using it. Flag immediately: faces loaded but used by zero elements, and any face
   used above the fold that is NOT preloaded while some other face IS.

2. THE CHAIN, per file. Reconstruct what had to happen before each request started and read
   the start time from the waterfall. Classify each file into exactly one chain: A =
   @font-face inline in the document AND the file preloaded; B = inline, no preload; C =
   @font-face in an external stylesheet on my origin; D = in a third-party stylesheet (the
   standard Google Fonts embed: DNS + TCP + TLS, then the stylesheet, and only then does the
   browser learn the file URLs, which live on yet another origin); E = the stylesheet is
   reached through @import from another stylesheet, so two serialised downloads happen
   before the @font-face is even parsed - the deepest chain in common use and invisible in
   the HTML; F = the stylesheet is injected by JavaScript, so add that script's download and
   execution. Record chain letter, hop count, request start, and the delta from Start
   Render. Sort by hops, not by transfer size. A file whose request starts AFTER Start
   Render is not delaying the first paint - its cost is a repaint and a shift, so do not
   report it as a render-start problem.

3. WHICH FACE IS CRITICAL. From (b) above and from Stage 90: which single face renders the
   LCP text? If Stage 90 says the LCP element is an image, say so - the budget probably
   belongs to that image. Put every loaded face in one of three tiers: critical (renders the
   LCP element or the primary above-fold copy), above the fold but secondary, and below the
   fold / interaction-only / icon font. Check the failure mode that appears in almost every
   page builder: a theme preloading one weight globally while individual templates render a
   different weight, so on most page types the budget is spent on a file the page never uses.
   Verify per page type, not once for the site.

4. THE BUDGET. State N for this page type and which file gets it. Then list every currently
   preloaded file as KEEP or REMOVE. Where preload stays, verify it is correct, because a
   wrong preload is worse than none: as="font", type="font/woff2", crossorigin present (font
   requests are CORS-mode even same-origin - without it the file is fetched TWICE, check the
   waterfall for the duplicate), and the preloaded URL byte-identical to the URL @font-face
   will request (watch cache-busting query strings and CDN rewrites). Say plainly what
   preload does not do: it does not shorten the chain for the other faces, does not change
   font-display, and does not make a third-party origin local.

5. THE TRADE-OFF. Record font-display per face and whether it was chosen or inherited from a
   provider default - auto is an unmade decision, not a setting, and behaves like block.
   Then MEASURE the swap: find the layout shift events in the trace and check whether their
   timestamps coincide with the font finishing loading. Give me the shift value and the
   moment. If the choice is swap, check whether a metric-matched fallback exists
   (size-adjust, ascent-override, descent-override, line-gap-override) and verify it in the
   computed styles, not just in the CSS - swap with no metric matching is an accepted layout
   shift that nobody decided to accept. Note where the framework already generates these so
   we do not do the work twice.

6. BYTES, in this order and only now: cut faces nothing renders and weights the design does
   not use; compare a variable font against the actual sum of my static files (often smaller,
   not automatically); check subsetting and unicode-range against MY content - for Polish
   that is latin + latin-ext and a subset that drops the diacritics is a bug, not an
   optimisation; WOFF2 or it is a finding. Ask me whether my faces are licensed for
   modification BEFORE proposing a subset. If an icon font is present, count how many of its
   glyphs the page actually uses.

7. ORIGIN AND CACHE. Self-host unless there is a reason not to: a third-party origin costs
   DNS + TCP + TLS on the first visit before the browser even knows which files it needs, and
   the cache is not shared across sites. If it is Google Fonts and the template cannot change,
   record Cloudflare Fonts as the mitigation - and record it accurately: it moves the origin,
   it does not reduce my declared faces, does not subset, does not choose the preload. Fonts
   are immutable, so a long TTL (a year) is correct; with a third-party provider the TTL is
   theirs, which is itself an argument for self-hosting.

8. VERDICT, then stop and show me: deepest chain and the file at the end of it; the critical
   face; the budget N and who gets it; the cost being chosen and the measured shift; total
   font transfer, faces, and how many are unused above the fold; and how all of that compares
   to what Stages 60, 80 and 90 found - is the font chain a visible share of this page's
   problem or noise next to a 2-second TTFB. FIX NOW / FIX LATER / NO ACTION. Wait for me.

9. EXPERIMENT, only on FIX NOW. Build local HTML variants as in Stage 60.5 (download the
   page, inject <base href>, one edit per variant, plus an untouched baseline) - do NOT use
   initScript, it does not survive the reload the trace performs. Run at least three:
   baseline; preload the critical face ONLY; preload EVERY face. The third is not padding -
   it is the control that demonstrates preload is zero-sum, and on a page with several faces
   it frequently measures worse than the baseline. 3 rounds rotated A,B,C,A,B,C, first load
   discarded, medians compared against the larger spread. Record FCP, Start Render, LCP AND
   CLS for every variant - a change that improves LCP while adding a shift has not
   necessarily won, and this is the stage where that happens. SUPPORTED / REGRESSION /
   INCONCLUSIVE.

10. POLICY PER FILE: preload / eager / defer / drop, plus font-display, in this order - cut
    unused faces, shorten the chain (no @import, first-party origin, @font-face reachable
    early), spend the budget on exactly one file and spend it correctly, choose the cost and
    pay for swap with metric matching, then bytes, then cache.
    Route by my stack: WordPress - check the theme and builder settings FIRST (native
    "host Google Fonts locally" and a font-display setting beat writing code); builders
    preload a fixed set of faces globally, so when the LCP heading uses a different weight per
    template the budget is being spent on the wrong file, and the reliable fix is a small
    must-use plugin emitting the preload for the face THIS template renders - verify the
    emitted <link> in the served HTML, not the code. Next.js - next/font self-hosts at build
    time and generates the metric-matched fallback, so confirm it is actually in use rather
    than a raw <link> to a third party, and check what it preloaded against the budget,
    because several font loaders preload several faces. Astro - fonts are plain static assets
    and everything here is explicit and under my control; confirm the preload link and the
    @font-face reference the identical URL after the build hashes it.

11. FIX one thing at a time in the file you named, re-measure on the same emulation, report
    before/after including CLS. Last line, always: these numbers are for <viewport>, and the
    swap shift I measured belongs in the CLS budget for the CLS stage.
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
