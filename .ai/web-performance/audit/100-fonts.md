# Stage 100: Fonts - a chain, a budget, and one trade-off you cannot avoid

> Can be run STANDALONE or from the index (.ai/web-performance/audit/00-index.md).
> Behaviour: DO stage, ending in an EXPERIMENT. Unlike Stage 90 the inventory here is not
> the hard part - the browser hands you the loaded fonts in one call. The hard part is that
> every lever in this stage is ZERO-SUM against the same connection, so recommendations
> that each look sensible in isolation make the page slower together. See the budget rule.
> Read domain/URLs/stack from site-profile.md; if missing, ASK me and WAIT - do not guess.
> Input:  site-profile.md > ## Project, ## Page types, ## Render start (Stage 60),
>         ## Preconnect (Stage 30), ## Cache & CDN (Stage 40), ## Scripts (Stage 80),
>         ## Media (Stage 90) + the chrome-devtools MCP server + MY SOURCE CODE
>         (+ the WebPerf Snippets skill if installed - see 100.1)
> Output: site-profile.md > ## Fonts + findings.md, ending with ONE preload budget, a
>         per-file policy table that names MY font files, and a measured experiment.
>         Runs AFTER Stage 60 and AFTER Stage 90 (Stage 90 tells you whether the LCP
>         element is text - which decides this whole stage).
> Idempotent: if ## Fonts exists, update it, do not create a second one.

THIS RUNS AGAINST MY PROJECT, NOT AN EXAMPLE (read first):
The page you measure is MY page - the URL in site-profile.md > ## Project, for the page
type I am auditing. Never substitute a demo, a documentation example, or a site mentioned
anywhere in this package. If the URL, the page types or the stack are missing from the
profile, ASK me and WAIT.

Any figure that appears in this file is there to describe a MECHANISM or a threshold
defined by the tooling - not a measurement of my site. My project's numbers will be
different. Never copy a figure from this file into findings.md and never treat one as a
target.

The deliverable is not "self-host your fonts and use font-display: swap". It is: THIS font
file, at THIS weight, discovered after THIS many hops, needed (or not) by THIS element
above the fold, from THIS declaration in my repo - and the one policy that applies to it.

## SCOPE AND HARD BOUNDARY (read first - IMPORTANT)

A font is not a file, it is the END OF A CHAIN. Nothing about a font request is decided by
the font. It is decided by how many things had to happen before the browser knew the file
existed:

    HTML -> (external CSS) -> (@import: another CSS) -> CSSOM -> selector matched to a real
    element in the DOM -> only NOW does the font file enter the network queue

That is why the weight of a font file is almost never the interesting number, and why
`@font-face` declarations do not equal downloads: a stylesheet with 15 declarations
downloads only the faces whose selectors actually matched something. The cost lives in the
hops, not in the kilobytes.

**THE BUDGET RULE (this is the rule that makes or breaks this stage).**
Preload is zero-sum. It does not make a font faster in isolation; it moves that font ahead
of something else on the same connection - the CSS, the JavaScript, the LCP image. A page
that preloads every font it uses has not prioritised anything, it has just widened the
initial burst. This is not a theoretical caution: in the workshop demo, preloading ALL
fonts measured WORSE than preloading none (FCP 2.2 s vs 2.0 s on the same page), while
preloading exactly one critical font measured better than both (1.7 s). The stage therefore
must produce a NUMBER - how many font files may be preloaded on this page type - and that
number is usually 0 or 1, occasionally 2. If your report proposes preloading more than two
font files, you have almost certainly not done this stage; justify it with a measurement or
withdraw it.

**THE TRADE-OFF YOU CANNOT AVOID.**
While a font is in flight the browser must show something, and both options cost:
- show nothing (`block`, and `auto` in practice) -> invisible text, up to ~3 s of it;
- show the fallback (`swap`) -> visible text immediately, and a layout shift when the real
  font arrives, because the two fonts have different metrics.
There is no third option that is free. `optional` buys near-zero CLS by accepting that the
custom font may not appear at all on the first visit. This stage must state which cost my
project is choosing and why - not recommend `swap` reflexively. And when it chooses `swap`,
metric matching (`size-adjust`, `ascent-override`, `descent-override`) is the price of that
choice, not an optional extra.

Two further boundaries:

- **This stage does not own CLS, but it is the largest single contributor to it.** Measure
  the shift caused by the font swap here, because only this stage knows when the swap
  happens; record it and route the total CLS budget to the CLS stage.
- **Legibility and licensing outrank bytes.** A subset that drops Polish diacritics, or a
  modification the licence forbids, is not an optimisation. Check both before proposing a
  change to a file.

## 100.0 Read what earlier stages already found (Do, no new measurement yet)
- From site-profile.md > ## Media (Stage 90): **is the LCP element text or an image?** This
  decides the whole stage. If the LCP element is a heading rendered in a custom font, that
  font is the one candidate for the preload budget. If the LCP element is an image, the
  budget most likely belongs to that image and the answer here may legitimately be "preload
  nothing".
- From site-profile.md > ## Render start (Stage 60): Start Render / FCP for the slow page
  type, and which render-blocking resources are already deferred. A font that arrives after
  Start Render is not delaying the first paint - it is causing a repaint and a shift, which
  is a different finding with a different fix.
- From site-profile.md > ## Preconnect (Stage 30): whether an external font origin is
  already warmed. If fonts come from a third party, the first visit pays DNS + TCP + TLS
  before the first byte of CSS, let alone the font.
- From site-profile.md > ## Cache & CDN (Stage 40): the TTL on static assets. Fonts are the
  most immutable asset class on the page - a short TTL here is a pure loss (see 100.7).
- From site-profile.md > ## Scripts (Stage 80): whether a script injects the stylesheet that
  contains the `@font-face` rules. If it does, add the script's own download and execution
  to every chain you measure in 100.2.
- GATE: if Stage 50 says TTFB is the wall, or Stage 90 found a badly delivered LCP image,
  note it and carry it into 100.8. Run the stage anyway - the inventory is cheap - but the
  verdict must say out loud whether font work is the next thing to do or noise next to what
  is already on the table.

## 100.1 Inventory - what actually loaded, and what is actually used (Do)
Everything here runs through the chrome-devtools MCP server.

- Call `emulate` ONCE: mobile viewport ~412x765x2.6 (mobile, touch), "Fast 4G", CPU
  throttling 4x. Do not change it for the rest of the stage. State the viewport next to
  every number you record.
- `navigate_page` to the page type you are auditing, then `performance_start_trace` with
  `reload: true` and `autoStop: true`.
- Also `curl` the page and keep the raw server HTML. You need it in 100.2 to tell an
  `@font-face` that ships in the document from one that arrives inside a stylesheet.

**PRIMARY PATH - use the WebPerf Snippets skill if it is installed.** This stage does not
ask you to re-implement a cross-reference that already exists and is maintained. If the
`webperf` / `webperf-loading` skill (nucliweb, github.com/nucliweb/webperf-snippets) is
available, invoke it - it ships the snippet locally as
`scripts/Fonts-Preloaded-Loaded-and-used-above-the-fold.js`, version-pinned with a checksum,
so there is no network fetch and no drift. Its "Font Loading Optimization" workflow runs
three snippets and you want all three:
- `Fonts-Preloaded-Loaded-and-used-above-the-fold.js` - the font audit below;
- `Resource-Hints-Validation.js` - checks preload correctness, and flags a font preload
  without `crossorigin` as an error. This does most of the mechanical work of 100.4;
- `Find-render-blocking-resources.js` - tells you whether the stylesheet carrying the
  `@font-face` is itself render-blocking, which feeds 100.2.

READ 100.4 BEFORE ACTING ON THE SKILL'S RECOMMENDATIONS. Its decision tree says "fonts used
but not preloaded -> recommend adding preload", and its resource-hints check treats 5-6
preloads as the acceptable ceiling. Both are reasonable defaults for a general audit and
both are WRONG for this stage: the skill does not know your preload budget, and this stage
exists to produce one. Use the skill for the inventory and the mechanical checks; take the
preload decision from 100.4, not from the skill.

**FALLBACK 1 - if the skill is not installed**, take the snippet from the site:

  "Fonts Preloaded, Loaded, and Used Above The Fold" - section Loading
  https://webperf-snippets.nucliweb.net/Loading/Fonts-Preloaded-Loaded-and-used-above-the-fold

Fetch its source and run it through `evaluate_script` on the page you are auditing. It
returns four blocks, and between them they are most of what 100.1 needs:

1. **Preloaded fonts** - filename, type, third-party or not, and whether `crossorigin` is
   present. Carry the crossorigin column straight into 100.4.
2. **Loaded fonts** - family, weight, style and the `font-display` value in effect. Carry
   `font-display` into 100.5.
3. **Fonts used above the fold** - family, weight, style and the number of elements using
   each. This is the input to 100.3.
4. **Potential issues** - preloaded but unused above the fold, used but not preloaded, and
   missing `crossorigin`.

Treat block 4 as a set of LEADS, not as the verdict. It tells you a face is used but not
preloaded; it does not know your preload budget, so "used but unpreloaded" is not
automatically something to fix - on most pages the correct action is to preload none of them
(see 100.4). Confirm a missing `crossorigin` by finding the duplicate request on the
waterfall, not by trusting the flag.

**What the snippet does NOT give you**, and what you must still take from the trace and the
network list - this is where the rest of the stage lives:
- transfer size and origin per file;
- `unicode-range` per face;
- WHEN each request started, which is the whole of 100.2.

**FALLBACK 2 - if you can neither run the skill nor fetch the snippet**, collect the same
two things yourself.
`document.fonts` is authoritative for (a): it reports the faces the browser actually
instantiated, not what the CSS declares.

```js
// (a) One row per face the browser actually loaded.
[...document.fonts].filter(f => f.status === 'loaded').map(f => ({
  family: f.family, weight: f.weight, style: f.style,
  stretch: f.stretch, unicodeRange: f.unicodeRange,
  display: f.display,                 // the font-display value in effect
}));
```

```js
// (b) Families genuinely used above the fold, with how many elements use each.
const used = new Map();
for (const el of document.body.querySelectorAll('*')) {
  const r = el.getBoundingClientRect();
  if (r.top >= innerHeight || r.bottom <= 0 || !el.textContent.trim()) continue;
  const cs = getComputedStyle(el);
  const key = `${cs.fontFamily.split(',')[0].replace(/["']/g,'').trim()}|${cs.fontWeight}|${cs.fontStyle}`;
  used.set(key, (used.get(key) || 0) + 1);
}
[...used].map(([k, n]) => ({ face: k, elements: n }));
```

Then read the preload links out of the served HTML yourself.

- Whichever path you used, SAY SO in the report - skill, fetched snippet, or hand-rolled.
- Cross-reference the network requests from the trace so every loaded face maps to a real
  file with a real transfer size and origin.

Build ONE row per FONT FILE (not per declaration):

| file | family / weight / style | origin | format | transfer | unicode-range | font-display | preloaded? | used above the fold? | elements using it |
|---|---|---|---|---|---|---|---|---|---|

Three things to flag immediately, before any analysis:
- a face that **loaded but is used by zero elements** - a download for nothing;
- a face **used above the fold but not preloaded**, while some other face IS preloaded -
  the budget is being spent on the wrong file;
- the count of `@font-face` **declarations** in my CSS versus the count of **loaded faces**.
  A large gap is normal and healthy (declarations are free, downloads are not) - but state
  both numbers so nobody "optimises" declarations that never cost anything.

## 100.2 The discovery chain - count the hops per file (Do)
This is the core measurement of the stage. For EVERY font file from 100.1, reconstruct what
had to happen before the request started, and read the start time from the waterfall.

Classify each file into exactly one chain:

- **Chain A - `@font-face` inline in the document `<head>`, file preloaded.** Shortest
  possible. The preload scanner sees the file while the HTML is still being parsed.
- **Chain B - `@font-face` inline in the document, no preload.** The declaration is found
  early, but the request still waits for CSSOM, selector matching and layout.
- **Chain C - `@font-face` in an external stylesheet on my origin.** Add: download and parse
  that stylesheet first.
- **Chain D - `@font-face` in an external stylesheet on a third-party origin** (the standard
  Google Fonts embed). Add: DNS + TCP + TLS for that origin, then the stylesheet, and only
  then does the browser learn which font files exist - which live on yet another origin,
  needing its own connection.
- **Chain E - the stylesheet is reached through `@import` from another stylesheet.** The
  worst case: two serialised stylesheet downloads before the `@font-face` is even parsed.
  This is the single deepest chain in common use and it is invisible in the HTML.
- **Chain F - the stylesheet is injected by JavaScript** (Stage 80 will have named the
  script). Add the script's download, parse and execution to the chain.

For each file record: **chain letter, number of serialised network hops before the request,
request start time, and the delta from Start Render.** Then state the obvious consequence:

- request starts **before** Start Render -> this font is competing with the critical path.
  Whether that is good or bad depends entirely on whether it is the critical font (100.3).
- request starts **after** Start Render -> this font is not delaying the first paint. Its
  cost is a repaint and a layout shift, not a slow start. Do not report it as a render-start
  problem.

Sort the table by hops, not by transfer size. The heaviest file at chain A is usually less
of a problem than a small file at chain E.

## 100.3 Which font is actually critical (Do - this is the question the budget answers)
At most a small number of faces are needed for the first paint. Find them, do not assume.

- Take the faces used above the fold from 100.1(b), ranked by element count and by whether
  they render the LCP element (from Stage 90).
- Answer explicitly: **which single face renders the LCP text?** If Stage 90 found the LCP
  element is an image, say so and note that the font budget probably belongs to the image.
- Distinguish three tiers and put every loaded face in one:
  - **Critical** - renders the LCP element or the primary above-the-fold copy. Candidate for
    the preload budget.
  - **Above the fold, secondary** - real text in the viewport, but not the metric. Eager, no
    preload.
  - **Below the fold / interaction-only / icon font** - must not compete with anything.
- Check the failure mode that appears in almost every page builder: a **theme that preloads
  one weight globally** while individual page templates render a different weight. On the
  home page you then preload a weight the page never uses, and the weight it does use starts
  at chain C. Verify per page type, not once for the site.

## 100.4 The preload budget - produce the number (Do)
Now issue the constraint the rest of the stage lives under.

- State the budget: **N font files may be preloaded on this page type**, where N is derived
  from 100.3, not from taste. Normally 0 or 1. Two only when the LCP text genuinely uses two
  faces (a display weight for the heading and a body weight immediately below it).
- List every file currently preloaded and mark each KEEP / REMOVE against the budget.
- Where preload is kept, check it is correct, because a wrong preload is worse than none -
  it downloads a file the CSS will then not use:
  - `as="font"` and `type="font/woff2"` present;
  - `crossorigin` present (font requests are CORS-mode even from my own origin; without it
    the file is fetched twice). The snippet in 100.1 already flags this per preload - confirm
    it by finding the duplicate request on the waterfall before reporting it;
  - the preloaded URL is byte-identical to the URL the `@font-face` will request - watch for
    cache-busting query strings and for a CDN rewriting one of them.
- **If you ran the skill in 100.1, this is where you overrule it.** Its decision tree
  recommends adding a preload for every face used above the fold but not preloaded, and its
  resource-hints check only complains past 5-6 preloads. Applied literally on a page with
  four faces above the fold, that produces exactly the configuration the workshop measured
  as SLOWER than preloading nothing. The skill reports a mismatch; the budget decides what
  to do about it. Record both, and say in the report where you departed from the tool.
- Say plainly what preload does NOT do: it does not shorten the chain for the OTHER faces,
  it does not change `font-display` behaviour, and it does not make a third-party origin
  local. Moving `@font-face` into the document is worth a few percent; preload is the lever.
  Do not report the former as the main win.

## 100.5 font-display and the fallback - choose a cost, then pay it properly (Do)
- Record the `font-display` in effect per face (100.1 reports it) and whether it was set
  deliberately or inherited from a provider default.
- State what each value means for MY page, in terms of the two costs:
  - **`auto`** - the browser decides, and in practice behaves like `block`. Record it as an
    unmade decision, not as a setting.
  - **`block`** - invisible text for up to ~3 s. Defensible for an icon font, where a
    fallback glyph is worse than a reserved empty box. Wrong for body copy.
  - **`swap`** - text immediately in the fallback, then a swap and a shift. The usual right
    answer for content, and the reason this stage owes the CLS stage a number.
  - **`fallback`** - a short block window (~100 ms), then fallback, then a swap only if the
    font arrives within a few seconds. In practice inherits the worst of both on a first
    visit.
  - **`optional`** - a short block window and then the browser may stay on the fallback for
    the whole visit. The only value that gets CLS near zero. Pairs well with preload: the
    preload gives the font a real chance to win the short window, the `optional` guarantees
    no shift if it does not.
- **Measure the swap, do not assume it.** With the mobile emulation still applied, record the
  layout shift attributable to the font swap: run the trace, find the shift events, and check
  whether their timestamps coincide with the font load completing. Report the shift value and
  the moment. This is the number that decides whether metric matching is worth the effort.
- **Metric matching (only if `swap` is the choice).** Compare the fallback stack against the
  custom face and record the overrides needed: `size-adjust`, `ascent-override`,
  `descent-override`, `line-gap-override`. The Fallback Font Generator produces these; frameworks
  often generate them automatically (Next.js does for its font loader). Record whether my
  project already has them - a `swap` with no metric-matched fallback is an accepted layout
  shift that nobody decided to accept.
- Note where the framework already solved this so you do not propose work twice. If the
  overrides exist, verify they are actually applied to the fallback stack in the computed
  styles rather than merely declared.

## 100.6 Bytes - the second-order lever, in a fixed order (Do)
Only after 100.4 and 100.5. These reduce transfer; they rarely move a headline metric on
their own, and this stage must not present them as if they did.

Order: **fewer faces -> subset -> format -> variable font**.

- **Fewer faces.** List every weight and style downloaded and how many elements use each.
  Weights loaded "because the family has them" are the biggest single saving here and the
  only one that needs no tooling.
- **Variable font.** If my design genuinely uses many weights, compare the sum of the static
  files against the single variable file. Frequently the variable file is smaller than three
  or four statics - but it is not automatic, it depends on the family and on the axes it
  carries. Report the actual comparison for MY files, never a rule of thumb.
- **Subsetting and `unicode-range`.** Record the character coverage each file carries versus
  what my content needs. For Polish content the relevant subset is normally latin +
  latin-ext; a subset that drops the diacritics is a bug, not an optimisation. Record whether
  `unicode-range` is used to split the family so the browser only downloads the ranges a page
  actually contains - this is how Google Fonts works internally and it is worth reproducing
  when self-hosting. Transfonter produces subsets and the accompanying CSS; Wakamai Fondue
  shows what is inside a file before you decide what to cut.
  - **Licence gate:** commercial fonts frequently forbid modifying the file. Ask me whether
    my faces are licensed for modification BEFORE proposing a subset. Note the distinction:
    generating a subset for web delivery and altering the typeface are not the same act, but
    the licence, not this document, decides.
  - **THE SUBSET PASS (Do - this stage produces a real file, not a recommendation).** Once
    the licence gate is clear, actually run it on the heaviest self-hosted face:
    1. Open the source file in Wakamai Fondue (wakamaifondue.com) and record what is inside:
       glyph count, character sets, features, axes. Decide from that what can go, and say so
       explicitly - "drop the Cyrillic and Greek ranges", not "subset it".
    2. Regenerate through Transfonter (transfonter.org): subset latin + latin-ext for Polish
       content, WOFF2 only, and the `font-display` value chosen in 100.5. Keep the generated
       CSS - the `unicode-range` it emits is part of the deliverable.
    3. Record BEFORE and AFTER in bytes, per file and as a total for the page type.
    4. **Verify the rendering, not the size.** Swap the file in locally and check the Polish
       diacritics on real content, plus any character the design depends on (currency,
       arrows, ligatures, the tabular figures in a price table). A subset that renders a
       tofu box has not made the file smaller, it has broken it. State that you checked.
    5. Report this as TRANSFER SAVED, with the header metrics from before and after next to
       it. On most pages this moves no headline metric and that is the expected result - it
       saves bandwidth, battery and the user's data plan. Do not inflate it into an LCP win.
- **Format.** WOFF2 or it is a finding. Anything served as TTF, OTF or WOFF is carrying
  avoidable bytes; TTF in particular is frequently several times the size of the same face as
  WOFF2. There is no meaningful browser left that needs a fallback format.
- **Icon fonts.** If an icon font is present, count how many glyphs of it the page actually
  uses. A thousand-glyph file for six icons is a routine finding; the fix is usually inline
  SVG for the few above the fold and a subset or SVG sprite for the rest. Route the SVG side
  of it to Stage 90 and record the decision here.

## 100.7 Origin and caching (Do)
- **Self-host unless there is a reason not to.** A third-party font origin costs DNS + TCP +
  TLS on the first visit, before the browser has even read which files it needs, and the
  cache is not shared across sites. Record whether my fonts are first-party today.
- If the answer is Google Fonts and it cannot be changed in the template, record
  **Cloudflare Fonts** as the available mitigation: it rewrites the document at the edge so
  the faces are served from my own origin. Record it accurately in the report - it removes
  the third-party connection, it does NOT reduce the number of faces I declare, it does NOT
  subset for me, and it does NOT choose what to preload. It moves the origin; the rest of
  this stage still applies.
- **Cache TTL.** Fonts are immutable and content-hashed or version-named in every modern
  build. A long immutable TTL (a year is the normal answer) is correct. If Stage 40 recorded
  a short TTL on static assets, this is the asset class where it costs the most on repeat
  visits. Note also that with a third-party provider the TTL is theirs, not mine - which is
  a reason to self-host, and worth stating when a Lighthouse cache audit flags it and there
  is nothing to fix.

## 100.8 The priority verdict (Do - STOP and show me)
Before proposing a single fix, answer in one short block:

- **the chain**: the deepest chain letter on this page type, which file is at the end of it,
  and how many hops that is;
- **the critical face**: which face renders the LCP element (or: the LCP element is an image,
  from Stage 90);
- **the budget**: N = <0 / 1 / 2>, which file gets it, and which currently-preloaded files
  must lose it;
- **the trade-off**: which `font-display` cost this project is choosing, the shift measured
  in 100.5, and whether metric matching exists;
- **the bytes**: total font transfer, how many faces, how many unused above the fold;
- **how this compares** to what Stages 60, 80 and 90 found - is the font chain a visible
  share of this page's problem, or noise next to a 2-second TTFB or a badly delivered LCP
  image?
- verdict: **FIX NOW** / **FIX LATER** / **NO ACTION**, with the single highest-value change.

Show me this block and wait. Do not continue to the experiment on a FIX LATER or NO ACTION
verdict unless I ask.

## 100.9 The experiment (Do - only on FIX NOW)
The budget claim is exactly the kind of claim that must be measured, because it is
counter-intuitive and because both directions are plausible on any given page.

- Build the variants as local HTML exactly as in Stage 60.5: download the page, inject
  `<base href="{PAGE_URL}">`, one edit per variant, plus an unmodified baseline. Serve
  locally and measure with the same emulation. Do NOT mutate the page at runtime with
  `initScript` - it does not survive the reload `performance_start_trace` performs (verified
  in Stage 70).
- Run at least these three, because two of them are the point of the stage:
  1. baseline, untouched;
  2. **preload the critical face only** (everything else unchanged);
  3. **preload every face** the page loads.
  Variant 3 is not padding. It is the control that shows preload is zero-sum, and on a page
  with several faces it frequently measures worse than the baseline.
- Optionally add: `@font-face` moved from the external stylesheet into the document head, so
  we can see how small that effect is next to preload.
- 3 rounds, rotated A,B,C,A,B,C - never in blocks. Discard the first warm-up load. Compare
  medians against the larger of the spreads. Record FCP / Start Render / LCP **and CLS** for
  every variant - a font change that improves LCP while adding a shift has not necessarily
  won, and this is the stage where that happens.
- Compare local against LOCAL baseline only.
- Verdicts: SUPPORTED / REGRESSION / INCONCLUSIVE.

## 100.10 OUTPUT - policy per file, ROUTED BY STACK (Do)
One policy per font FILE: preload / eager / defer / drop, plus its `font-display`. Do NOT
edit yet.

WordPress:
- Check the theme and page-builder settings FIRST. Modern builders have a native
  "host Google Fonts locally" switch and a `font-display` setting; using them beats writing
  code.
- Builders typically preload a fixed set of faces globally, chosen once for the whole site.
  When the LCP heading uses a different weight per template, that global preload is spending
  the budget on the wrong file on most page types. The reliable fix is a small **must-use
  plugin** that emits the preload link for the face this specific template actually renders,
  and nothing else. This is exactly the kind of narrow, well-specified job an AI assistant
  writes correctly in one pass - describe the routing rules, then verify the emitted `<link>`
  in the served HTML rather than trusting the code.
- Icon fonts arriving from a builder (Font Awesome and friends) are usually loaded on every
  page whether or not an icon appears. Check before subsetting anything else.

Next.js / React:
- `next/font` self-hosts the files at build time, generates the metric-matched fallback and
  applies the CSS - which removes most of this stage's manual work. Verify it is actually in
  use rather than a raw `<link>` to a third party.
- It preloads the faces it considers used. Confirm what it emitted in the served HTML against
  the budget from 100.4; a layout that pulls in several font loaders will preload several
  faces.
- Verify the generated fallback overrides are present in the computed styles, not merely in
  the generated CSS.

Astro:
- Fonts are usually plain static assets: self-hosting, subsetting and the preload link are
  explicit and under my control, which is an advantage here. Check that the `<link
  rel="preload">` and the `@font-face` reference the identical URL after the build hashes it.

Any stack - the order that matters:
1. **CUT** faces nothing renders, and weights the design does not use.
2. **SHORTEN THE CHAIN**: no `@import`, first-party origin, `@font-face` reachable from the
   document as early as the stack allows.
3. **SPEND THE BUDGET**: preload the critical face, and only it, correctly (`crossorigin`,
   matching URL).
4. **CHOOSE THE COST**: `font-display` per face, and pay for `swap` with a metric-matched
   fallback.
5. **THEN BYTES**: subset, WOFF2, variable font where it genuinely wins.
6. **CACHE**: immutable, long TTL, on my own origin.

## 100.11 Known traps (do not rediscover them the expensive way)
- **Preloading every font.** The headline trap of this stage. Preload is zero-sum; preloading
  all faces widens the initial burst and frequently measures worse than preloading none.
  Measure it (100.9, variant 3) before arguing about it.
- **Taking the preload advice of a general-purpose audit tool at face value.** The WebPerf
  Snippets skill, Lighthouse and most audit tooling flag "used above the fold but not
  preloaded" as an opportunity and tolerate 5-6 preloads. That is a sane default for a tool
  that cannot know which face decides your LCP - and it is how a page ends up preloading
  four faces and measuring worse than preloading none. The tool finds the mismatch; 100.4
  decides what to do with it.
- **A preload without `crossorigin`.** Font requests are CORS-mode even same-origin. Without
  it the browser fetches the file a second time and the preload has cost you bandwidth for
  nothing. Check the waterfall for the duplicate rather than reading the markup.
- **A preload whose URL does not match the `@font-face` URL.** Query strings, hashed
  filenames and CDN rewrites all cause it. Same symptom: two downloads.
- **Counting `@font-face` declarations as downloads.** They are not. The browser downloads a
  face when a selector using it matches a real element. Fifteen declarations can be zero
  requests. Do not "optimise" declarations that cost nothing while ignoring the two files
  that actually load.
- **`@import` in CSS.** Serialises two stylesheet downloads before the `@font-face` is even
  parsed, and is invisible in the HTML. The deepest chain in common use.
- **A CSS `@font-face` block treated as the fix.** Moving the declaration into the document
  head is worth a few percent. Preload is the lever. Do not report the small one as the win.
- **`swap` with no metric-matched fallback.** That is a decision to accept a layout shift,
  taken by default rather than on purpose. Measure the shift and say so.
- **`auto` reported as a setting.** It is the absence of a decision, and in practice behaves
  like `block`.
- **`block` on body copy.** Up to three seconds of invisible text. It is defensible for an
  icon font and almost nothing else.
- **Subsetting away Polish diacritics.** A file that cannot render the content is not a
  smaller file, it is a broken one. Latin-ext, and check the rendering.
- **Subsetting a font whose licence forbids modification.** Ask before proposing it.
- **A variable font assumed to be smaller.** Often true, not always. Compare the actual files.
- **Serving TTF or OTF on the web.** Avoidable bytes with no compatibility argument left.
- **An icon font kept for six icons.** Count the glyphs actually used before optimising
  anything else on the page.
- **"We moved to Cloudflare Fonts, fonts are done."** It moves the origin. It does not reduce
  the faces, does not subset, does not choose the preload. The rest of this stage still
  applies.
- **A Lighthouse font-cache warning on a third-party provider.** The TTL is theirs. The fix is
  self-hosting, not a header you cannot set.
- **Auditing at desktop width.** The above-the-fold set differs per viewport, so the critical
  face can differ too. Measure at the emulation the rest of the audit used.
- **`initScript` does not survive the trace's own reload** (verified in Stage 70). Build
  variants as HTML files.

## Save results
- site-profile.md > ## Fonts: the per-file inventory with chains and hop counts, the critical
  face, the preload budget with KEEP/REMOVE per file, the `font-display` decision and the
  measured swap shift, the byte findings, the verdict, and the per-file policy table.
- findings.md: dated entry with the per-run numbers, the three experiment variants and the
  reasoning.

## STOP line (end every run with this, verbatim intent)
"Font triage done. Deepest chain: <letter> (<n> hops) on <file>. Critical face:
<family/weight> rendering <LCP element / above-fold copy>, or <LCP is an image - see Stage
90>. Preload budget: <N>, granted to <file>, removed from <files>. font-display: <value>
chosen, measured swap shift <value> at <time>, metric-matched fallback <present / missing>.
Bytes: <n> faces, <n KB> total, <n> unused above the fold. Verdict <FIX NOW / FIX LATER / NO
ACTION>. Next: <the single next action>. Measured at <viewport>."

## At the end
Check off in site-profile.md > ## Progress: [x] Stage 100 - fonts. Report the budget, the
trade-off chosen and the single next action - nothing more.
