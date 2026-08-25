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
Paste after workshop 13 (11.08). Needs the chrome-devtools MCP server. Install the WebPerf
Snippets skill (webperf / webperf-loading) first if you have not - the stage uses it for the
inventory. Run Stage 90 first if you can - whether the LCP element is text or an image
decides this whole stage.
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
   Do NOT hand-roll the inventory. If the WebPerf Snippets skill (webperf / webperf-loading,
   nucliweb) is installed, use it - it ships the snippet locally, version-pinned, and its
   "Font Loading Optimization" workflow runs the three I want: Fonts-Preloaded-Loaded-and-
   used-above-the-fold (the audit), Resource-Hints-Validation (preload correctness, flags a
   font preload without crossorigin) and Find-render-blocking-resources (is the stylesheet
   carrying the @font-face itself render-blocking). If the skill is not installed, take the
   snippet from webperf-snippets.nucliweb.net, section Loading, "Fonts Preloaded, Loaded, and
   Used Above The Fold", and run it through evaluate_script. Only if you can do neither,
   collect it yourself: (a) iterate document.fonts and keep the faces with status "loaded",
   with family, weight, style, unicodeRange and display; (b) walk the elements inside the
   initial viewport that contain text, read the computed font-family, weight and style, and
   count elements per face. Say which path you used.
   WARNING about the skill, and it matters: its decision tree says "used above the fold but
   not preloaded -> add a preload", and its resource-hints check tolerates 5-6 preloads.
   Applied literally that is exactly the configuration that measures WORSE than preloading
   nothing. Use the skill for the inventory and the mechanical checks; take the preload
   decision from step 4 below and tell me where you departed from the tool.
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
   not automatically); WOFF2 or it is a finding.
   Then DO THE SUBSET PASS on the heaviest self-hosted face - I want a real file out of this
   step, not a recommendation. Ask me first whether my faces are licensed for modification.
   Then: open the source file in Wakamai Fondue (wakamaifondue.com) and tell me what is
   actually inside - glyph count, character sets, features, axes - and name what can go
   ("drop Cyrillic and Greek", not "subset it"). Regenerate through Transfonter
   (transfonter.org): latin + latin-ext for Polish content, WOFF2 only, and the font-display
   value chosen in step 5; keep the generated CSS, the unicode-range it emits is part of the
   deliverable. Report BEFORE and AFTER in bytes, per file and for the page type. Then VERIFY
   THE RENDERING, not the size: swap the file in locally and check the Polish diacritics on
   real content plus anything else the design leans on (currency, arrows, ligatures, tabular
   figures) - a subset that renders a tofu box is not a smaller file, it is a broken one, and
   I want you to say that you checked. Report it as TRANSFER SAVED with the header metrics
   before and after next to it; on most pages this moves no headline metric and that is the
   expected result. Do not inflate it into an LCP win.
   If an icon font is present, count how many of its glyphs the page actually uses.

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

## 3g. Stage 110 standalone: consent / CMP
Paste after workshop 14 (13.08). Needs the chrome-devtools MCP server. Run Stage 90 first if
you can - it tells you what your LCP element is when the banner is NOT in the way, which is
the comparison this stage is built on. Clear cookies before every run or you will measure a
page no first-time visitor ever sees.
```
Run .ai/web-performance/audit/110-consent-cmp.md for MY project.

Inputs: MY page URL and stack from site-profile.md, what Stage 90 found in ## Media (what is
the LCP element with consent already stored?), Stage 80 in ## Scripts, Stage 60 in ## Render
start, Stage 30 in ## Preconnect, the chrome-devtools MCP server, and MY SOURCE CODE - the
repo you are running in. Show me each step before moving on.

Measure MY page, the URL from site-profile.md. Do not substitute any example, demo or vendor
documentation page. If the URL, page types or stack are missing from the profile, ask me and
wait.

READ THIS FIRST, it decides whether the run is any good.
ONE component, THREE unrelated failures. The consent banner damages three metrics through
three mechanisms that share no fix: (1) a synchronous script high in <head> blocks the parser
- a LOADING problem; (2) the banner gets classified as the LCP element - a CLASSIFICATION
problem, nothing is actually slower; (3) the consent click releases a cascade of tags that
runs before the next paint - a SCHEDULING problem. Give me three separate verdicts. A single
verdict for "the cookie banner" means you have not done this stage. Two of the fixes also
pull against each other: anything that stops the script blocking makes the banner appear
LATER, which is worse for the user.
Second, THE CONSENT GATE, and it outranks every performance finding here. The vendor tells
you to load that script synchronously right after <head> on purpose - it is the driver that
sets the default denied state before anything else can write a cookie. So: no loading change
counts as done until you have verified on the MODIFIED page, as a first-time visitor, that no
non-essential cookie is written before consent. Show me the before and after lists. If you
cannot verify it, say UNPROVEN - do not say done.
Third, THE HONESTY TEST. The off-screen technique improves the LCP value by changing when the
banner becomes visible, not by making anything faster. So measure TIME-TO-BANNER-VISIBLE next
to LCP every time. If LCP improves and the banner arrives materially later, that is a
REGRESSION and I want to hear it called one.

0. Call emulate ONCE (mobile 412x765x2.6, Fast 4G, CPU 4x) and never change it. CLEAR COOKIES
   AND STORAGE BEFORE EVERY SINGLE RUN - the banner only appears to a first-time visitor, and
   measuring with consent stored is the standard way this whole stage silently produces
   nothing. Say that you did it. Navigate, then performance_start_trace with reload and
   autoStop. Also curl the page and keep the raw server HTML.

1. INVENTORY. Vendor, script origin (first- or third-party), transfer, loading attribute
   (none/async/defer), position in the document, how it arrives (directly in the HTML / as a
   tag inside a tag manager / injected by a script), tag manager container if any, consent
   framework (Consent Mode v2 / TCF / none). If BOTH a tag manager and a hardcoded consent
   script are present, say so - their ordering is now an invariant maintained by hand.

2. FAILURE 1, RENDER START. Is it render-blocking, and how many serialised hops before the
   request starts? Then stop arguing and measure it: BLOCK the CMP origin, reload, and give me
   Start Render / FCP with and without. That delta is the ceiling on what any loading fix can
   win - measured, not estimated.

3. FAILURE 2, LCP. On the first-time-visitor run, what is the LCP element? Compare against
   what Stage 90 recorded with consent stored. Three outcomes and they go different places:
   the banner IS the LCP element (failure real, step 6 applies); the LCP element is unchanged
   and the banner merely overlaps it (failure NOT real - say so and do not propose the
   off-screen trick); the element changed because the banner pushed content (that is a shift,
   route it to CLS). Give me the banner area on screen so this is quantified.

4. FAILURE 3, INP. Click the primary consent button with the trace running. Input delay,
   processing, presentation, total duration. What the click starts: how many requests, how
   much transfer, how much main-thread work in that frame. And the number that matters to the
   user: time from click to the banner being GONE. Do not assume this failure still exists -
   the large vendors shipped scheduling fixes for it. If it is already fast, record NO ACTION
   and move on. If the cascade is tags firing on a consent event rather than the CMP itself,
   name which.

5. THE LOADING DECISION plus THE GATE. Three options: leave it synchronous (correct, slowest);
   add defer (the minimum, partial win); relocate it into the tag manager as a tag on the
   consent-init event, so the document references the vendor origin nowhere (prefer the
   vendor's official GTM template over a custom HTML tag). Whichever you propose, RUN THE
   GATE: list non-essential cookies and storage written before consent, before and after, and
   confirm that with consent denied the analytics tags stay silent and the anonymised Consent
   Mode pings still go out. Then price in the cost: relocation makes the banner appear later,
   so pay it back with a preconnect to the vendor origin, or a preload of the script where the
   URL is stable.

6. THE OFF-SCREEN POLICY - only if step 3 said the banner IS the LCP element. Render it
   outside the viewport from the first frame (transform: translateY(110vh)) and bring it in
   after the page has painted. Read the container selector from the rendered DOM on MY page -
   the classes differ per vendor and change between versions, so do not copy one from
   anywhere. transform and opacity only. Never display:none or visibility:hidden - a consent
   dialog a keyboard or screen-reader user cannot reach is a worse defect than the metric it
   fixed, so verify it is still reachable. Then run the honesty test: LCP before/after, LCP
   element before/after, and time-to-banner-visible before/after.

7. THREE VERDICTS, then STOP and show me. One per failure, each FIX NOW / FIX LATER / NO
   ACTION, plus the gate result, plus how all this compares to what Stages 60, 80 and 90
   found. Do not start the experiment until I say so.

8. EXPERIMENT, only on a FIX NOW. Local HTML variants exactly as in Stage 60.5 (download the
   page, inject <base href>, one edit per variant, plus an untouched baseline; initScript does
   not survive the trace reload). Variants: baseline / defer / relocated-to-tag-manager with
   preconnect / off-screen CSS (that last one only if the banner is the LCP element). 3 rounds
   rotated, never in blocks, discard the warm-up, compare medians against the larger spread,
   AND CLEAR COOKIES BETWEEN EVERY RUN. For every variant record Start Render, LCP and WHICH
   ELEMENT WON IT, CLS, and time-to-banner-visible. Verdicts SUPPORTED / REGRESSION /
   INCONCLUSIVE.

Write results to site-profile.md > ## Consent / CMP and findings.md, tick Stage 110 in
## Progress, and end with the STOP line from the stage file.
```

## 3h. Stage 120 standalone: INP / interactions
Paste after workshop 15 (18.08). Needs a RUM tool with INP attribution, the chrome-devtools MCP
server, and - for the source mapping - the project running locally. Run Stage 80 first (it names
the scripts) and Stage 70 if you can (it owns recalculation cost, which is where a
presentation-delay problem goes). Without RUM you can still run it, but the target is a guess
and the stage says so in every line of the output.
```
Run .ai/web-performance/audit/120-inp-interactions.md for MY project.

Inputs: MY page URL and stack from site-profile.md, my RUM tool, what Stage 80 found in
## Scripts, Stage 70 in ## DOM size, Stage 110 in ## Consent / CMP, the chrome-devtools MCP
server, and MY SOURCE CODE - the repo you are running in. Show me each step before moving on.

Measure MY page and an element MY users actually click. Do not substitute any example or demo.
If the URL, page types or stack are missing from the profile, ask me and wait.

READ THIS FIRST, it decides whether the run is any good.
THE TARGET COMES FROM THE FIELD, NOT FROM THE LAB. INP is the duration of ONE interaction, and
my page offers dozens of them while real users only ever hit a handful. So you start in RUM:
which page, which element, which subpart dominates, how often it is hit, and when in the visit.
Only then do you reproduce it in the browser. Do NOT drive the browser to click every element
looking for the slow one - it burns tokens and time, and it still measures one device and one
network, mine, which is not the question. If I have no RUM, say so, agree with me on at most
three candidate elements, and mark the whole finding UNCONFIRMED IN THE FIELD.
Second, THE SUBPART IS THE DIAGNOSIS, the total is a symptom, and it decides whether this stage
owns the problem at all. Input delay dominating means the main thread was busy when the click
landed - that is a LOADING problem and belongs to Stages 60 and 80, and nothing in the handler
will fix it. Presentation delay dominating means the frame itself is expensive - that is
recalculation and layout, Stage 70. Only processing time belongs here. Tell me which one it is
before you propose anything.
Third, INP ENDS AT THE NEXT PAINT, not at the end of my JavaScript. The standard fix therefore
REORDERS work, it does not remove it. Do not report "we removed 400 ms of processing" unless
work was genuinely deleted. Say what now lands before the frame and what runs after it.

0. Call emulate ONCE (mobile 412x765x2.6, Fast 4G, CPU 4x) and never change it. Without CPU
   throttling almost anything passes on a laptop and this stage will clear a page that is broken
   on a mid-range phone. State the emulation next to every number.

1. TARGET, FROM RUM. Give me: the worst page type, its INP (p75 and the worst observed
   interaction), the subpart breakdown, the element selector or label, the interaction type
   (pointer or keyboard - keyboard on inputs is routinely worse and routinely forgotten), how
   often users hit it, when in the visit it happens, and the attributed script domain if the
   tool reports one. If two candidates are close, ask me which the business cares about -
   frequency beats severity.

2. REPRODUCE IT. Navigate, performance_start_trace with reload, let the page settle, perform
   THAT ONE interaction, stop. Reproduce the user's timing too: if RUM says it happens ten
   seconds in, do not click it 200 ms after load. Then go to the interactions track, find the
   pointer marker, narrow to it, and read input delay, processing duration, presentation delay
   and the total. If the lab number does not resemble the field number, STOP and tell me -
   usually it is the device class or some state I have and you do not (a full cart, a long list,
   logged in).

3. ATTRIBUTE IT. In the main track under the interaction, whose code is running: my bundle, the
   framework, a tag manager (gtm.js), another vendor? Use debug with AI / Find Improvements - it
   is fast and useful - but treat what it says as a HYPOTHESIS and confirm it in the trace. Then,
   because the public site is minified and neither of us can map a bundle offset to my repo, run
   the project LOCALLY from source, reproduce the same interaction, and give me the real file and
   line. If I have a RUM MCP server connected, you can also just ask it for the slowest
   interaction with script attribution from inside my repo folder.

4. ROUTE BY SUBPART, and say it plainly. Input delay dominant -> loading problem, route to
   Stages 60/80, the button's code is not at fault. Presentation delay dominant -> recalculation
   and layout, route to Stage 70, and check the animation is limited to transform and opacity.
   Processing time dominant -> continue here. Total under 200 ms and nothing dominant -> NO
   ACTION with the numbers attached, which is a legitimate and common result on a project that
   already did Stages 60-110.

5. IS ANY OF IT NEEDED? Before scheduling work more cleverly, look at what the handler actually
   does: a computation whose result is never read, measurement of elements nobody animates, a
   loop sized by a forgotten constant. Deleting that is the ONLY fix here that genuinely reduces
   work. Ask me before removing anything.

6. THE SCHEDULING DECISION. Name the visual change the user must see first - spinner, state
   change, opened panel, navigation - and put it first. Then yield: a promise around setTimeout
   works everywhere but sends the continuation to the BACK of the queue; scheduler.yield keeps
   its priority after user input but is not universally supported, so ship it behind a feature
   check with a setTimeout fallback. Do not present the two as the same fix. Where the handler
   waits on a network write, consider showing the result optimistically and reconciling after.
   Where the cost is a genuinely heavy pure computation, a web worker is the right home - it has
   no DOM access, so anything touching the page comes back as a message. Where the cost is a tag
   cascade, the visual change goes first and the dataLayer push after it - then VERIFY the events
   still arrive, on a normal navigation and on an SPA route change.

7. VERDICT, then STOP and show me. Target and how it was chosen, field and lab numbers with
   subparts, the dominant subpart and its route, the attribution with file and line, the
   decision, and how this compares to what Stages 60, 70, 80 and 110 already found. Do not start
   the experiment until I say so.

8. EXPERIMENT, only on a FIX NOW. Same emulation. One change per variant, plus an untouched
   baseline; DevTools local overrides on the HTML document are fine for testing a snippet before
   it goes in the repo - say when a number came from an override. Repeat the interaction at
   least 5 times per variant, rotated between variants and never in blocks, discarding the first
   run after each load: interaction timings are noisier than load timings. Compare medians
   against the spread and call anything smaller INCONCLUSIVE. For every variant record the total,
   all three subparts, and TIME FROM CLICK TO THE VISUAL CHANGE ON SCREEN - a variant that
   improves the metric and delays what the user sees is a REGRESSION. Verdict is SUPPORTED IN
   LAB, not "fixed": a lab click is a reproduction. Set a date to re-read RUM and confirm the
   change reached real users.

Write results to site-profile.md > ## INP / interactions and findings.md, tick Stage 120 in
## Progress, and end with the STOP line from the stage file.
```

## 3i. Stage 130 standalone: JS runtime after load
Paste after workshop 16 (20.08). Needs the chrome-devtools MCP server, a WebPageTest run of the
same page, and - for the source mapping - the project running locally. Run Stage 80 first (it
names the scripts) and Stage 70 if you can (it owns recalculation cost). This is the stage that
looks at the period nobody measures: after Document Complete, with nobody waiting.
```
Run .ai/web-performance/audit/130-js-runtime.md for MY project.

Inputs: MY page URL and stack from site-profile.md, what Stage 80 found in ## Scripts, Stage 70
in ## DOM size, Stage 120 in ## INP / interactions, the chrome-devtools MCP server, and MY
SOURCE CODE - the repo you are running in. Show me each step before moving on.

Measure MY page. Do not substitute any example or demo. If the URL, page types or stack are
missing from the profile, ask me and wait.

READ THIS FIRST, it decides whether the run is any good.
A COMPONENT THAT APPEARS ON SCROLL IS NOT A COMPONENT WHOSE CODE RUNS ON SCROLL. A section can
fade in on an observer while the library behind it was downloaded, parsed and executed at
startup inside a shared vendor chunk. It looks lazy and it is not, and the cost is smeared
across the load instead of sitting in one obvious task - which is why it hides. Read the TRIGGER
in my source and confirm it in the trace. Never infer it from where the component sits on the
page.
Second, BYTES ARE ONE QUARTER OF THE BILL: download, parse and compile, execution on the main
thread, memory and GC. A finding that quotes only transfer size has measured the cheapest part.
Third, THE ORDER IS DELETE, THEN DEFER THE CODE, THEN DEFER THE WORK, THEN DEFER THE RENDERING.
content-visibility belongs in the last group - it skips layout and paint, not JavaScript. Do not
sell it as a fix for a heavy script.

0. Call emulate ONCE (mobile 412x765x2.6, Fast 4G, CPU 4x) and never change it. State it next to
   every number.

1. FIND THE TAIL. navigate_page, performance_start_trace with reload, and LET IT RUN several
   seconds past the point where the page looks finished. Do not scroll and do not click - the
   whole stage is about the period when nobody is waiting. Give me: Total Blocking Time, the
   individual long tasks that make it up, and how long the main thread stayed busy after
   Document Complete. Cross-check the JS execution bands against Document Complete on a
   WebPageTest run of the same page. If the thread goes quiet within a second and TBT is small,
   say NO ACTION with the numbers and stop.

2. ATTRIBUTE IT. Name the file behind each long task. Then BLOCK that request in the Network
   panel, reload, and tell me what visually breaks - that is the cheapest way to turn a hashed
   chunk into a feature I can name. It is a diagnostic, not a fix. Say what else travels in that
   chunk: unrelated libraries sharing one vendor file is its own finding, and its fix is build
   config, not code. Then run the project LOCALLY from source and give me the real file, because
   neither of us can map a bundle offset to my repo. debug with AI is a fast hypothesis, not a
   verdict - and its DOM and style recommendations belong to Stage 70, so route them.

3. READ THE TRIGGER for each candidate: module scope, a DOMContentLoaded handler, an
   intersection observer, a framework directive, hydration, or something repeating (scroll
   handler, rAF loop, timer, an observer that never disconnects). For an observer, tell me
   whether it triggers the IMPORT or only the RENDER. For a framework directive, confirm in the
   BUILT output that the code really landed in its own chunk. Then ask the question the trigger
   cannot answer: does the user ever see this component on this page?

4. HYDRATION AND RE-RENDERS, framework stacks only - say so and skip if it does not apply.
   Hydration cost follows the size of the component tree, not what is on screen; tell me what
   could be excluded from it. For re-renders, use a highlighter (React Scan injects fine through
   DevTools local overrides, in head before other scripts, no install needed) and record a
   throttled trace while doing something realistic, so highlights become main-thread time. Then
   say which of those re-renders HAD to happen. A deterministic repo scan is worth a run for the
   inventory; hand the shortlist to an agent for fixes. If a memoising compiler is available,
   treat it as a lever to measure, not a rescue - the win scales with how bad the code was.

5. DECIDE, per candidate: delete (the only option that removes all four costs), defer the code
   into its own chunk imported when needed, defer the expensive initialisation while keeping the
   module, defer the RENDERING with content-visibility and contain-intrinsic-size, or stop the
   repetition. Say which one and what will happen instead of what happens today.

6. VERDICT, then STOP and show me: the tail, the owners, the triggers, an explicit
   lazy-appearance-without-lazy-code line, the decision per candidate, and how this compares to
   what Stages 60, 70, 80 and 120 already found. Do not edit until I say so.

7. EXPERIMENT, only on a FIX NOW. Same emulation, one change per variant, baseline first, at
   least 5 runs per variant, medians with the spread, anything smaller INCONCLUSIVE. Record all
   four costs before and after - bytes, parse and compile, execution, and where the last long
   task now ends relative to Document Complete - plus TBT. Check whether a load-time cost just
   became an interaction-time cost: deferred code executes when the user arrives, and if that is
   on a click, Stage 120 now owns a new problem. Verify the feature still works on a mid-range
   device and a slow connection.

Write results to site-profile.md > ## JS runtime and findings.md, tick Stage 130 in ## Progress,
and end with the STOP line from the stage file.
```

## 3j. Stage 140 standalone: navigation and bfcache
Paste after workshop 16 (20.08). Needs CrUX data for the origin (any tool exposing navigation
types) and a browser. Cheap to run and frequently ends in NO ACTION - which is the point: the
field data tells you within minutes whether any of it is worth an afternoon.
```
Run .ai/web-performance/audit/140-navigation-bfcache.md for MY project.

Inputs: MY origin and page types from site-profile.md, what Stage 40 found in ## Cache & CDN,
Stage 110 in ## Consent / CMP, my RUM tool if I have one, and a browser (chrome-devtools MCP is
enough). Show me each step before moving on.

READ THIS FIRST.
SIZE IT FROM THE FIELD BEFORE DOING ANYTHING. Navigation types are published for any origin with
enough traffic. Read the share of back/forward navigations against the share that were served
from back/forward cache - a large first number with a near-zero second one is the finding, and a
back/forward share of 1% means this stage should end in NO ACTION. Split by device where you
can: back and forward is mostly mobile behaviour, so my own desktop habits are a bad proxy.
Second, BFCACHE IS NOT BUILT, IT IS MERELY NOT BROKEN. Never propose "implementing" it. Ask the
browser whether it works and what is blocking it.
Third, PREFETCH AND PRERENDER MOVE ORIGIN WORK EARLIER, THEY DO NOT REMOVE IT. The output here
is a budget, not a switch.

1. FIELD SIZING. Give me the navigation type shares for my origin: navigate, navigate cache,
   back/forward, back/forward cache, reload, restore, prerender - and the gap between the two
   back/forward numbers. No CrUX data for the origin? Say so, fall back to RUM, and mark the
   sizing UNCONFIRMED IN THE FIELD.

2. BFCACHE. Open the page, navigate away, come back, and read DevTools > Application >
   Back/forward cache with its Test action. Record verbatim whether it was served and every
   blocker listed. Do this on EACH page type, not just the homepage - blockers are usually
   per-template. Do not reason from Cache-Control headers: the rules changed and the browser is
   authoritative. Attribute each blocker to my code or to a named vendor script using the
   Stage 80 inventory; an unload handler inside a third-party tag is the classic case, and the
   fix is a conversation, not a commit.

3. ANALYTICS ON RESTORE. A restored page does not load, so nothing re-runs and a page view that
   fires once per load will not fire at all. Test it: navigate away, come back, watch the
   network. Tell me whether I undercount, double count, or am correct, and whether a pageshow
   path exists. This matters because fixing bfcache without it looks like a drop in traffic and
   somebody will revert a correct change.

4. PREFETCH TODAY. Establish what my framework prefetches by DEFAULT and prove it: open the
   page, open the navigation menu, and count the requests that fire before any click. Cost it
   against how many pages a visitor actually opens, remembering that a prefetched server-rendered
   response is real origin work. Then propose a policy - leave it alone, prefetch on hover, or
   prefetch only the routes analytics and heatmaps show people actually take.

5. SPECULATION RULES, document navigations only - if my site navigates client-side inside a
   framework app, say so and skip this. Check whether rules are already present (some platforms
   ship them by default, so I may be speculating on a policy nobody chose) and whether that
   policy is defensible. Distinguish prefetch (fetches the document) from prerender (loads AND
   renders the target page, scripts included) - they do not cost the same. Recommend an
   eagerness and justify the budget; moderate prerender on a handful of key routes buys the most
   for the least, because a hover is a signal of intent. Then verify that a prerendered page
   that is never visited does not count as a visit and writes nothing before consent.

6. VERDICT, then STOP and show me: field sizing, bfcache verdict per page type with attributed
   blockers, the analytics result, what fires before a click today, and the policy you propose
   with the reason for that budget. NO ACTION is a legitimate outcome here - say it if the field
   data says it.

7. EXPERIMENT, only on a FIX NOW. Remove or route the blocker, re-run the Application test, and
   confirm the restore on a throttled connection. Count requests before and after for the
   prefetch policy, and measure click to painted target on speculated routes. Set a date to
   re-read the navigation types in the field - until then the bfcache result is SUPPORTED IN
   LAB.

Write results to site-profile.md > ## Navigation and findings.md, tick Stage 140 in ## Progress,
and end with the STOP line from the stage file.
```

## 3k. Stage 150 standalone: layout stability (CLS)
Paste after workshop 17 (25.08). Needs field data for the origin (CrUX or RUM), WebPageTest and a
browser. Half of this stage is a pass no page-load test can run, so budget a few minutes of
actually using the page with an observer attached.
```
Run .ai/web-performance/audit/150-layout-stability.md for MY project.

Inputs: MY origin and page types from site-profile.md, what Stage 60 found in ## Render start,
Stage 90 in ## Media, Stage 100 in ## Fonts, Stage 110 in ## Consent / CMP, Stage 130 in
## JS runtime, my field data (CrUX or RUM), WebPageTest and a browser (chrome-devtools MCP is
enough). Show me each step before moving on.

READ THIS FIRST.
A LAB SCORE MEASURES A WINDOW; THE FIELD MEASURES THE WHOLE VISIT. A lab zero only says nothing
moved before the tool stopped watching. Size this from the field first, split by device and page
type, and where lab and field disagree treat the field as the measurement and the lab as the
reproduction.
Second, THE COUNTED SET AND THE HARMFUL SET ARE NOT THE SAME. A confirmation message I scroll the
user to on purpose scores as a shift and is good UX. A reflow within 500 ms of a click is excluded
from the metric and still happened under the user's finger. Report both verdicts and let them
disagree.
Third, MOST SHIFTS HERE ARE THE PRICE OF AN EARLIER FIX - async CSS, lazy media, a swapped font, a
late consent banner, layout decided in JavaScript. Route each finding back to the stage that owns
it instead of treating it as a new topic.
Fourth, A RESERVATION WRITTEN IN JAVASCRIPT IS NOT A RESERVATION. It arrives exactly as late as
the element it was supposed to hold.

1. FIELD SIZING. CLS at p75 for my origin, split by device and per page type where the data
   allows, next to the lab number for the same page. Name the source. No field data? Say so, fall
   back to RUM, mark it UNCONFIRMED IN THE FIELD.

2. LOAD-PHASE INVENTORY. WebPageTest event details for the CLS metric (a screenshot per shift with
   the moved regions boxed), then a trace through the browser on ONE stated emulation (mobile,
   Fast 4G, CPU 4x). Give me a SELECTOR per shift, not a region, plus its score and whether it is
   above the fold. Compare the screenshots either side of each shift: an element that is unstyled
   in the first has a CSS arrival problem, an element that is absent has an insertion problem.

3. THE USE-PHASE PASS - do not skip this, it is why the stage exists. Attach a PerformanceObserver
   for layout-shift, logging value, hadRecentInput and the sources array, then USE the page for at
   least 15 seconds after it looks ready: wait without touching anything, scroll slowly to the
   bottom, trigger the real interactions, and navigate to another page type and back. Report
   shifts during LOAD and shifts during USE as two separate populations.

4. ATTRIBUTION, in this order. Is the sizing present in the HTML the server sent - checked in
   view-source, not in the DOM inspector? Which earlier technique caused it? Whose code is it
   (my template, theme, plugin or page builder, third-party tag)? Does it recur across page
   types? On WordPress search the plugins and the builder's saved post meta too. Unattributed
   shifts are reported as "source not identified", never guessed.

5. RESERVATION PER SHIFT, in CSS wherever the source allows: dimensions or aspect-ratio on media,
   a min-height per breakpoint on ad slots sized to what is actually served, the box on the
   wrapper rather than on the widget, and - the one most often missed - the initial state of a
   JS-driven component expressed in CSS so the first render is already correct. Fonts go back to
   Stage 100. Nothing gets injected above the user's current position unless the same change
   scrolls them there on purpose. Animations on transform and opacity only.

6. VERDICT, then STOP and show me: field sizing, both inventories with owners, the metric verdict
   against the UX verdict, the reservations ordered by what users actually meet, and what you
   routed back to another stage. NO ACTION is a legitimate outcome if the field is clean and
   nothing moves under the user's hands.

7. EXPERIMENT, only on a FIX NOW. Prove the reservation with Local Overrides on the real page
   before any deploy - override the document from the Network panel and paste the CSS inline.
   Re-measure on the same emulation, 5 runs minimum, compare medians and state the spread; a
   difference smaller than the spread is INCONCLUSIVE. Re-run the use-phase pass afterwards, not
   only the load trace, and report honestly how much of the shift is left. Set a date to re-read
   the field - until then the result is SUPPORTED IN LAB.

Write results to site-profile.md > ## Layout stability and findings.md, tick Stage 150 in
## Progress, and end with the STOP line from the stage file.
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
