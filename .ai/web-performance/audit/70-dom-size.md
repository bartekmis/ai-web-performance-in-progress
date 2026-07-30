# Stage 70: DOM size - the cost of style and layout recalculation, and who pays it

> Can be run STANDALONE or from the index (.ai/web-performance/audit/00-index.md).
> Behaviour: DO stage. Like Stage 60 you do not read a value and stop - you measure, you
> attribute the cost to a named owner, and you prove the fix by re-measuring. Unlike
> Stage 60 the experiment is cheaper: the browser hands you the attribution.
> Read domain/URLs/stack from site-profile.md; if missing, ASK me and WAIT - do not guess.
> Input:  site-profile.md > ## Project, ## Page types, ## Render start (Stage 60 verdict)
>         + the chrome-devtools MCP server + MY SOURCE CODE (the repo you are running in)
> Output: site-profile.md > ## DOM size + findings.md, ending with a priority verdict and
>         a stack-routed fix list that names MY files. Runs AFTER Stage 60.
> Idempotent: if ## DOM size exists, update it, do not create a second one.

THIS RUNS AGAINST MY PROJECT, NOT AN EXAMPLE (read first):
The page you measure is MY page - the URL in site-profile.md > ## Project, for the page
type I am auditing. Never substitute a demo, a documentation example, or a site mentioned
anywhere in this package. If the URL, the page types or the stack are missing from the
profile, ASK me and WAIT.

Every number written in this file - element counts, milliseconds, script names - comes from
an UNRELATED reference page and exists only to calibrate the METHOD: to show what the tools
return, how much the readings move between runs, and which traps produce a false result.
My project's numbers will be different. Never copy a reference figure into findings.md,
never treat one as a target or a threshold, and never conclude anything about my page from
a number that was measured on someone else's.

The deliverable is not "your DOM is deep". It is: this specific chain / duplicate / list in
MY codebase, in THIS file, costs THIS much, and here is the change. A finding that does not
end at a file in my repo (or an explicit "this markup is not mine - it comes from X") is
half a finding.

SCOPE AND HARD BOUNDARY (read first - IMPORTANT):
This stage answers ONE question: does the size and shape of the DOM cost measurable main-
thread time on this page, and if so, which part of the structure and which script is
responsible?

Two boundaries, both new:

1. **Node count is NOT the criterion, and has not been for a while.** The old Lighthouse
   rule (warn above ~800 body nodes, error above ~1400) is gone. Today the audit fires on
   a LARGE LAYOUT OR STYLE RECALCULATION - Chrome flags updates that take roughly 40 ms or
   more. So a page with 5000 nodes and no heavy recalculation is fine, and a page with 600
   nodes doing layout on all of them inside a third-party script is not. Never report node
   count alone as a problem. If you catch yourself writing "your DOM has N elements, that
   is too many", stop - you have not measured anything yet.

2. **This stage can find a real cost that is not worth fixing.** A 50 ms layout on a page
   whose LCP is 4.5 s is a rounding error, and the fix competes for the same afternoon as
   the LCP work. So this stage ends with an explicit PRIORITY verdict, not just a fix list.
   Reporting a 50 ms win as if it mattered, while a 3 s problem sits in the same profile,
   is the failure mode here.

## 70.0 Read what earlier stages already found (Do, no new measurement yet)
- From site-profile.md > ## Render start: Start Render / FCP / LCP for the slow page type,
  and which resources Stage 60 already deleted or deferred.
- From site-profile.md > ## Backend: the Stage 50 class.
- GATE: if LCP is still measured in seconds, or Stage 50 says TTFB is the wall, note it and
  carry it into 70.4. You may still run this stage - the data is cheap - but the verdict
  must say out loud that DOM work is not the next thing to do.

## 70.1 The measurement (Do)
Everything here runs through the chrome-devtools MCP server.

- Call `emulate` ONCE: mobile viewport ~412x765x2.6 (mobile, touch), "Fast 4G", CPU
  throttling 4x. Do not change it again for the rest of the stage. Every number below -
  including which elements count as hidden and which sit below the fold - depends on the
  viewport, so a mid-stage change invalidates the comparison.
- `navigate_page` to the page type you are auditing.
- `performance_start_trace` with `reload: true` and `autoStop: true`. This reloads and
  stops on its own; you get back a list of insight sets and the insights available in each.
- Look for an insight named **`DOMSize`** in the list.
  - If it is ABSENT: Chrome found no layout or style recalculation big enough to flag.
    Record that, run 70.3 anyway (structure data is useful for the fix list even when the
    cost is currently zero), and skip the experiment. Do NOT manufacture a problem.
  - If it is present: `performance_analyze_insight` with `insightName: "DOMSize"` and the
    insight set id from the trace summary.
- The DOMSize insight returns, verbatim:
  - `Total elements: N`
  - `DOM depth: N nodes, starting with element '<TAG class=...>' (node id: N)` - the element
    that STARTS the deepest chain. This is your first suspect and usually the most useful
    single line in the whole stage.
  - `Most children: N, for parent '<TAG>' (node id: N)`
  - `Large layout updates/style calculations:` one line per event, each
    `Duration: N ms, with X of Y nodes needing layout`
  - `Estimated savings:` - for this insight it is normally `none`. Good. There is no
    prediction here to mistake for a measurement; do not invent one.
- Repeat the trace 3 times and record the layout duration from each, then work out YOUR
  spread - that is the noise floor for my page, and it is the only bar that counts here.
  [Reference calibration, measured on an unrelated page - do not expect these values]: the
  structural statistics came back IDENTICAL on every run while the layout duration moved by
  ~4 ms between runs. Expect the same PATTERN on my page: structure stable, timing noisy.
  Treat a difference under my measured spread as noise, not as a result.

## 70.2 Attribution - who triggers the recalculation (Do)
A layout duration on its own tells you the cost, not the owner. The owner is in a
different insight, and this step is why the audit beats reading the panel by hand.

- If the trace lists an insight named **`ForcedReflow`**, call `performance_analyze_insight`
  on it. It returns:
  - the top function call that caused the reflow, as `name @ URL:line:column`,
  - `Total reflow time: N ms`,
  - a list of call frames that trigger reflow, each with its own ms.
- Read the URLs. A first-party bundle and a third-party tag are different problems with
  different fixes, and the URL settles it without guessing. Match the URL against my repo
  and my dependencies: a path under my build output is my code, anything on a vendor domain
  is not. [Reference example, unrelated page: the frames came back as two first-party
  bundle chunks plus an analytics vendor's domain - i.e. a tag reading geometry during
  load. Your page will name different scripts.]
- A forced reflow means JavaScript asked for a geometric property (`offsetWidth`,
  `getBoundingClientRect`, `offsetTop`...) after the DOM or styles had been invalidated, so
  the browser had to lay out synchronously to answer. The DEEPER and WIDER the tree, the
  more that forced answer costs. This is the actual mechanism linking "big DOM" to "slow
  page", and it is worth stating in the report in one sentence.
- So the finding is always a PAIR: the structure that makes layout expensive (70.1, 70.3)
  and the code that forces layout to happen (70.2). A fix can attack either side. Removing
  the caller is usually faster to ship; flattening the structure helps every future caller.
- If `ForcedReflow` is absent but `DOMSize` is present, the recalculation came from ordinary
  rendering rather than a synchronous query. Say so - the fix is then structural only.

## 70.3 Structure interrogation (Do - the actionable part)
The insight tells you the tree is expensive. It does not tell you which part to delete.
Run this with `evaluate_script`, on the SAME emulated viewport, after load:

```js
() => {
  const body = document.body;
  const els = [...body.querySelectorAll('*')].filter(e => !e.closest('svg') || e.tagName === 'SVG');
  const label = e => e.tagName.toLowerCase() + (e.id ? '#' + e.id : '') +
    (typeof e.className === 'string' && e.className.trim() ? '.' + e.className.trim().split(/\s+/)[0] : '');
  const path = e => { const p = []; let n = e; while (n && n.nodeType === 1 && p.length < 40) { p.unshift(label(n)); n = n.parentElement; } return p.join(' > '); };
  const depth = e => { let d = 0, n = e; while (n.parentElement) { d++; n = n.parentElement; } return d; };
  let deepest = null, maxd = -1;
  for (const e of els) { const d = depth(e); if (d > maxd) { maxd = d; deepest = e; } }
  const generic = els.filter(e => ['DIV', 'SPAN'].includes(e.tagName)).length;
  const wrappers = els.filter(e => e.children.length === 1 && ['DIV', 'SPAN'].includes(e.tagName)
    && !e.getAttribute('role') && !e.id
    && [...e.childNodes].every(n => n.nodeType === 1 || (n.nodeType === 3 && !n.textContent.trim())));
  const hiddenRoots = []; let hiddenCount = 0;
  for (const e of els) {
    if (getComputedStyle(e).display === 'none' && !hiddenRoots.some(r => r.contains(e))) {
      hiddenRoots.push(e); hiddenCount += 1 + e.querySelectorAll('*').length;
    }
  }
  const vh = innerHeight;
  const below = els.filter(e => e.getBoundingClientRect().top >= vh);
  const repeated = els.filter(e => e.children.length >= 5).map(e => {
    const cls = [...e.children].map(c => c.className || c.tagName);
    return { sel: label(e), items: e.children.length, subtree: e.querySelectorAll('*').length, uniform: new Set(cls).size <= 2 };
  }).filter(x => x.uniform).sort((a, b) => b.subtree - a.subtree).slice(0, 5);
  return {
    totalInBody: body.querySelectorAll('*').length, countedIgnoringSvgInternals: els.length,
    maxDepth: maxd, deepestPath: path(deepest), over12deep: els.filter(e => depth(e) > 12).length,
    divSpanPct: Math.round(generic / els.length * 100),
    removableWrappers: wrappers.length, wrapperSample: wrappers.slice(0, 3).map(path),
    hiddenSubtrees: hiddenRoots.length, elementsInHidden: hiddenCount,
    hiddenSample: hiddenRoots.sort((a, b) => b.querySelectorAll('*').length - a.querySelectorAll('*').length)
      .slice(0, 4).map(e => label(e) + ' [' + (1 + e.querySelectorAll('*').length) + ' el]'),
    belowFold: below.length, belowFoldPct: Math.round(below.length / els.length * 100),
    biggestSubtrees: els.filter(e => e.children.length > 2 && e.tagName !== 'SVG')
      .map(e => ({ sel: label(e), children: e.children.length, subtree: e.querySelectorAll('*').length }))
      .sort((a, b) => b.subtree - a.subtree).slice(0, 6),
    repeatedLists: repeated, viewport: innerWidth + 'x' + innerHeight
  };
}
```

Read the output against these five questions, and write ONE table row per finding:

| signal in the output | what it usually means | the move |
|---|---|---|
| `hiddenSample` shows a large subtree that is `display:none` at this viewport, and a sibling with a `-mobile` / `-desktop` name exists | the navigation (or hero, or menu) is BUILT TWICE and one copy is hidden by CSS - the browser still parses, styles and keeps both | one structure for both breakpoints, styled by CSS. If the design truly forbids it, at least render the second copy on demand. NOTE: this wins parse/style/memory and HTML bytes, NOT layout time - hidden elements are not laid out (measured, see 70.5). |
| `elementsInHidden` is a large share of the total, concentrated in a cookie modal / preference centre / off-canvas panel | a panel nobody has opened is paying full price on every load | render it on interaction, not on load. Check FIRST whether it is even in the server HTML - consent panels are usually injected by their own script, so no HTML edit can remove them and the fix belongs to the plugin's config or loading strategy. |
| `deepestPath` ends in a long single-purpose chain (a dropdown, a card, a tab) | this is almost always the same element the DOMSize insight named as the start of the deepest chain - the two should agree, and if they do you have your target | flatten the chain: drop the wrappers that exist only for styling |
| `removableWrappers` is high and `divSpanPct` is around or above 40% | divitis - div-in-div wrappers with no semantics | React/Vue fragments instead of wrapper divs; semantic tags (`button`, `nav`, `ul`) instead of stacks of divs |
| `belowFoldPct` is roughly half or more | most of the tree is rendered for content nobody has scrolled to yet | islands / lazy sections / `content-visibility: auto` for below-the-fold blocks |
| `repeatedLists` shows a uniform list or carousel with a large subtree | N identical items rendered up front | pagination or virtualisation; a carousel can start with a slice and extend on interaction |

Cross-check the deepest chain from the script against the element named by the DOMSize
insight in 70.1. If they disagree, say so and trust the insight - it reflects what the
browser actually laid out; the script reflects the DOM at the moment you queried it, which
can be later.

## 70.3b Map every finding to MY source (Do - this is what makes it my project's audit)
A class name in a rendered page is not yet an actionable finding. Turn each row from 70.3
into a file I can open.

- Collect the identifying tokens from the output: the class and id fragments in
  `deepestPath`, `hiddenSample`, `biggestSubtrees`, `repeatedLists`, `wrapperSample`.
- Strip the build hash before searching. CSS Modules, styled-components and Tailwind's JIT
  all append or generate suffixes, so `navigation_dropdown__label__2tt1k` is found in the
  repo as `navigation_dropdown__label`, or as the `dropdown` class inside a
  `*.module.css`. Search for the STABLE part of the token, not the rendered string.
- Search my repo for each token and name the owner. Report one row per finding:

| finding (from 70.3) | token searched | file:line in my repo | owner |
|---|---|---|---|

  Owner is one of:
  - **my component / template** - I can edit it directly. This is where the fix list goes.
  - **my theme / child theme** - editable, but say whether the change survives an update.
  - **a dependency I configure** (page builder, consent plugin, UI library) - the markup is
    generated, so the fix is a SETTING or a different component API, not a source edit.
    Name the setting if you can find it in the docs.
  - **a third-party script** - the markup does not exist in my repo at all (check: it is
    also absent from the server HTML). The only levers are loading strategy, configuration,
    or removing the vendor. Say so plainly instead of proposing an edit I cannot make.
- If a token cannot be found anywhere, say so and stop guessing. An unattributed finding
  goes into the report as "source not identified" - it is still a valid measurement, just
  not yet an actionable fix.
- For WordPress specifically: search the active theme AND the plugins directory AND the
  page builder's saved content (builder markup lives in post meta, not in files) before
  concluding the markup is not mine.

This table is what 70.6 turns into a fix list. Do not write a fix for a finding whose owner
you have not established.

## 70.4 The priority verdict (Do - STOP and show me)
Before proposing a single fix, answer in one short block:
- measured layout/recalc cost: median of the 3 runs, with the spread;
- what else is on this page: LCP, FCP and Start Render from Stage 60;
- the ratio: what share of the page's problem is this?
- verdict, one of:
  - **FIX NOW** - the recalculation is a visible share of the page's cost, or it repeats on
    interaction (see the note below).
  - **FIX LATER** - real but small next to what Stage 60 found. Say what to do first.
  - **NO ACTION** - no DOMSize insight, or a cost inside the noise. Record the structure
    findings as hygiene for the next redesign and stop.
- Note on interaction: this stage measures the LOAD. A tree that is expensive to lay out is
  also expensive every time something changes later - opening a mobile menu, a filter, a
  tab. That cost lands in INP, not here. If the page has such interactions, say that the
  load-time number is a LOWER BOUND on the real cost. Do not measure INP here; that is its
  own stage.

Show me this block and wait. Do not continue to the experiment on a FIX LATER or NO ACTION
verdict unless I ask.

## 70.5 The experiment (Do - only on FIX NOW)
Same discipline as Stage 60, cheaper to run because you do not need to rebuild the page.

- Pick ONE change with the largest expected effect from 70.3, plus - if 70.2 named a
  third-party caller - one variant with that script removed.
- Build the variants exactly as in Stage 60.5: download the page HTML, inject
  `<base href="{PAGE_URL}">`, one file per change, each differing by EXACTLY ONE edit, plus
  an unmodified baseline. Verify each edit actually matched something before measuring.
- Serve locally, then for each variant: `navigate_page` type=`url`, then
  `performance_start_trace` (reload true, autoStop true), then read `DOMSize`.
- 3 rounds, rotated A,B,A,B - never in blocks. Discard the first warm-up load.
- Compare THREE numbers, and know what each one can and cannot move:
  - `Total elements` - responds to any element you delete, visible or not. Proxy for parse,
    style and memory cost.
  - `X of Y nodes needing layout` - responds ONLY to elements that are actually laid out.
  - the layout `Duration` in ms - noisy; see the calibration in 70.1.
- READ THIS BEFORE CONCLUDING ANYTHING. [Measured on an unrelated reference page while
  writing this stage - the mechanism transfers, the numbers do not]: removing a
  `display:none` subtree of 69 elements moved `Total elements` down by 69 and left the
  layout completely untouched - the same "322 of 322 nodes" before and after, 51 ms vs
  52 ms. Hidden elements are parsed, styled and held in memory, but they are NOT laid out.
  So:
  - deleting a hidden duplicate is a real win in parse/style/memory and in the size of the
    HTML - report it as that, and do not expect the layout number to move;
  - to move the LAYOUT number you must reduce what is VISIBLE and laid out: flatten visible
    chains, render fewer visible list or carousel items, or take blocks out of layout with
    `content-visibility: auto`.
  - a variant that changes `Total elements` but not `X of Y` has not failed - it has proven
    which of the two costs it addresses. Say which.
- Verdicts as in Stage 60: SUPPORTED / REGRESSION / INCONCLUSIVE, where the bar is the
  larger of the two spreads - applied per metric, since the three above can disagree.

## 70.6 OUTPUT - "DOM fixes", ROUTED BY STACK (Do)
For each accepted finding: which file to edit, what to change, what to re-measure.
Do NOT edit yet.

Next.js / React:
- Wrapper divs -> `<>...</>` fragments. A wrapper that exists only to hold a class can
  usually move the class onto the child.
- Below-the-fold sections -> `next/dynamic` with `ssr: false`, or an `IntersectionObserver`
  wrapper. Watch the SEO note below before hiding anything indexable.
- Long lists and carousels -> virtualise (`react-window`, TanStack Virtual) or paginate.
- Keep state close to the component that uses it: a state change high in the tree
  re-renders and re-styles everything under it, which is the same cost measured here.
- Duplicated mobile/desktop navigation is the most common single win: one component,
  CSS for the difference.

WordPress:
- Page builders: check the builder's own DOM options first - Elementor v4 (flexbox
  containers, "optimized DOM output") removes whole wrapper levels with a setting, which
  beats any hand editing. Verify by re-running 70.3 with the option on and off.
- Enable HTML minification where the stack offers it (e.g. W3 Total Cache). It does not
  reduce node count, but it reduces the bytes of the document - a separate, real win. Do
  not report it as a DOM-size fix.
- Widgets rendered on every page but used on few: dequeue per page type.
- The cookie/consent plugin is a frequent offender: a full preference centre in the markup
  on every page load. Check whether it can render on interaction.

Astro:
- Islands: `client:visible` for anything below the fold, `client:idle` for the rest. Astro
  ships no JS by default, so a heavy interactive tree here is a deliberate choice - revisit it.
- Sections far below the fold can stay static markup; move only what needs interactivity.

Any stack - in this order:
1. **DELETE** the second copy (duplicated navigation, an unused widget, a legacy wrapper).
2. **DEFER** what is not visible yet (modals, off-canvas panels, below-the-fold sections).
3. **FLATTEN** what stays (fragments, semantic tags, fewer styling wrappers).
4. **STOP FORCING LAYOUT** - fix or defer the caller named in 70.2; batch reads before
   writes so the browser is not asked for geometry mid-invalidation.

SEO note (applies to every "render it later" item above): what the crawler sees must match
what the user sees. Googlebot executes JavaScript, so lazily rendered sections are normally
fine - but do not hide indexable content behind an interaction that a crawler will never
perform, and do not serve a different tree to bots. If a section carries content you want
ranked, keep it in the markup and optimise its shape instead.

## 70.7 Known traps (verified - do not rediscover them the expensive way)
- **Node count is not the finding.** See the boundary at the top. Chrome flags a 40 ms+
  recalculation, not a number of elements. A report that leads with the element count has
  buried the actual result.
- **Hidden elements do not participate in layout.** The single most misleading result in
  this stage: delete a `display:none` subtree and `Total elements` drops while the layout
  event does not move at all. [Reference measurement: -69 elements, layout unchanged at the
  same node count, 51 ms vs 52 ms.] Both numbers are true and they measure different costs.
  Never present a hidden-tree deletion as a layout win.
- **`initScript` does NOT survive the trace's own reload.** `navigate_page` with an
  `initScript` applies to that navigation; `performance_start_trace` with `reload: true`
  performs its own, and the script does not run again. Verified: the removals never
  happened and the insight returned the unchanged page - identical statistics, no warning
  anywhere. If you try to build variants by mutating the page at runtime you will "prove"
  that your fix does nothing. Build variants as HTML files, as in 70.5.
- **A local copy is not production.** [Reference measurement: the same page served locally
  reported roughly a third fewer elements than live], because third-party scripts (consent
  panel, analytics) behave differently or do not inject at all off-domain. Compare variant
  against LOCAL baseline only. Never put a local number next to a production number in the
  same table.
- **Part of the DOM you measured is not in the HTML at all.** [Reference measurement: an
  entire cookie preference centre of 126 elements appeared zero times in the server HTML]
  - it is injected by its own script. You cannot remove such markup with an HTML edit, and
  an experiment that tries will show no difference for the wrong reason. Always `curl` my
  page and check whether the markup you want to change is even server-rendered, before
  designing a variant around it. This check also feeds 70.3b: markup absent from both my
  repo and the server HTML is owned by a vendor script.
- **The viewport changes the answer.** `display:none` subtrees and below-the-fold counts
  are viewport-dependent - the desktop navigation is hidden on mobile and vice versa. Call
  `emulate` once and state the viewport next to every number.
- **Scope the structure script to `document.body`.** Include `<head>` and its ~50 elements
  land in "hidden subtrees" and make the result nonsense.
- **SVG internals inflate the element count** and are not divitis - you cannot flatten an
  icon's paths. Count them separately, as the script above does.
- **The DOMSize insight can legitimately be missing.** That is a result, not a tooling
  failure. Report "no large recalculation on this load" and move on.
- **`getComputedStyle` and `getBoundingClientRect` in a loop are themselves slow** on a big
  tree, and they force layout. That is fine for a one-off audit query, but never leave that
  pattern in shipped code - it is the exact anti-pattern 70.2 is looking for.
- **CSS Selector Stats is a DevTools UI feature, not an MCP one.** The panel can attribute
  recalculation cost to individual selectors (Performance > Settings > Enable CSS Selector
  Stats, then record). It is worth doing by hand when 70.2 blames style recalculation
  rather than layout, and it is where wildcard selectors and `[style]`-matching rules show
  up. Do not claim you ran it through the MCP server - you cannot.
- **Selectors are matched right to left.** A rule ending in `*` or `[style]` is evaluated
  against every element before the left-hand side narrows anything. This is why a single
  careless selector can put a whole tree into a recalculation, and why "how you write CSS"
  belongs in a DOM-cost stage at all.

## Save results
- site-profile.md > ## DOM size: the insight statistics, the layout duration median and
  spread, the attributed caller from 70.2, the structure findings, the priority verdict,
  and the routed fix list.
- findings.md: dated entry with the per-run numbers, the raw script output, and the reasoning.

## STOP line (end every run with this, verbatim intent)
"DOM-size triage done. Measured: <median layout duration, N of M nodes>. Attributed to:
<caller from 70.2, or 'ordinary rendering'>. Structure findings: <top 2>. Priority:
<FIX NOW / FIX LATER / NO ACTION> because <one clause comparing it to LCP/FCP>.
Next: <the single next action>. Remember this measured the LOAD - the same tree is
re-laid-out on every interaction, and that cost shows up in INP, not here."

## At the end
Check off in site-profile.md > ## Progress: [x] Stage 70 - DOM size. Report the priority
verdict and the single next action - nothing more.
