# Stage 160: Rendering and animations - which pipeline phase each effect wakes up

> Can be run STANDALONE or from the index (.ai/web-performance/audit/00-index.md).
> Behaviour: DO stage, ending in AN EFFECT INVENTORY WITH A PIPELINE ENTRY POINT PER EFFECT
> and, where the entry point is earlier than it needs to be, THE CHEAPER EQUIVALENT that
> produces the same visual result from a later phase.
> Read domain/URLs/stack from site-profile.md; if missing, ASK me and WAIT - do not guess.
> Input:  site-profile.md > ## Project, ## Page types, ## Measurement, ## DOM size (Stage 70),
>         ## INP / interactions (Stage 120), ## JS runtime (Stage 130),
>         ## Layout stability (Stage 150)
>         + the chrome-devtools MCP server (or DevTools by hand) + RUM where it exists
> Output: site-profile.md > ## Rendering and animations + findings.md, ending with AN
>         INVENTORY OF EFFECTS (animations, transitions, hover states, scroll effects),
>         THE PHASE EACH ONE TRIGGERS, AN OWNER, and A CHEAPER EQUIVALENT OR A KEPT COST
>         per effect.
> Idempotent: if ## Rendering and animations exists, update it, do not create a second one.

THIS RUNS AGAINST MY PROJECT, NOT AN EXAMPLE (read first):
The origin, the pages and the effects are MINE. Never substitute a demo or a site mentioned
anywhere in this package. If the URL, page types or stack are missing from the profile, ASK me
and WAIT.

Any figure in this file describes a MECHANISM or a threshold - not a measurement of my site.

## SCOPE AND HARD BOUNDARY (read first - IMPORTANT)

**THE FRAME BUDGET RULE.** There is no Core Web Vital for smoothness. The threshold is the
screen: at 60 Hz the browser has ~16.7 ms to produce a frame, and every frame it misses is
visible as jank. TBT and INP correlate with the same congestion but neither measures a dropped
frame during scroll or a stuttering transition. So this stage is sized by frames, not by a
metric score - and a page with green vitals can still fail it.

**THE PHASE RULE.** The cost of a visual change is decided by WHERE it enters the rendering
pipeline: style -> layout -> paint -> composite. A change that enters at layout re-runs
everything after it, on the changed element AND its neighbours; a change that enters at
composite moves an already-painted layer on the GPU while the main thread stays free. The
same visual effect can usually be expressed from more than one entry point, and the whole
job of this stage is to find effects that enter earlier than they need to and move them later.
`transform` and `opacity` are, in practice, the only properties that animate without touching
layout or paint.

**THE MEASURE-FIRST RULE.** This stage does not strip effects on sight. A hover transition on
a button is almost never a problem; the same transition on a full-viewport section on a
3000-pixel-wide screen can be, because paint cost scales with painted area and resolution.
Every verdict here needs a trace or a visual proof (the element lights up in Paint Flashing,
the frame counter drops), not an aesthetic objection. Removing an effect nobody measured is
optimisation on reflex, and the stage reports it as such.

**THE PROMOTION PARADOX.** The browser already promotes layers and optimises `transform` and
`opacity` on its own. Forcing promotion (`will-change`, transform hacks) is the easiest way
in this stage to make performance WORSE: every promoted layer costs memory, especially on
mobile, and `will-change` on everything is a known anti-pattern. Promotion is a targeted,
justified decision per element - never a default.

**THE SCROLL BLIND SPOT.** Scrolling is the one interaction nearly every visit performs, and
INP does not score it. If the field data shows nothing, that is not evidence of smooth scroll -
it is the reason the lab pass in this stage exists.

## 160.0 Read what earlier stages already found (Do)
Pull these before measuring anything - they name the likely suspects:
- ## DOM size (Stage 70): the recalculation hotspots. Style recalc cost scales with node count,
  and an animation over a large subtree multiplies it.
- ## INP / interactions (Stage 120): the slow interactions and their subparts. A long
  presentation delay is frequently THIS stage's problem wearing Stage 120's badge.
- ## JS runtime (Stage 130): scripts that keep the main thread busy after load - an animation
  competes with them for the same frame budget.
- ## Layout stability (Stage 150): shifts already attributed. A transition animated on a
  layout-triggering property may already be on that list as a CLS source.

## 160.1 Size it, field first where the field can see (Do)
- Read INP p75 and its subparts from Stage 120's section rather than re-measuring. A large
  presentation delay on the flagged interactions is this stage's entry ticket.
- If the RUM tool collects Long Animation Frames (most modern ones build on the LoAF API),
  pull the worst frames and their script attribution for the flagged page types. This is the
  one field signal that names the causing function and file - use it before any lab work.
- State plainly what the field CANNOT see here: scroll smoothness, hover effects, transitions
  between pages. For those the lab pass below is the measurement, not a reproduction.
- No RUM at all? Say so, size the stage from the lab pass, and mark the findings
  UNCONFIRMED IN THE FIELD.

## 160.2 The visual pass - four switches, five minutes (Do)
DevTools > More Tools > Rendering, on the page types the profile flags, mobile viewport:
- **Paint Flashing**: scroll the full page slowly, then hover and click what users hover and
  click. Everything that flashes green is being repainted. A region that flashes ON EVERY
  SCROLL FRAME (the classic: `background-attachment: fixed`) is a finding on its own.
- **Layer Borders**: read which elements live on their own compositor layer. Note both
  surprises: an animated element that does NOT have its own layer, and dozens of layers
  nobody asked for (a `will-change` wildcard, a transform hack applied globally).
- **Frame Rendering Stats**: the live FPS / GPU meter. Scroll and interact; note where the
  frame rate drops and what was on screen when it did.
- **Scrolling Performance Issues**: the browser annotates elements that slow scrolling
  (non-passive listeners, repaint-on-scroll regions). Record the selectors it names.
Record an inventory: effect, element, where it appears (page type), what the switches showed.

## 160.3 The trace - where the frames actually go (Do)
- Performance panel through the chrome-devtools MCP server (or by hand): emulate ONCE
  (mobile, CPU 4x - state it next to every number), record load PLUS a few seconds of real
  use: the hovers, the menu, the scroll, one page transition.
- Read the dropped frames on the frame track and correlate each with the event log: how much
  of the frame went to Recalculate Style, Layout, Paint, Composite. The phase distribution IS
  the diagnosis - a frame dominated by Layout points at geometry properties or thrashing, a
  frame dominated by Paint points at painted-area effects.
- Look for the **forced reflow** warning (a dense violet band of Layout events): that is
  JavaScript interleaving reads and writes. Note the function the trace names.
- For attribution beyond the trace, run a Long Animation Frames observer (the WebPerf
  Snippets collection has a ready one: LoAF with script attribution) in the console and
  interact again: it names the script, the function and the character position for every
  long frame, which the Long Tasks view alone does not.

## 160.4 Attribute each effect to its pipeline entry point (Do)
For every effect in the inventory (CSS transitions and animations, JS-driven movement, hover
states, scroll effects, page transitions), answer in this order:
1. **Which property actually animates?** From the CSS or the code, not from the visual
   impression. `width`, `height`, `top`, `left`, `right`, `bottom`, `margin`, `padding` enter
   at LAYOUT - and geometry changes recalculate neighbours, not just the element.
   `background-color`, `box-shadow`, `border-radius`, filters enter at PAINT - cost scales
   with painted area. `transform` and `opacity` enter at COMPOSITE. When unsure, the
   compositor-safe list in the web.dev Animations Guide settles it.
2. **How big is the painted area?** The same property is cheap on a button and expensive on
   a hero section. Painted area x resolution is the multiplier; say it per effect.
3. **Is JavaScript in the loop?** A scroll or input handler that reads geometry
   (`offsetTop`, `offsetWidth`, `getBoundingClientRect`, `getComputedStyle`, `scrollTop`)
   and then writes styles is layout thrashing - each read after a write forces a synchronous
   layout, hundreds of times per second during scroll. Note read/write interleaving
   explicitly; the fix is a different shape from a CSS property swap.
4. **Whose code is it?** My stylesheet or component, the theme, a plugin or page builder
   (WordPress: an animation option ticked in the builder counts), or a third-party widget.
   Unattributed effects are "source not identified", never guessed.
5. **Does the effect run when nobody sees it?** An animation below the fold, or in a closed
   panel, that runs from page load burns frame budget for no one.

## 160.5 The cheaper equivalent, per effect (Do)
Propose one line per effect - same visual result, later pipeline entry, or a kept cost named:
- **Movement**: `transform: translate...` instead of `top`/`left`. **Resize**:
  `transform: scale()` instead of `width`/`height`. **Show and hide**: `opacity` instead of
  geometry or `display` where the layout allows it.
- **The overlay trick** for painted properties: a pseudo-element carrying the target state
  (the hover colour, the shadow) animated with `opacity` 0 -> 1 replaces a repaint with a
  composite. Worth it on large or repeated elements; on a lone small button, note it and
  move on (the measure-first rule).
- **Repaint-on-scroll**: replace `background-attachment: fixed` with a separate
  `position: fixed` layer behind the content. Scroll then moves layers instead of repainting.
- **JS reads and writes**: batch - all reads first, then all writes; writes inside
  `requestAnimationFrame` so they land at the next frame; `cancelAnimationFrame` when the
  animation ends. State what rAF does NOT do: it fixes the SCHEDULING of the work, not its
  COST - and rAF re-armed inside a scroll handler runs in a loop and recreates the problem.
- **Position against the viewport**: IntersectionObserver instead of reading `scrollY` or
  `offsetTop` in a scroll handler.
- **Scroll-driven effects**: CSS scroll-driven animations where support allows - no listener,
  no JS on the scroll path at all. Progressive enhancement; state the fallback.
- **Page transitions**: the View Transitions API instead of framework transition libraries -
  native, no layout and no paint in the frame even though the whole page changes. Progressive
  enhancement: where unsupported, the page simply navigates; nothing is lost.
- **Promotion, only where measured**: `will-change` declares INTENT - it goes on the element
  BEFORE the change (on the parent, at rest state - not inside the `:hover` that is already
  animating, where it arrives too late to help). In JS, add it before the change and REMOVE
  it after, or the browser holds the layer's memory forever. Never `will-change` on a
  wildcard or "on everything animated" - promotion costs memory and the browser already
  handles `transform`/`opacity` on its own. An element already isolated by `position: fixed`
  may not need it at all - measure.
- **Effects that run unseen**: pause them. A playing-state class toggled by
  IntersectionObserver, animation only while in the viewport; `content-visibility` for
  whole offscreen sections (noting, per Stage 130, that it skips layout and paint - the
  section's JS still runs).
- **`prefers-reduced-motion`**: respect it. Users with animations disabled at the OS level
  (accessibility, battery) get the reduced path; on an animation-heavy page this is both a
  correctness and a performance line, and it costs a media query.
- **CSS vs JS is not a war**: CSS for small UI state (buttons, menus, sidebars), JS where
  the animation needs control - sequencing, pause, reaction. The wrong question is "which is
  faster"; the right one is whether the property it animates enters at composite.

State for each: the change, the file it belongs in, and the stage that owns it if not this one
(a thrashing scroll handler found here may already be Stage 130's chunk; a transition that
shifts layout belongs on Stage 150's list too).

## 160.6 The verdict (Do - STOP and show me)
- **sizing**: what the field could see (INP presentation delay, LoAF attribution) and what
  only the lab pass covered, sources named;
- **effect inventory**: effect, element, page type, pipeline entry point, painted area,
  owner;
- **the offenders**: effects entering at layout or paint with a measured or visually proven
  cost (dropped frames, full-frame flashing), worst first;
- **thrashing sites**: where JS interleaves reads and writes, with the function the trace
  named;
- **layer picture**: elements that should be promoted and are not, and promotions nobody
  justified;
- **cheaper equivalents proposed**, and effects whose cost is KEPT deliberately (measured,
  small, or worth it visually) - a kept cost with a number is a legitimate outcome;
- **routed back**: what belongs to Stage 70, 120, 130 or 150;
- **how this compares** to what those stages already have open. A page whose effects all
  enter at composite and whose frame counter holds can legitimately end in NO ACTION.

Show me this block and wait.

## 160.7 The experiment (Do - only on a FIX NOW)
- Prove the swap the same way Stage 150 proves a reservation: Local Overrides in DevTools,
  paste the changed CSS onto the production page, no deploy.
- Re-record the SAME scenario on the SAME emulation (state it), minimum 5 runs, and compare
  the same evidence that made the finding: the dropped-frame count on the frame track, the
  phase distribution in the event log (the Layout/Paint share should fall or vanish), Paint
  Flashing on the affected region. For an interaction, re-read the INP subparts per
  Stage 120. Medians, spread, and INCONCLUSIVE when the difference is smaller than the
  spread - the evidence rule from 00-index.md applies unchanged.
- Re-run the visual pass (160.2) after the fix, not only the trace: a swap that cleans the
  trace but leaves a full-viewport region flashing on scroll is a partial result and is
  reported as one.
- Where the finding came from LoAF in RUM, set a date to re-read the field; until then the
  result is SUPPORTED IN LAB.

## 160.8 Known traps
- **Optimising smoothness by a metric score.** There is no vital for it. Green vitals and a
  janky scroll coexist happily; the frame budget is the threshold.
- **Judging an animation by how it looks in the code.** A transition on `left` looks identical
  to a transition on `transform` in the browser - until the frame track is open. Attribute by
  property, prove by trace.
- **`will-change` inside the `:hover` that already animates.** It declares a FUTURE change;
  applied at the moment of the change it is too late to help and still costs memory.
- **`will-change` on everything.** Layer memory, especially on mobile. Promotion is a targeted
  decision, and the browser already optimises `transform`/`opacity` unprompted.
- **rAF sold as a cost reduction.** It schedules the work at the right time; the work costs
  the same. And rAF re-armed per scroll event runs in a loop.
- **A cheap property on an expensive area.** Paint cost scales with painted pixels; the same
  transition is free on a button and heavy on a full-screen section at 3000-pixel widths.
- **Deleting effects nobody measured.** The stage's job is to move effects to a later phase or
  prove a cost, not to flatten the design. An unmeasured removal is a design change, not an
  optimisation.
- **The lab-window blindness inherited from Stage 150.** A curtain transition that runs only
  BETWEEN pages never appears in a page-load test; RUM (or the use-phase pass) sees it.
- **Treating scroll as covered by INP.** It is not scored. The most frequent interaction on
  the site is the one the metric ignores.
- **Forgetting the non-human consumer.** An AI agent or e2e test driving the page drops its
  target when the page moves mid-click - smoothness findings change priority when automated
  clients matter (the profile says whether they do).

## Save results
- site-profile.md > ## Rendering and animations: the sizing with its sources, the effect
  inventory with entry points and owners, the thrashing sites, the layer picture, the cheaper
  equivalents decided (and the costs kept deliberately), what was routed back to which stage,
  and the experiment result with medians and spread.
- findings.md: dated entry with the numbers and the reasoning.

## STOP line (end every run with this, verbatim intent)
"Rendering triage done. Sizing: <what field saw / lab-only>, worst frames <where, phase
distribution>. Effects: <n> inventoried, <n> entering at layout, <n> at paint, <n> at
composite. Thrashing: <sites, function names>. Layers: <promotions missing / unjustified>.
Swaps proposed: <list>. Kept costs: <list with numbers>. Routed: <what, to which stage>.
Verdict: <FIX NOW / FIX LATER / NO ACTION>. Next: <the single next action>."

## At the end
Check off in site-profile.md > ## Progress: [x] Stage 160 - rendering and animations
(smoothness). Report the sizing, the inventory with entry points, the swaps and the kept
costs - nothing more.
