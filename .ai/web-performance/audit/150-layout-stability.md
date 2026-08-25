# Stage 150: Layout stability - what the tool measured, and what the user actually lived through

> Can be run STANDALONE or from the index (.ai/web-performance/audit/00-index.md).
> Behaviour: DO stage, ending in TWO POPULATIONS OF SHIFTS AND A RESERVATION PER SHIFT. The
> load-phase shifts every lab tool reports, and the ones that happen while the page is being
> USED - which the lab never sees, because its test had already finished.
> Read domain/URLs/stack from site-profile.md; if missing, ASK me and WAIT - do not guess.
> Input:  site-profile.md > ## Project, ## Page types, ## Measurement, ## Render start (Stage 60),
>         ## Media (Stage 90), ## Fonts (Stage 100), ## Consent / CMP (Stage 110),
>         ## JS runtime (Stage 130)
>         + CrUX or RUM + WebPageTest + the chrome-devtools MCP server
> Output: site-profile.md > ## Layout stability + findings.md, ending with A FIELD SIZING, AN
>         INVENTORY SPLIT BY PHASE, AN OWNER AND A RESERVATION PER SHIFT, and where they differ,
>         A METRIC VERDICT AND A UX VERDICT that are allowed to disagree.
> Idempotent: if ## Layout stability exists, update it, do not create a second one.

THIS RUNS AGAINST MY PROJECT, NOT AN EXAMPLE (read first):
The origin, the pages and the shifts are MINE. Never substitute a demo or a site mentioned
anywhere in this package. If the URL, page types or stack are missing from the profile, ASK me
and WAIT.

Any figure in this file describes a MECHANISM or a threshold - not a measurement of my site.

## SCOPE AND HARD BOUNDARY (read first - IMPORTANT)

**THE WINDOW RULE.** Every lab tool measures a WINDOW: it loads the page, waits for its own
finish condition, and stops. CLS in the field is collected for the whole visit. So a lab score
of zero says one thing only - nothing shifted before the tool stopped watching. A bar that drops
in three seconds after load, a section that reflows on scroll, a transition that reflows on the
way to the next page: all real, all counted for real users, none of them in the lab number. Size
this stage from the field first. Where lab and field disagree, THE FIELD IS THE MEASUREMENT and
the lab is the reproduction.

**THE TWO-DIRECTION VISIBILITY RULE.** The metric and the experience are not the same set, and
this stage must report both.
- Counted but harmless: the page moves because I deliberately moved it. A confirmation message
  injected after a form submit, with the user scrolled to it on purpose, is good UX that scores
  as a shift.
- Uncounted but harmful: anything after the tool's window, and anything within 500 ms of a real
  interaction, which the metric excludes by design (`hadRecentInput`). The exclusion is a
  measurement rule, not a permission - a click that reflows the page under the user's finger is
  still a bad experience and belongs in the finding.
A stage that only chases the number will optimise the wrong shifts and defend the harmful ones.

**THE SIDE-EFFECT RULE.** On an audited project most shifts are the PRICE OF AN EARLIER FIX.
Async CSS (Stage 60), lazy loading and missing dimensions (Stage 90), `swap` on a font whose
metrics do not match (Stage 100), the consent banner (Stage 110), rendering deferred to
JavaScript (Stage 130) - each of them buys a faster first paint by letting something arrive
late, and late arrivals move the page. This stage re-audits the techniques the audit itself
recommended. Treat every finding here as a question to an earlier stage, not as a new topic.

**THE RESERVATION RULE.** The fix is almost never "remove the element" and almost always "hold
its space before it arrives". Space held in CSS is there when the first byte of CSS is parsed.
Space computed in JavaScript arrives when the JavaScript arrives - which is exactly the delay
that caused the shift. **A reservation implemented in JS is not a reservation.** If a height
depends on a calculation, the stage's job is to find the CSS expression that expresses it
(`aspect-ratio`, `min-height`, viewport units, a container query) and to name what breaks if
it is approximate.

**THE NON-HUMAN CONSUMER.** An automated client - an AI agent driving a real browser, a scraper,
an end-to-end test - loses its target the same way a person does: it locates an element, the
layout moves, and the click lands somewhere else. When the project is used through such clients,
say so, because it changes the priority of a shift that a human would merely find annoying.

## 150.0 Read what earlier stages already found (Do)
Pull these before measuring anything - they name the likely owners:
- ## Render start (Stage 60): is CSS loaded asynchronously, and is there a critical/inline CSS
  split? Both are prime suspects for above-the-fold shifts.
- ## Media (Stage 90): images without intrinsic dimensions, and which images are lazy.
- ## Fonts (Stage 100): the `font-display` choice and whether the fallback was metric-matched.
  `swap` without matching is a shift by design.
- ## Consent / CMP (Stage 110): the banner's insertion point and its timing.
- ## JS runtime (Stage 130): components whose rendering, sizing or initial state is decided in
  JavaScript.

## 150.1 Size it from the field (Do)
- Read CLS at p75 for my origin from CrUX (or my RUM tool), **split by device**. Mobile is
  usually where this metric fails, and a desktop-only reading hides it.
- Read it PER PAGE TYPE where the data allows. A single origin-level number hides a template
  that is broken for a fraction of the traffic.
- Compare the field number against the lab number for the SAME page. A lab score near zero next
  to a field score above 0.1 is not a contradiction - it is the finding, and it says the shifts
  live outside the lab window.
- Thresholds: good below 0.1, poor above 0.25. Closer to zero is better, but zero is not the
  goal on a page with ads or dynamic sections - the goal is that nothing moves under the user.
- **No CrUX data** (too little traffic)? Say so, fall back to RUM, and mark the sizing
  UNCONFIRMED IN THE FIELD.

## 150.2 Load-phase inventory (Do)
- WebPageTest, mobile emulation, on the page types the field flagged. Open the CLS metric's
  **event details**: it gives a screenshot per shift with the moved regions boxed, and hovering
  highlights the element. Record every shift: the timestamp, the region, and whether it sits
  above or below the fold.
- Then the same page through the chrome-devtools MCP server: emulate ONCE (mobile, Fast 4G,
  CPU 4x), trace the load, and read the layout shift clusters. State the emulation next to every
  number.
- For each shift record the ELEMENT, not the area. "Something at the top moves" is not a
  finding; a selector is.
- Compare the two screenshots on either side of a shift before reasoning about causes. An
  element that appears UNSTYLED in the first and styled in the second is a CSS arrival problem;
  an element ABSENT in the first is an insertion problem. They have different owners and
  different fixes.

## 150.3 The pass nobody runs: after the page looks finished (Do - do not skip this)
This is the half of the stage that the lab cannot do for you.
- Register a `PerformanceObserver` for `layout-shift` and keep it running. Log for each entry:
  `value`, `hadRecentInput`, the `sources` array with the moved nodes, and the time.
- Then USE the page for at least 15 seconds after it looks ready, and record what moves:
  - wait without touching anything - late ads, embeds, a consent or notification bar, anything
    on a timer;
  - scroll to the bottom slowly - lazy images, infinite scroll, sticky elements, sections that
    reserve nothing;
  - trigger the interactions that matter (accordion, tabs, filters, "load more") and note which
    shifts carry `hadRecentInput: true`;
  - navigate to the next page type and back, and watch the transition itself - a page transition
    animated on properties that trigger layout produces shifts that no page-load test will ever
    show.
- Separate the two populations in the output: shifts DURING LOAD and shifts DURING USE. The
  second population is why this stage exists.

## 150.4 Attribute each shift to an owner (Do)
For every shift in both populations, answer in this order:
1. **Is the sizing in the document the server sent?** Open the raw HTML (view-source), not the
   DOM inspector, and look for the element. An attribute or an inline style present in the DOM
   but ABSENT in the source was added by JavaScript, and its arrival time is the shift. This one
   check settles most findings.
2. **Which technique put it there?** Map back to the stage that owns it (150.0). If the shift is
   the price of an async CSS split, the fix may belong in Stage 60, not here.
3. **Whose code is it?** My template or component, my theme, a plugin or page builder I
   configure, or a third-party tag whose markup is not in my repo. On WordPress search the
   plugins and the builder's saved post meta too. Unattributed shifts are reported as "source
   not identified", never guessed.
4. **Does it recur across page types?** A shift present on every template is one fix; a shift on
   one template is a different priority.

## 150.5 The reservation decision, per shift (Do)
Propose one line per shift, in CSS wherever the source allows it:
- **Images and video**: intrinsic `width` and `height` attributes, or `aspect-ratio` in CSS. This
  is the cheapest fix in the stage and it is also the one Stage 90 should already have closed -
  if it is still open, say so.
- **Ad slots**: a `min-height` per breakpoint on the slot's own wrapper, sized to what the
  provider actually serves. Where the creative size varies, reserve the largest common size and
  state the whitespace as the accepted cost. If the position is contractual ("always between the
  first and second heading"), that predictability IS the reservation - use it.
- **Embeds, iframes, widgets**: the wrapper gets the box, not the widget.
- **Components whose initial state is decided in JS** (an accordion opened by a script, a
  panel expanded after hydration, a height computed with `calc` and written by JS): express the
  initial state in CSS so the first render is already correct, and let the script take over
  afterwards. This is the highest-value fix in the stage and the one most often missed.
- **Fonts**: metric-match the fallback (`size-adjust`, `ascent-override`, `descent-override`) or
  accept the swap. Route the trade-off back to Stage 100 rather than re-deciding it here.
- **Anything injected above the user's current position**: do not, unless the same change also
  scrolls the user there deliberately. That is the whole distinction between a shift that helps
  and one that disorients.
- **Animations and transitions**: `transform` and `opacity` only. A transition animated on
  layout-triggering properties both costs a forced layout and inflates the metric - and because
  it happens after the load, it is invisible in the lab.

State for each: the fix, the file it belongs in, and the stage that owns it if it is not this one.

## 150.6 The verdict (Do - STOP and show me)
- **field sizing**: CLS p75 by device and page type, source named, against the lab number for the
  same page;
- **load-phase shifts**: element, score, above or below the fold, owner;
- **use-phase shifts**: what moved and when, including the ones excluded by `hadRecentInput` and
  why they still matter;
- **metric verdict vs UX verdict**: name any shift that scores badly but serves the user, and any
  that scores clean but disorients. They are allowed to disagree; both go in the report;
- **reservations proposed**, ordered by what the field says users actually meet;
- **routed back**: which shifts belong to Stage 60, 90, 100, 110 or 130 rather than here;
- **how this compares** to what is already open from earlier stages. A page whose field CLS is
  below 0.1 with no shift under the user's hands can legitimately end in NO ACTION.

Show me this block and wait.

## 150.7 The experiment (Do - only on a FIX NOW)
- Test the reservation BEFORE any deploy. In DevTools > Sources, enable **Local Overrides**, then
  override the document (or the stylesheet) from the Network panel and paste the reserving CSS
  inline. This proves the fix on the real production page with no access to the repository, which
  is frequently the situation on a client project.
- Re-measure on the SAME emulation, minimum 5 runs, compare medians, and state the spread. The
  evidence rule from 00-index.md applies: a difference smaller than the spread is INCONCLUSIVE.
- Re-run the use-phase pass (150.3) after the fix, not only the load trace. A reservation that
  fixes the load and leaves the scroll shift in place is a partial result and must be reported as
  one.
- **Then re-read the field.** Layout stability is a session metric, so the lab result is
  SUPPORTED IN LAB until the field data catches up. Set a date.
- A partial improvement is a legitimate outcome: a multi-layer cause (a JS dependency behind an
  async stylesheet) is rarely closed by one pasted rule, and the honest report says how much of
  the shift is left and where it lives.

## 150.8 Known traps
- **Reading a lab zero as stability.** The tool measured a window. The user lives in a session.
- **Auditing only the load.** Everything that arrives late, on scroll, on a timer, or during a
  transition is invisible to a page-load test and fully visible to the user.
- **Reserving space in JavaScript.** The reservation then arrives exactly as late as the element
  it was meant to hold. A height computed with `calc` and written by a script is this trap.
- **Trusting the DOM inspector for "is the size in the HTML".** Read the source the server sent.
  Half this stage is settled by that one comparison.
- **Critical CSS that missed an element.** An inline critical set that skips one above-the-fold
  component gives that component its size only when the async stylesheet lands - and the split
  was supposed to make the page faster.
- **Treating every shift as a defect.** A deliberate scroll to a confirmation message is good UX
  that happens to score.
- **Treating the 500 ms interaction exclusion as permission.** The metric forgives it; the user
  under whose finger the page moved does not.
- **Blaming the metric for an animation.** A transition on layout-triggering properties really
  does move the layout. Fix the property, not the score.
- **Measuring the homepage on desktop.** This metric fails on mobile templates, and per page type.
- **Assuming an ad slot has a fixed size.** Providers serve different creatives; reserve for what
  is actually served and accept the whitespace on purpose.
- **Chasing zero on a page that legitimately grows** (infinite scroll, live content). The target
  is "nothing moves under the user", not a number.

## Save results
- site-profile.md > ## Layout stability: the field sizing with its source, both shift
  inventories with elements/scores/owners, the metric and UX verdicts, the reservation decided
  per shift, what was routed back to which stage, the experiment result with medians and spread,
  and the date set for field confirmation.
- findings.md: dated entry with the numbers and the reasoning.

## STOP line (end every run with this, verbatim intent)
"Layout stability triage done. Field: CLS p75 <x> mobile / <y> desktop (<source>), lab on the
same page <z>. Load-phase shifts: <n>, worst <element> <score>. Use-phase shifts: <n>, worst
<what moved, when>. Owner split: <mine / plugin / vendor>. Metric vs UX: <where they disagree>.
Reservations: <list, with the stage that owns each>. Verdict: <FIX NOW / FIX LATER / NO ACTION>.
Field confirmation due <date>. Next: <the single next action>."

## At the end
Check off in site-profile.md > ## Progress: [x] Stage 150 - layout stability (CLS). Report the
field sizing, both inventories, the two verdicts and the reservations - nothing more.
