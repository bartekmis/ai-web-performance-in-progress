# Stage 110: Consent / CMP - one component, three unrelated failures, and a gate you may not cross

> Can be run STANDALONE or from the index (.ai/web-performance/audit/00-index.md).
> Behaviour: DO stage, ending in an EXPERIMENT. Unlike Stage 100 the difficulty here is not
> arithmetic - it is that ONE component fails in THREE different ways, through three
> unrelated mechanisms, each with its own fix, and two of those fixes make the third worse.
> A run that returns a single verdict for "the cookie banner" has got this stage wrong.
> Read domain/URLs/stack from site-profile.md; if missing, ASK me and WAIT - do not guess.
> Input:  site-profile.md > ## Project, ## Page types, ## Render start (Stage 60),
>         ## Preconnect (Stage 30), ## Scripts (Stage 80), ## Media (Stage 90)
>         + the chrome-devtools MCP server + MY SOURCE CODE
> Output: site-profile.md > ## Consent / CMP + findings.md, ending with THREE separate
>         verdicts (render start, LCP, INP), a relocation decision, and a measured experiment.
>         Runs AFTER Stage 80 (it names the script) and AFTER Stage 90 (it says what the LCP
>         element is when the banner is absent).
> Idempotent: if ## Consent / CMP exists, update it, do not create a second one.

THIS RUNS AGAINST MY PROJECT, NOT AN EXAMPLE (read first):
The page you measure is MY page - the URL in site-profile.md > ## Project, for the page type
I am auditing. Never substitute a demo, a vendor documentation example, or a site mentioned
anywhere in this package. If the URL, the page types or the stack are missing from the
profile, ASK me and WAIT.

Any figure in this file describes a MECHANISM or a vendor default - not a measurement of my
site. Never copy a figure from here into findings.md and never treat one as a target.

The deliverable is not "add defer to the cookie script". It is: THIS consent script, loaded
from THIS origin through THIS chain, blocking (or not) the parser, classified (or not) as my
LCP element, costing THIS much on the consent click - with one policy per failure and proof
that consent still works.

## SCOPE AND HARD BOUNDARY (read first - IMPORTANT)

**THE THREE-FAILURE RULE. This is the rule that makes or breaks the stage.**
A consent banner is one component that damages three metrics through three mechanisms that
have nothing to do with each other:

| failure | mechanism | class of problem | fix lives in |
|---|---|---|---|
| Start Render / FCP | synchronous script high in `<head>` blocks the parser | LOADING | 110.2, 110.5 |
| LCP | the banner (or a paragraph inside it) is classified as the LCP element | CLASSIFICATION | 110.3, 110.6 |
| INP | the consent click releases a cascade of tags that runs before the next paint | SCHEDULING | 110.4 |

They do not share a fix, and two of them pull against each other: relocating the script so it
stops blocking the parser makes the banner appear LATER, which is worse for the user and can
move which element wins LCP. A stage that reports "the CMP is slow, defer it" has collapsed
three findings into one and will fix at most one of them.

**THE CONSENT GATE (this outranks every performance finding in this stage).**
The vendor's own install instructions tell you to put the script immediately after `<head>`
with no `async` and no `defer`, and that is not incompetence - it is deliberate. That script
is the DRIVER for every other script on the page: it must establish the default consent state
(everything denied) before anything that could write a cookie gets a chance to run. Change how
it loads and you can silently start writing cookies before consent.

Therefore: **no loading change proposed in this stage may be reported as done until you have
verified, on the modified page, that no cookie outside the strictly-necessary category is
written before consent is given.** A faster page that sets an analytics cookie on a visitor
who never agreed is not an optimisation, it is a legal defect that I am now shipping. Verify
it (110.5), state that you verified it, and if you cannot verify it, say the change is
UNPROVEN rather than done.

**THE HONESTY TEST (110.6 exists because of this).**
The off-screen technique in 110.6 improves the LCP value by changing WHEN the banner becomes
visible, not by making anything faster. That is legitimate only while the banner still reaches
the user quickly. The stage must therefore measure TIME TO BANNER VISIBLE as a first-class
number next to LCP. If LCP improves and time-to-banner-visible degrades materially, the change
is metric-gaming and must be reported as a REGRESSION, whatever Lighthouse says.

One further boundary: **this stage does not own CLS, but the banner is a routine contributor.**
It is injected late into the top of the page. Measure the shift it causes here, record it, and
route the total CLS budget to the CLS stage.

## 110.0 Read what earlier stages already found (Do, no new measurement yet)
- From site-profile.md > ## Scripts (Stage 80): is the consent script already inventoried,
  with its origin, transfer size and loading attribute? If yes, do not re-measure it - carry
  the row forward. If Stage 80 recorded it as render-blocking, that is finding 1 already half
  done.
- From site-profile.md > ## Media (Stage 90): **what is the LCP element when the banner is not
  in the way?** This is the number 110.3 compares against. If Stage 90 named a hero image,
  that image is what the banner is stealing LCP from, and 110.6 has a target to restore.
- From site-profile.md > ## Render start (Stage 60): Start Render / FCP for the slow page type,
  and which render-blocking resources are already deferred.
- From site-profile.md > ## Preconnect (Stage 30): is the CMP vendor origin already warmed? If
  the script is relocated into a tag manager (110.5) it starts LATER, and a preconnect stops
  being a nicety and becomes part of the fix.
- GATE: if Stage 50 says TTFB is the wall, note it and carry it into 110.7. Run the stage
  anyway - the inventory is cheap - but the verdict must say whether consent work is next or
  noise next to what is already on the table.

## 110.1 Inventory - what is actually there (Do)
Everything here runs through the chrome-devtools MCP server.

- Call `emulate` ONCE: mobile viewport ~412x765x2.6 (mobile, touch), "Fast 4G", CPU throttling
  4x. Do not change it for the rest of the stage. State the viewport next to every number.
- **Measure as a FIRST-TIME visitor.** The banner only appears when no consent cookie exists.
  Clear cookies and storage before every single run in this stage, and say that you did. A
  measurement taken with consent already stored is measuring a different page and is the most
  common way this whole stage silently produces nothing.
- `navigate_page` to the page type you are auditing, then `performance_start_trace` with
  `reload: true` and `autoStop: true`.
- Also `curl` the page and keep the raw server HTML. You need it to tell a script that ships in
  the document from one injected later.

Record:

| item | value |
|---|---|
| CMP present? | vendor name, or none (then stop and say so) |
| vendor | CookieYes / Cookiebot / Google (AdSense/Funding Choices) / own build / other |
| script origin | first-party or third-party host |
| transfer size | KB |
| loading attribute | none (synchronous) / `async` / `defer` |
| position in the document | e.g. line n of `<head>`, before or after the CSS |
| how it arrives | directly in the HTML / as a tag inside a tag manager / injected by a script |
| tag manager present? | GTM container id, or none |
| consent framework | Consent Mode v2 / TCF / none |

If there is a tag manager AND the consent script is also hardcoded in the document, record it
as **two independently loaded entry points** and note that their ordering is now something the
project has to guarantee by hand. That is a finding in itself.

## 110.2 Failure 1 - does it block the parser? (Do)
- From the trace, read whether the consent script is marked render-blocking, and record its
  request start, download and execution times.
- Reconstruct the chain the same way Stage 100 does: how many serialised hops before the
  request starts. A third-party CMP referenced directly from `<head>` costs DNS + TCP + TLS on
  a first visit before the first byte arrives.
- Record the delta between Start Render and the moment the consent script finishes executing.
- **The counterfactual, and it is cheap.** Do not argue about the cost - block the request and
  re-measure. Use request blocking (as Stage 80 does) to drop the CMP origin, run the page
  again, and record Start Render / FCP with and without. That number is the entire size of
  failure 1, measured rather than estimated. It is not a proposal to remove the CMP; it is the
  ceiling on what any loading fix can win.

## 110.3 Failure 2 - is the banner the LCP element? (Do)
- Read the LCP element from the trace **on the first-time-visitor run** (banner present).
  Record the element, its size and its time.
- Compare against Stage 90's LCP element (measured with consent stored, so no banner). Three
  possible outcomes, and they lead to different places:
  - **the banner or a node inside it IS the LCP element** -> failure 2 is real, 110.6 applies;
  - **the LCP element is unchanged, the banner merely overlaps it** -> failure 2 is NOT real.
    Say so plainly and do not propose the off-screen trick. It buys nothing here;
  - **the LCP element changed to something else entirely** (the banner pushed content) -> this
    is a layout problem, record the shift and route it to the CLS stage.
- Record the LCP element's size on screen. A full-width overlay is frequently the largest
  paint on a mobile viewport by a wide margin, which is why it wins - state the area so the
  finding is quantified rather than asserted.

## 110.4 Failure 3 - what does the consent click cost? (Do)
- With the trace running, click the primary consent button and record: the input delay, the
  processing time, the presentation delay, and the total interaction duration.
- Record what the click STARTS: how many requests fire, their total transfer, and the total
  main-thread work in the same frame. This is the tag cascade.
- Record the number that actually matters to the user: **time from click to the banner being
  gone from the screen.** An interaction that loads eleven tags but repaints in 40 ms is not
  the failure; one that keeps the banner on screen for 600 ms is.
- **Do not assume this failure exists.** The large vendors have shipped scheduling fixes for
  exactly this - deferring the cascade behind `setTimeout` or the Scheduler API so the visual
  change lands first. Measure it on MY page. If the interaction is already fast, record NO
  ACTION for failure 3 and move on; that is a legitimate and common result now.
- If Stage 80 recorded a tag manager, check whether the cascade is tags firing on a consent
  event rather than the CMP itself. The fix then belongs in the tag configuration, not in the
  CMP - name which one it is.

## 110.5 The relocation decision - and the consent gate (Do)
This is the main loading fix and the place the gate applies.

Three options, in increasing order of both benefit and risk:

1. **Leave it synchronous.** Correct by construction, worst for Start Render. The baseline.
2. **Add `defer`.** The minimum. The script stops blocking the parser but still runs before
   the page becomes interactive. Cheap, low risk, partial win.
3. **Relocate it into the tag manager.** The consent script is removed from the document
   entirely and becomes a tag fired on the container's initialisation/consent event. The
   document then contains no reference to the vendor origin at all. The tag manager loads
   asynchronously anyway, so nothing in the document blocks. Vendors ship official GTM
   templates for this - prefer the template to a custom HTML tag.

Record which option applies, and then, whichever change is proposed:

- **Run the consent gate. This is not optional.** On the modified page, as a first-time
  visitor: enumerate cookies and storage writes BEFORE any consent is given, and list every
  one that is not strictly necessary. Compare against the same list on the unmodified page.
  Any new entry means the change broke consent and must be withdrawn regardless of what it did
  to the metrics. State the two lists in the report.
- Check the consent signal still reaches the tag manager: with consent denied, confirm that
  analytics tags do not fire, and that Consent Mode v2 pings (if the vendor supports them) are
  still sent in their anonymised form. Losing those pings silently is a real cost of a careless
  relocation - it is analytics data the project was legally entitled to keep.
- **Relocation makes the banner appear later. Price that in.** The script now waits for the tag
  manager. Record the new time-to-banner-visible, and mitigate with a `preconnect` to the
  vendor origin in `<head>` (Stage 30 owns the tag; this stage supplies the reason). Where the
  vendor URL is stable, a `preload` of the script itself is the stronger version - the browser
  starts fetching immediately while execution still waits for the tag.
- If the document ends up with BOTH the tag manager and a hardcoded consent script, say so:
  the ordering between them is now a hand-maintained invariant and a future deploy can break
  it silently.

## 110.6 The off-screen policy - only if 110.3 said the banner IS the LCP element (Do)
The technique: render the banner outside the viewport from the first frame
(`transform: translateY(110vh)` or equivalent), then move it in with a transition once the
page has painted, typically after a short delay. The script runs normally from the first
moment; only the UI position is deferred. The real element wins the LCP race; the banner still
arrives.

Rules for using it at all:

- **Only when 110.3 confirmed the banner is the LCP element.** Otherwise it changes nothing
  and adds a rule to maintain.
- **The class names are per vendor.** Do not copy a selector from anywhere - read the rendered
  DOM on MY page and record the actual container selector. State it in the report.
- **`transform` and `opacity` only.** Anything that animates layout reintroduces the shift this
  stage is trying to avoid.
- **Never `display: none` or `visibility: hidden` on the container as the mechanism.** It can
  prevent the banner from being interactive or announced, and a consent dialog that a keyboard
  or screen-reader user cannot reach is a worse defect than the metric it fixed. Verify focus
  order and that the dialog is reachable after it moves in.
- **Run the honesty test.** Record, before and after: LCP value, LCP element, and
  time-to-banner-visible. The change is a WIN only if LCP improves and time-to-banner-visible
  is materially unchanged. If the banner arrives noticeably later, report REGRESSION - you have
  moved a number without helping anyone.

Say plainly in the report what this does and does not do: it changes which element is
classified as LCP. It does not make the script, the page or the banner faster.

## 110.7 The three verdicts (Do - STOP and show me)
Before proposing a single fix, answer in one short block. Three verdicts, not one:

- **failure 1 - render start**: blocking yes/no, chain and hops, measured cost from the
  blocked-request counterfactual (110.2), verdict FIX NOW / FIX LATER / NO ACTION;
- **failure 2 - LCP**: is the banner the LCP element (yes/no), what the LCP element is without
  it, the banner's area on screen, verdict FIX NOW / FIX LATER / NO ACTION;
- **failure 3 - INP**: interaction duration on the consent click, what the cascade is, time
  from click to banner gone, verdict FIX NOW / FIX LATER / NO ACTION;
- **the consent gate**: cookies written before consent today - and, for any change proposed,
  whether the gate was re-run and passed;
- **how this compares** to what Stages 60, 80 and 90 found - is the CMP a visible share of this
  page's problem, or noise next to a 2-second TTFB?

Show me this block and wait. Do not continue to the experiment unless I ask.

## 110.8 The experiment (Do - only on a FIX NOW)
- Build variants as local HTML exactly as in Stage 60.5: download the page, inject
  `<base href="{PAGE_URL}">`, one edit per variant, plus an unmodified baseline. Serve locally
  and measure with the same emulation. Do NOT mutate the page at runtime with `initScript` - it
  does not survive the reload `performance_start_trace` performs (verified in Stage 70).
- **Clear cookies and storage between every run.** Without this the banner stops appearing
  after the first run and every subsequent number is measuring a different page.
- Run at least:
  1. baseline, untouched;
  2. consent script with `defer`;
  3. consent script relocated to the tag manager (plus `preconnect` to the vendor origin);
  4. baseline plus the off-screen CSS - **only if 110.3 said the banner is the LCP element.**
- 3 rounds, rotated A,B,C,D,A,B,C,D - never in blocks. Discard the first warm-up load. Compare
  medians against the larger of the spreads.
- Record for EVERY variant: Start Render / FCP, LCP **and which element won it**, CLS, and
  **time-to-banner-visible**. The last one is not optional - it is what stops variant 3 and
  variant 4 from looking like free wins.
- Compare local against LOCAL baseline only.
- Verdicts: SUPPORTED / REGRESSION / INCONCLUSIVE.

## 110.9 OUTPUT - one policy per failure, ROUTED BY STACK (Do)
Do NOT edit yet. One policy per failure, plus the gate result.

WordPress:
- The consent plugin usually injects the script through `wp_head` at a fixed priority. Check
  the plugin's own settings first - several now offer a "load through Google Tag Manager" or
  "delay until interaction" switch, and using it beats writing code.
- If the theme or a cache plugin has a "delay JavaScript execution" feature, check whether the
  consent script is on its exclusion list. It usually must be: delaying it until first
  interaction is precisely the thing that breaks the consent default state.
- The off-screen CSS belongs in the theme stylesheet, not in an inline `<style>` injected by
  another plugin that may load after the banner paints.

Next.js / React:
- Use the framework's script component with the strategy chosen here rather than a raw tag, and
  keep it out of any client component that remounts.
- If the banner is rendered by my own React tree rather than by the vendor, the off-screen
  technique is a CSS class on the first render, not a state toggle after hydration - a state
  toggle happens too late to matter and adds a shift.

Astro:
- The script is an explicit tag under my control, which makes both the `defer` and the
  relocation trivial and testable. Verify the built output rather than the source.

Any stack - the order that matters:
1. **PROVE THE GATE FIRST**: what is written before consent today. That is the constraint.
2. **STOP BLOCKING**: relocate to the tag manager, or at minimum `defer`. Re-run the gate.
3. **PAY THE LATENCY BACK**: `preconnect` (or `preload`) to the vendor origin, because the
   banner now starts later.
4. **THEN, AND ONLY IF IT IS THE LCP ELEMENT**: the off-screen policy, with the honesty test.
5. **SCHEDULE THE CLICK**: prioritise the visual change; let the cascade run after the paint.
6. **ROUTE THE SHIFT** the banner causes to the CLS stage.

## 110.10 Known traps (do not rediscover them the expensive way)
- **Measuring with consent already stored.** The banner never appears and the entire stage
  measures a page that no first-time visitor sees. Clear cookies before EVERY run.
- **Reporting one verdict for "the cookie banner".** Three mechanisms, three fixes. The stage
  exists to keep them apart.
- **`defer` or `async` applied without re-running the consent gate.** The vendor made that
  script synchronous on purpose. Faster and non-compliant is not a win.
- **Relocating to the tag manager and calling it done.** The banner now appears later. Measure
  time-to-banner-visible and pay it back with a preconnect or preload.
- **The off-screen trick applied when the banner is NOT the LCP element.** Buys nothing, costs
  a rule that some future vendor class rename will break silently.
- **The off-screen trick reported as a speed-up.** It changes which element is classified as
  LCP. Nothing got faster. Report it as what it is.
- **Copying vendor CSS selectors from an example.** They differ per vendor and change between
  versions. Read them from the rendered DOM on MY page.
- **Hiding the banner with `display: none` or `visibility: hidden`.** Breaks reachability for
  keyboard and screen-reader users. Move it with `transform`.
- **Assuming the INP failure still exists.** The large vendors shipped scheduling fixes.
  Measure before proposing a workaround.
- **Overlaying your own click target on the vendor's consent button.** It was a legitimate
  workaround when vendors ignored INP; it is fragile, it breaks on any DOM change, and it puts
  you between the user and a legal consent action. Prefer pressure on the vendor. If it is
  genuinely the only option, record it as a workaround with an owner and a review date.
- **Two entry points** - a tag manager AND a hardcoded consent script. The ordering becomes a
  hand-maintained invariant that a future deploy will break without a test.
- **Losing the anonymised Consent Mode pings** while relocating. Analytics the project was
  entitled to keep, gone silently.
- **Auditing at desktop width.** The banner covers a far larger share of a mobile viewport,
  which is exactly why it wins LCP there and not on desktop.
- **`initScript` does not survive the trace's own reload** (verified in Stage 70). Build
  variants as HTML files.

## Save results
- site-profile.md > ## Consent / CMP: the inventory, the three failures with their measured
  costs and separate verdicts, the consent gate result, the relocation decision, the
  off-screen decision with the honesty test, and the per-failure policy table.
- findings.md: dated entry with per-run numbers, the experiment variants and the reasoning.

## STOP line (end every run with this, verbatim intent)
"Consent triage done. Vendor: <name>, loaded <how>, <blocking / not blocking>. Failure 1
render start: <cost from blocked-request counterfactual>, verdict <FIX NOW / FIX LATER / NO
ACTION>. Failure 2 LCP: banner <is / is not> the LCP element (without it: <element>), verdict
<...>. Failure 3 INP: <duration> on the consent click, banner gone after <time>, verdict
<...>. Consent gate: <n> non-essential cookies before consent today; after the proposed change
<re-run and passed / UNPROVEN>. Next: <the single next action>. Measured at <viewport>, cookies
cleared before every run."

## At the end
Check off in site-profile.md > ## Progress: [x] Stage 110 - consent / CMP. Report the three
verdicts, the gate result and the single next action - nothing more.
