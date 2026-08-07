# Stage 90: Images and video - two populations, two different problems

> Can be run STANDALONE or from the index (.ai/web-performance/audit/00-index.md).
> Behaviour: DO stage. The measurement is cheap and scriptable (every image reports its own
> intrinsic size), but the stage fails if you treat all images as one population - see the
> hard boundary below.
> Read domain/URLs/stack from site-profile.md; if missing, ASK me and WAIT - do not guess.
> Input:  site-profile.md > ## Project, ## Page types, ## Render start (Stage 60),
>         ## Preconnect (Stage 30), ## Cache & CDN (Stage 40), ## Scripts (Stage 80)
>         + the chrome-devtools MCP server + MY SOURCE CODE
> Output: site-profile.md > ## Media + findings.md, ending with TWO verdicts (LCP element,
>         everything else) and a per-asset decision table that names MY files.
>         Runs AFTER Stage 60.
> Idempotent: if ## Media exists, update it, do not create a second one.

THIS RUNS AGAINST MY PROJECT, NOT AN EXAMPLE (read first):
The page you measure is MY page - the URL in site-profile.md > ## Project, for the page
type I am auditing. Never substitute a demo, a documentation example, or a site mentioned
anywhere in this package. If the URL, the page types or the stack are missing from the
profile, ASK me and WAIT.

Any figure that appears in this file is there to describe a MECHANISM or a threshold
defined by the tooling - not a measurement of my site. My project's numbers will be
different. Never copy a figure from this file into findings.md and never treat one as a
target.

The deliverable is not "your images are too heavy". It is: THIS image, at THIS intrinsic
size, rendered at THIS size, in THIS format, discovered at THIS moment, from THIS file in
my repo or THIS media library entry - and here is the one change that matters most for it.

## SCOPE AND HARD BOUNDARY (read first - IMPORTANT)

Images on a page are not one problem. They are two, and they are fixed by opposite means:

1. **The LCP element is a LATENCY problem.** One image, usually one only. Its cost is not
   bandwidth - it is the time between the HTML arriving and the browser even knowing the
   image exists, plus the priority it gets once it does. Making it smaller helps a little.
   Making it DISCOVERABLE and HIGH PRIORITY helps a lot. Lazy-loading it is actively
   harmful.
2. **Every other image is a BANDWIDTH and MAIN-THREAD problem.** Dozens of assets, none of
   them individually decisive, competing for the same connection as the CSS and the
   JavaScript. Their fix is dimension, format, compression and deferral - and the LCP
   treatment (eager, high priority, preloaded) applied here makes the page WORSE, because
   it puts the LCP image in a queue behind things nobody is looking at.

Conflating the two is the single most common way this stage produces a wrong answer. A
report that gives one verdict for "the images" has not done the work. Produce two.

Three further boundaries:

- **Order of levers is fixed, and it is not the order tools suggest.** Dimension first,
  then format, then compression, then loading strategy. Serving a 3000 px image into a
  300 px box is a 10x error; re-encoding that same file to AVIF at quality 80 is maybe a
  30% error. Starting with compression means you "win" on the small lever and leave the
  big one untouched. If you propose a format change for an image that is grossly
  oversized, you have skipped a step - say so in the report.
- **This stage does not own CLS, it only feeds it.** Missing `width`/`height` is recorded
  here as a finding because it is an image attribute, but the shift is measured in the CLS
  stage. Record it, route it, do not try to quantify layout shift here.
- **Anything a crawler must index stays indexable.** Deferring an image is safe; hiding
  content behind an interaction that no crawler performs is not.

## 90.0 Read what earlier stages already found (Do, no new measurement yet)
- From site-profile.md > ## Render start: Start Render / FCP / LCP for the slow page type,
  and which resources Stage 60 already deferred or deleted. If Stage 60 already identified
  the LCP element, carry that forward - but re-confirm it in 90.2, because it can differ per
  viewport.
- From site-profile.md > ## Preconnect (Stage 30): which external origins are warmed. You
  need this in 90.5 if images come from an image CDN.
- From site-profile.md > ## Cache & CDN (Stage 40): whether static assets are edge-cached
  with a long immutable TTL. Images are the largest population of static assets on most
  pages - a finding there applies here in bulk.
- From site-profile.md > ## Scripts (Stage 80): whether a script is responsible for
  inserting images (a slider, a gallery, a lazy-load library). An image injected by
  JavaScript cannot be found by the preload scanner, which changes every conclusion in 90.2.
- GATE: if Stage 50 says TTFB is the wall, note it and carry it into 90.8. Run the stage
  anyway - the inventory is cheap - but the verdict must say out loud whether media work is
  the next thing to do.

## 90.1 Inventory - every image and video on the page type (Do)
Everything here runs through the chrome-devtools MCP server.

- Call `emulate` ONCE: mobile viewport ~412x765x2.6 (mobile, touch), "Fast 4G", CPU
  throttling 4x. Do not change it for the rest of the stage. The viewport matters more here
  than in any previous stage: `srcset` picks a different file per viewport and per device
  pixel ratio, so an audit run at desktop width measures a different set of files than my
  users download. State the viewport next to every number you record.
- `navigate_page` to the page type you are auditing, then `performance_start_trace` with
  `reload: true` and `autoStop: true`.
- Also `curl` the page and keep the raw server HTML. An image present in the DOM but absent
  from the server HTML was injected by a script - it cannot be preloaded by the scanner and
  it usually cannot be fixed by editing my template.

Then collect the per-element data with `evaluate_script`. This is the measurement the whole
stage rests on, and the browser gives it to you directly:

```js
// One row per raster image actually in the document.
[...document.querySelectorAll('img')].map(el => {
  const r = el.getBoundingClientRect();
  return {
    src: el.currentSrc || el.src,          // currentSrc = what srcset ACTUALLY picked
    intrinsic: [el.naturalWidth, el.naturalHeight],
    rendered: [Math.round(r.width), Math.round(r.height)],
    dpr: devicePixelRatio,
    hasDims: el.hasAttribute('width') && el.hasAttribute('height'),
    loading: el.getAttribute('loading'),
    fetchPriority: el.getAttribute('fetchpriority'),
    decoding: el.getAttribute('decoding'),
    sizes: el.getAttribute('sizes'),
    srcset: el.getAttribute('srcset'),
    inViewport: r.top < innerHeight && r.bottom > 0,
  };
});
```

Then do the same for CSS backgrounds (`getComputedStyle(el).backgroundImage !== 'none'`)
and for `<video>` / `<iframe>` embeds. Backgrounds will not appear in the `img` list at all,
and they are the ones most likely to be the problem - see 90.5.

Build ONE row per asset:

| asset | element type | origin | intrinsic px | rendered px (x DPR) | format | transfer | loading | fetchpriority | in server HTML? |
|---|---|---|---|---|---|---|---|---|---|

- **element type** is `img` / `picture` / `css-background` / `svg-inline` / `svg-img` /
  `video` / `iframe-embed` / `data-uri`.
- **rendered px (x DPR)** is the CSS box multiplied by the device pixel ratio. That, not the
  CSS box, is the number of real pixels the image needs.
- If the trace exposes an image-delivery insight (`ImageDelivery` or equivalent), call
  `performance_analyze_insight` on it and record what it says - but treat its "estimated
  savings" as a PREDICTION, per the evidence rule in 00-index.md, never as a measurement.

## 90.2 The LCP element - identify it, then classify it (Do)
This is one image and it gets its own analysis. Do not average it into the rest.

- Identify it from the trace, not by eye. If an LCP insight is available
  (`LCPBreakdown` / `LCPDiscovery` or equivalent), call `performance_analyze_insight` and
  record the phase split: TTFB, load delay, load time, render delay. Repeat 3 times and
  record the spread.
- **Re-confirm it at my mobile viewport.** The LCP element on desktop is frequently not the
  LCP element on mobile - a different hero crops in, or a text block wins because the image
  is below the fold. State which viewport the answer applies to.
- Then classify how it is delivered, because the fix follows entirely from this:
  - **`<img>` in the server HTML.** Best case. The preload scanner finds it while the HTML
    is still being parsed, before CSS or JavaScript have run.
  - **`<img>` injected by a script.** The scanner never sees it. Discovery waits for the
    script to download, parse and execute (see Stage 80 for what that costs).
  - **CSS `background-image`.** The scanner cannot see it either: the URL is inside a
    stylesheet, so the browser must download the CSS, build the CSSOM, match the rule to an
    element and lay it out before it knows the image exists. This is the single most common
    reason an LCP image starts late, and no amount of compression fixes it.
  - **A slide in a carousel.** Add the carousel's own JavaScript to the discovery chain.
- Now read the phase split and say which half of the problem this is:
  - **load delay dominates** -> a DISCOVERY problem. Fix = make it findable (move it into
    the server HTML as `<img>`, or `preload` it), plus priority.
  - **load time dominates** -> a TRANSFER problem. Fix = dimension and format, as in 90.3
    and 90.4.
  - **render delay dominates** -> the image arrived and could not be painted. That is not
    an image problem; route it back to Stage 60 (render-blocking) or Stage 80 (main thread
    busy).
- Check the obvious failure modes on this one element specifically:
  - is it `loading="lazy"`? That is a direct regression - lazy defers discovery until layout
    has run. Record it as the finding, not as a detail.
  - what priority did it get in the network list? An LCP image landing at Low or Medium
    priority is a `fetchpriority="high"` candidate.
  - is it served from a different origin than the document? Then a first visit pays DNS +
    TCP + TLS before the first byte. Cross-check Stage 30: is that origin preconnected?

## 90.3 Dimension - the first lever, and usually the biggest (Do)
For every asset from 90.1 EXCEPT the LCP element:

- Compute the ratio `intrinsic px / (rendered px x DPR)` in each axis. A ratio above ~2 in
  the width axis means the file carries at least 4x the pixels it needs, since area scales
  with the square. Sort the table by wasted pixels, not by file size - a moderately heavy
  image at the right size is fine; a light image at 5x the size is a bug that will get worse
  the moment someone uploads a bigger one.
- For each oversized image, find out WHY, because the fix differs:
  - **no `srcset` at all** - the browser has one file and no choice.
  - **`srcset` present but `sizes` wrong or absent.** Without `sizes` the browser assumes
    the image spans the full viewport width (`100vw`) and picks accordingly. A column that
    is a third of the screen will therefore get a file three times too wide. This is the
    most common single cause and it is invisible unless you compare `currentSrc` against the
    rendered box - which 90.1 already did.
  - **the variants that exist are wrong.** The CMS generated 300 / 600 / 1000 px but the
    layout needs 1500 px, so the browser picks the largest available and upscales, or the
    theme hardcodes the full-size file to avoid blur.
- Record `currentSrc` for each. That is the file my users actually downloaded at this
  viewport; the `src` attribute frequently is not.
- Note where `sizes="auto"` applies: it lets the browser use the element's computed width
  instead of my declaration, which removes the whole class of wrong-`sizes` bugs. It is
  only valid on images that are also `loading="lazy"`, and support is not universal - in
  browsers without it the value falls back to the default. So it is a simplification for
  below-the-fold images, never a fix for the LCP element.

## 90.4 Format and compression - the second and third levers (Do)
Only after 90.3, and only for images that are already close to the right dimension.

- Record the current format per asset and whether a modern format is being served at all.
  AVIF and WebP typically buy a substantial reduction at equal perceived quality, but the
  figure varies enormously by image content - photographs, flat graphics and screenshots
  behave differently. Do not quote a generic percentage; measure the actual files if you
  propose a change.
- Check whether format is negotiated per browser (a `<picture>` with `<source type>`, or an
  image CDN doing content negotiation) or hardcoded. A single AVIF with no fallback is a
  bug on browsers that cannot decode it.
- Quality: the interesting range is usually well below the default the CMS uses. Report it
  as a hypothesis to test on MY images, not as a number to copy.
- Then state which of the two optimisation MOMENTS my project uses, because the change is
  made in a different place depending on the answer:
  - **Upload time** - conversion and resizing happen when the file enters the system (a CMS
    hook, a build step). The served file is already correct. Cheap at runtime, but it only
    applies to assets uploaded after the change - existing media needs a regeneration pass.
  - **Delivery time** - the origin keeps one large master and an image CDN transforms it per
    request, per device, caching the result at the edge. Flexible, applies retroactively to
    everything, but adds a third-party origin with its own connection cost, and the first
    request for each variant is slow until it is cached.
  - The two are not exclusive and the right answer is usually both. Ask me which exists
    today before proposing either - this is a question-driven step.
- Whichever moment applies: **an image CDN without correct `srcset` and `sizes` does almost
  nothing.** It will happily serve one enormous transformed file to every device. Do not
  report "we have an image CDN" as if it were a solution.

## 90.5 Loading strategy - who waits, who does not (Do)
Now the fourth lever. Go asset by asset.

- **Above the fold, not the LCP:** eager, no lazy, ordinary priority. Do not mark these
  `fetchpriority="high"` as a group. Priority is zero-sum: every high-priority image
  competes with the LCP image, with the CSS and with the JavaScript for the same connection.
  Flag any page that marks several images high - Next.js does this automatically for every
  image given the `priority` prop, and a hero section with ten "priority" images produces
  ten preloads that collectively slow down the one that matters.
- **Below the fold:** `loading="lazy"`, and consider `decoding="async"` to keep the decode
  off the critical rendering path. `decoding` is a hint with a modest effect - record it as
  hygiene, never as a headline finding.
- **Not visible at all until an interaction** (a mega-menu, an off-canvas panel, a modal, a
  tab that is not open): these are frequently loaded eagerly at full priority because the
  markup is present and merely hidden. Lazy alone may not be enough if the element is in the
  layout. Prefer not putting them in the document until the interaction.
- **CSS backgrounds below the fold:** there is no native lazy loading for
  `background-image`. If the waterfall shows a large decorative background loading
  alongside the CSS and the JavaScript, the options are: convert it to an `<img>` (which
  also gets it `alt` text and native lazy), or apply the background from an
  `IntersectionObserver` when the section approaches the viewport. Say which one fits the
  element - a genuinely decorative texture and a content image are different calls.
- **CSS background as the LCP element:** if 90.2 landed here, the fix order is: (1) make it
  an `<img>` if the design allows, (2) if it cannot move, `preload` it with
  `fetchpriority="high"` - and use `imagesrcset` / `imagesizes` on the preload so the
  preloaded file matches the one the CSS will actually request, otherwise you download two.
  Note in the report that a preloaded background still typically paints later than the same
  file as an `<img>`, because rendering still waits on the CSSOM and layout. Preload narrows
  the gap; it does not close it.
- **`<iframe>` embeds:** `loading="lazy"` is supported but browsers apply their own
  heuristics and frequently load them earlier than expected. Verify in the waterfall rather
  than assuming. If an embed below the fold still loads during the initial burst, an
  `IntersectionObserver` facade is the reliable answer (see 90.7).
- **Missing `width`/`height`:** record every asset without both attributes (or without a CSS
  `aspect-ratio`). This is a CLS finding - route it, do not measure it here. Note also that
  CSS can override the attributes: `height: auto` with a fixed width preserves the ratio,
  but a rule that sets both dimensions independently will discard the reserved box. Check
  the computed styles before reporting the attributes as sufficient.

## 90.6 SVG, sprites and data URIs (Do)
- For each SVG, record how it is embedded: `svg-inline` (markup in the document),
  `svg-img` (an external file in an `<img>`), or a sprite sheet referenced by `<use>`.
- The trade-off, stated plainly: inline costs no request and paints immediately, but adds
  its weight to EVERY HTML response and cannot be cached separately. An external file is
  cacheable and keeps the document small, at the cost of one request. A sprite sheet is one
  cacheable request for many icons, but if it contains every icon in the project it can be
  large enough to compete with the CSS and the JavaScript in the initial burst - check the
  waterfall before assuming a sprite is the efficient option.
- Reasonable default to test against: inline only the few icons above the fold that are
  simple, external `<img>` for the rest, sprite when the icon count is high. Report the
  actual document size contributed by inline SVG - if the HTML is unusually large, this is
  frequently why.
- Run the SVGs through an optimiser (SVGO) and report the before/after. Files exported from
  design tools routinely carry several times the markup they need. This is one of the few
  findings in this stage that is safe, mechanical and immediate.
- **Data URIs / base64:** flag every one. Encoding inflates the payload by roughly a third,
  moves the bytes into a document that is usually not cached, and forfeits separate caching
  of the asset itself. The "fewer requests" argument it rests on is largely obsolete over
  HTTP/2 and HTTP/3. Report the total base64 payload in the document - WordPress plugins
  that "automatically optimise" a site are a common source, and the result is an enormous
  HTML file re-downloaded on every visit.
- Low-quality image placeholders are a related pattern: a tiny blurred inline preview shown
  until the real image arrives. It is defensible only when the placeholder is genuinely
  minute. Do not propose it as an optimisation for an image that could simply be loaded
  properly and quickly - it adds machinery and delays the real image.

## 90.7 Video (Do, if any video is present)
- **Self-hosted `<video>`:** record duration, transfer size, resolution, and `preload`.
  `preload="auto"` (or the default) on a below-the-fold video means the browser starts
  buffering it during the initial page load, alongside the CSS and the JavaScript.
  `preload="none"` or `preload="metadata"` is nearly always correct for anything not
  playing immediately.
- **`poster`:** record whether it is set. Without one, the browser has nothing to paint
  until enough video has buffered. Check the trace to see what the LCP candidate actually
  is on a page with a hero video - a poster image is a normal LCP candidate and is treated
  as an image everywhere else in this stage; do not assert which element won, read it.
- **Animated GIF:** flag every one. A GIF is a sequence of full frames with no modern
  compression; the same clip as MP4 or WebM is typically an order of magnitude smaller. The
  replacement is a `<video>` with `autoplay muted loop playsinline` (`muted` is what makes
  autoplay permitted at all; `playsinline` is what stops iOS opening it fullscreen). That
  combination behaves like an animated image while giving you real compression and control.
- **Per-device variants:** there is no native responsive selection for video equivalent to
  `srcset`. If the same large file is served to phones and desktops, the options are
  multiple `<source>` elements chosen by script, or simply a smaller single file - a
  background video under an overlay tolerates far more compression than one the user watches
  deliberately. Report the current state; do not build the JavaScript solution here.
- **Third-party embeds (YouTube, Vimeo):** these are not one video file, they are a player -
  frequently megabytes of JavaScript and CSS before anything plays. Record the transfer and
  the main-thread cost from the Stage 80 methodology. The fix is a facade: render our own
  thumbnail with a play button and only insert the real embed on click.
  `lite-youtube-embed` and `@next/third-parties` provide this ready-made. Note in the
  report that this changes nothing for a user who plays the video and saves everything for
  the majority who do not.

## 90.8 The priority verdict - TWO of them (Do - STOP and show me)
Before proposing a single fix, answer in one short block. Two verdicts, separately:

**A. The LCP element**
- what it is, how it is delivered, at which viewport;
- the phase split from 90.2 with the spread, and which phase dominates;
- whether it is lazy-loaded, what priority it got, whether it is cross-origin and warmed;
- verdict: **FIX NOW** / **FIX LATER** / **NO ACTION**, with the single highest-value change.

**B. Everything else**
- total image transfer for the page type, and how much of it is wasted pixels (from 90.3);
- the worst 3 offenders by wasted pixels, with the reason (`srcset` / `sizes` / variants);
- SVG and base64 payload contributed to the document;
- video findings, if any;
- how this compares to what Stage 60 and Stage 80 found - is media a visible share of this
  page's problem, or noise next to a 2-second TTFB?
- verdict: **FIX NOW** / **FIX LATER** / **NO ACTION**.

The two can disagree, and frequently do: a page can have a badly delivered LCP image worth
fixing this afternoon and a perfectly reasonable long tail, or the reverse. Say so.

Show me this block and wait. Do not continue to the experiment on a FIX LATER or NO ACTION
verdict unless I ask.

## 90.9 The experiment (Do - only on FIX NOW)
- Pick ONE change - normally the LCP element's discovery or priority, since that is where
  the leverage is.
- Build it as a local HTML variant exactly as in Stage 60.5: download the page, inject
  `<base href="{PAGE_URL}">`, one edit per variant, plus an unmodified baseline. Serve
  locally and measure with the same emulation. Do NOT mutate the page at runtime with
  `initScript` - it does not survive the reload `performance_start_trace` performs
  (verified in Stage 70), and you will "prove" the fix does nothing.
- 3 rounds, rotated A,B,A,B - never in blocks. Discard the first warm-up load. Compare
  medians against the larger of the two spreads.
- Compare local against LOCAL baseline only. A local copy loads a different set of
  third-party resources than production.
- **Check what you are actually measuring.** Removing an image makes LCP better by making
  the page emptier. If the variant no longer paints the same content, the number is not a
  result. Confirm the rendered output matches before recording anything.
- Verdicts: SUPPORTED / REGRESSION / INCONCLUSIVE.
- Expect asymmetry here: a discovery fix on the LCP element frequently moves LCP by a
  visible margin, while resizing twenty below-the-fold images moves no headline metric at
  all - it saves transfer and battery. Both are legitimate; report them as what they are and
  do not inflate the second into an LCP win.

## 90.10 OUTPUT - decisions, ROUTED BY STACK (Do)
One decision per asset. Do NOT edit yet. Apply the levers in order: dimension, format,
compression, loading strategy.

WordPress:
- `add_image_size()` to create the variant the layout actually needs, then regenerate
  thumbnails - adding the size alone does nothing for existing media.
- Check what the page builder emits. Builders routinely output the full-size file, ignore
  `sizes`, and apply `loading="lazy"` to everything including the LCP element. Their
  "exclude the first N images from lazy loading" setting counts DOM order, so a logo, a
  header icon and a decorative graphic can consume the allowance before the real hero is
  reached. Verify against the element the trace named in 90.2; if the heuristic misses it,
  a small must-use plugin that removes `loading` from that specific image is more reliable
  than raising N.
- Core emits `srcset` for images inserted through the media library; images hardcoded in
  templates or injected by a builder module frequently have none.
- Conversion at upload time is a plugin decision - ask me which one is installed before
  proposing a second one.

Next.js / React:
- `next/image` handles variants and formats, but the `sizes` prop still has to describe the
  real layout - without it you get the full-width assumption and the whole benefit is lost.
- The `priority` prop sets `fetchpriority="high"` AND emits a preload. Use it on the LCP
  element only. Audit every other usage: several priority images means several competing
  preloads.
- At scale, on-the-fly optimisation runs on MY server. On a large catalogue with real
  traffic this is a measurable CPU and memory cost on the Next.js process itself, and it can
  become the bottleneck rather than the images. If Stage 50 found the origin under pressure,
  connect the two findings: move the transformation to an image CDN or generate variants at
  upload time and serve them directly.
- A plain `<img>` is sometimes the better call for a single asset where the component's
  automatic preload is unwanted.

Astro:
- `<Image>` / `<Picture>` from `astro:assets` generate variants at build time - declare
  `widths`, `formats` and `sizes` to match the layout.
- Build-time generation means no per-request server cost, but also that remote images need
  explicit configuration to be processed at all - check whether they are being passed
  through untouched.

Any stack - the order that matters:
1. **RESIZE** to what the layout needs, and give the browser real `srcset` / `sizes`.
2. **CONVERT** to a modern format with a working fallback.
3. **COMPRESS** to the lowest quality that still looks right on MY images.
4. **DEFER** everything below the fold; keep the LCP element eager, discoverable and
   high priority.
5. **WARM THE CONNECTION** (`preconnect`, Stage 30) for the image origin if it is external -
   and reconsider serving the LCP image from my own origin, so the first visit does not pay
   DNS + TCP + TLS before the most important byte on the page.

## 90.11 Known traps (do not rediscover them the expensive way)
- **Lazy-loading the LCP element.** The most common self-inflicted regression on this stage.
  Lazy defers discovery until layout has run; on the one image that decides the metric, that
  is pure loss. Page builders and "optimisation" plugins do this by default.
- **Marking many images high priority.** Priority is zero-sum. Ten preloads for ten
  above-the-fold images delay the one that matters, plus the CSS and the JavaScript behind
  them.
- **`srcset` without `sizes`.** The browser assumes `100vw` and picks a file for the full
  viewport width. Silent, extremely common, and invisible unless you compare `currentSrc`
  against the rendered box.
- **`sizes="auto"` on the hero.** It requires `loading="lazy"`, which the LCP element must
  not have. It is a fix for the long tail, not for the hero.
- **Measuring at desktop width.** `srcset` picks a different file per viewport and DPR. A
  desktop run audits files my mobile users never download.
- **CSS `background-image` for anything that matters.** Invisible to the preload scanner,
  gated behind CSSOM and layout, no `alt`, no native lazy loading, not indexed as an image.
  Fine for decoration, wrong for content and wrong for the LCP element.
- **"We have an image CDN" as a conclusion.** Without correct `srcset` and `sizes` it serves
  one large transformed file to everyone. And it adds an origin whose connection cost is
  paid before the first byte - which can consume the entire saving on the hero image.
- **Treating a Lighthouse "properly size images" or "serve in next-gen formats" saving as a
  measurement.** It is a prediction (evidence rule, 00-index.md). Cite it as a hypothesis.
- **Reporting bytes saved as an LCP win.** Resizing twenty below-the-fold images saves real
  transfer and moves no headline metric. Say which one you are claiming.
- **`decoding="async"` presented as a headline finding.** It is a hint with a modest effect.
  Hygiene, not a result.
- **Deleting an image and calling the faster page an improvement.** Confirm the variant
  renders the same content before recording any number.
- **Inline SVG everywhere.** Adds its full weight to every HTML response and forfeits
  separate caching. If my document is unusually large, check this before anything else.
- **base64 as a request optimisation.** Inflates the bytes, moves them into an uncached
  document, and solves a problem HTTP/2 and HTTP/3 already solved.
- **An animated GIF left in place because it is "just a small animation".** It is a sequence
  of uncompressed frames; the video equivalent is typically an order of magnitude smaller.
- **A third-party video embed counted as one video.** It is a player: megabytes of
  JavaScript and CSS, most of it downloaded by users who never press play.
- **`initScript` does not survive the trace's own reload** (verified in Stage 70). Build
  variants as HTML files.

## Save results
- site-profile.md > ## Media: the inventory table with intrinsic vs rendered sizes, the LCP
  element analysis with its phase split, the wasted-pixel ranking, the SVG and base64
  payload, the video findings, BOTH priority verdicts, and the per-asset decision table.
- findings.md: dated entry with the per-run numbers, the experiment rounds and the
  reasoning.

## STOP line (end every run with this, verbatim intent)
"Media triage done. LCP element: <what it is>, delivered as <img / injected / background /
carousel>, dominant phase <discovery / transfer / render>, verdict <FIX NOW / FIX LATER /
NO ACTION>. Everything else: <n> images, <n> oversized, worst <asset> at <intrinsic> into
<rendered>, verdict <FIX NOW / FIX LATER / NO ACTION>. SVG/base64 in the document: <n KB>.
Video: <finding, or none>. Next: <the single next action>. Measured at <viewport> - srcset
picks different files at other widths. Missing width/height was recorded, not measured;
that belongs to the CLS stage."

## At the end
Check off in site-profile.md > ## Progress: [x] Stage 90 - images and video. Report BOTH
priority verdicts and the single next action - nothing more.
