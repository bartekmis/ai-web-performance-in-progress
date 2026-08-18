# Stage 120: INP - a target you cannot choose in the lab, and a fix that reorders work rather than removing it

> Can be run STANDALONE or from the index (.ai/web-performance/audit/00-index.md).
> Behaviour: DO stage, ending in an EXPERIMENT. What is new here is where the target comes
> from. Every stage up to 100 measured A PAGE and the page told you what was wrong. This one
> measures AN INTERACTION - and a page has hundreds, of which real users only ever hit a
> handful. The lab cannot tell you which one matters. Only the field can.
> Read domain/URLs/stack from site-profile.md; if missing, ASK me and WAIT - do not guess.
> Input:  site-profile.md > ## Project, ## Page types, ## Scripts (Stage 80),
>         ## DOM size (Stage 70), ## Consent / CMP (Stage 110)
>         + a RUM tool with INP attribution + the chrome-devtools MCP server + MY SOURCE CODE
> Output: site-profile.md > ## INP / interactions + findings.md, ending with a NAMED TARGET
>         (element and page, chosen from field data), a ROUTE by dominant subpart, a
>         SCHEDULING DECISION, and a measured before/after on that one interaction.
>         Runs AFTER Stage 80 (it names the scripts) and AFTER Stage 70 (it owns recalculation
>         cost, which is where a presentation-delay problem goes).
> Idempotent: if ## INP / interactions exists, update it, do not create a second one.

THIS RUNS AGAINST MY PROJECT, NOT AN EXAMPLE (read first):
The page and the element you measure are MINE - the URL in site-profile.md > ## Project, and an
element that MY users actually click. Never substitute a demo, a documentation example, or a
site mentioned anywhere in this package. If the URL, the page types or the stack are missing
from the profile, ASK me and WAIT.

Any figure in this file describes a MECHANISM or a threshold - not a measurement of my site.
Never copy a figure from here into findings.md and never treat one as a target.

The deliverable is not "split long tasks". It is: THIS element, on THIS page, clicked by real
users on THEIR devices, costing THIS much in THIS subpart, fixed by THIS scheduling change,
with the visual response measured before and after.

## SCOPE AND HARD BOUNDARY (read first - IMPORTANT)

**THE TARGET RULE. This is the rule that makes or breaks the stage.**
INP is not a property of a page load. It is the duration of ONE interaction, and the metric
reports roughly the worst one of a visit. A typical listing page offers tens of clickable
elements; an e-commerce page, hundreds. You cannot measure them all, and measuring the ones
that happen to be convenient is how this stage produces a fix nobody needed.

So the target is CHOSEN FROM FIELD DATA and only then reproduced in the lab. **Field first,
lab second - never the other way round.** A target picked in the lab is a guess with a
decimal point on it.

**And do not automate your way around this.** You can drive a browser through MCP and click
every element on the page. It will work, it will cost a lot of tokens and time, and it will
still be one device, one network, one browser profile - mine, not my users'. Worse, it answers
the wrong question: it finds interactions that are slow FOR YOU, not interactions that are slow
AND FREQUENT for real people. If I ask for it anyway, say plainly what it does and does not
buy.

**THE NEXT-PAINT RULE (this is what the metric measures, and most fixes are misdescribed).**
INP ends at the NEXT PAINT after the interaction, not at the end of my JavaScript. Therefore
the standard fix does not reduce the work the browser has to do - it lets a frame through
first and finishes the rest afterwards. The total work is identical. Anything reported as
"less work" or "we removed 400 ms of processing" is wrong unless work was genuinely deleted
(120.6). Say REORDERED, and say what now lands before the paint.

**THE SUBPART RULE (it decides whether this stage owns the problem at all).**
The three subparts fail for three unrelated reasons and only one of them belongs here:

| dominant subpart | what it means | owner |
|---|---|---|
| input delay | the main thread was busy when the click arrived - usually still loading | NOT this stage: Stages 60 / 80, and see 120.4 on early interactions |
| processing time | my handlers, my framework, or a tag cascade run before the frame | THIS stage: 120.5, 120.6 |
| presentation delay | the visual change itself is expensive: recalculation, layout, paint | Stage 70 (recalc cost), plus the CLS stage for anything that shifts |

A stage that reports "INP is 900 ms, split the long task" without naming the dominant subpart
has skipped the only branch that matters. The subpart is the diagnosis; the total is a symptom.

**THE FIELD CONFIRMATION RULE.** A lab measurement of one click is a REPRODUCTION, not the
metric. The metric moves when real users stop hitting the slow path, on their devices. Every
fix in this stage therefore ends with an open item: re-read RUM after the change has been live
long enough to collect data, and record whether field INP followed the lab result. Until then
the result is SUPPORTED IN LAB, not fixed.

## 120.0 Read what earlier stages already found (Do, no new measurement yet)
- From site-profile.md > ## Scripts (Stage 80): which third-party scripts and which tag manager
  are on the page, and how much main-thread time each already costs. If a tag manager is
  present, expect its cascade to show up on click handlers - 120.3 will confirm or clear it.
- From site-profile.md > ## DOM size (Stage 70): the recalculation cost and whether a
  ForcedReflow insight was recorded. If the dominant subpart turns out to be presentation
  delay, that stage - not this one - already owns the fix.
- From site-profile.md > ## Consent / CMP (Stage 110): failure 3 there is an INP failure on one
  specific button. If it was recorded as FIX NOW, carry it in as a known target and do not
  re-measure it from scratch.
- From site-profile.md > ## Render start (Stage 60): if first paint is still slow, expect early
  interactions with inflated input delay, and read 120.4 before proposing anything here.
- GATE: if Stage 50 says TTFB is the wall or Stage 60 says the page paints in seconds, note it
  and carry it into 120.7. Run the stage anyway - reading RUM is cheap - but the verdict must
  say whether interaction work is next or noise.

## 120.1 Choose the target - FROM THE FIELD (Do)
This step produces one line: which element, on which page, and how much it costs real users.

- Open the RUM tool recorded in ## Measurement and read, for the worst page type:
  - the INP value (p75 and, where available, the worst observed interaction);
  - the **subpart breakdown**: input delay / processing time / presentation delay;
  - the **element selector or label** the tool attributes the interaction to;
  - the interaction type (pointer/click, keyboard input) - keyboard interactions on inputs are
    routinely worse and routinely forgotten;
  - **when in the visit it happens** relative to load. An interaction at second 1 and the same
    interaction at second 10 are different problems (120.4);
  - the attributed script domain where the tool reports one (frequently labelled "primary
    script domain" or similar). My own domain points at my code; a tag or vendor domain points
    at somebody else's.
- Ask me which of these the business actually cares about if two are close. Frequency beats
  severity: an element clicked by 40% of visitors at 300 ms outranks one clicked by 2% at 900.
- **If there is no RUM on the project**, stop and say so. Then: (a) recommend installing one,
  because every subsequent run of this stage is guesswork without it; (b) to keep moving, agree
  with me on a shortlist of AT MOST THREE candidate elements from what the page actually offers
  (mobile menu, a filter, a consent button, the primary CTA); (c) mark the finding
  **UNCONFIRMED IN THE FIELD** everywhere it appears, including the STOP line. Do not quietly
  promote a lab guess to a target.

Record the chosen target and why it was chosen over the others.

## 120.2 Reproduce it in the lab (Do)
Everything here runs through the chrome-devtools MCP server.

- Call `emulate` ONCE: mobile viewport ~412x765x2.6 (mobile, touch), "Fast 4G", **CPU throttling
  4x**. Do not change it for the rest of the stage. State it next to every number.
  Without CPU throttling a modern laptop makes almost any interaction look acceptable and the
  stage produces a clean bill of health for a page that is broken on a mid-range phone.
- Where RUM reports the device class users are actually on, say whether 4x is generous or harsh
  for them. Calibrate if Stage 10 recorded a calibration; otherwise state the assumption.
- `navigate_page` to the page, `performance_start_trace` with `reload: true`, wait for the page
  to settle, perform the ONE interaction from 120.1, then stop the trace.
- Reproduce the user's timing. If RUM said the interaction happens ten seconds into the visit,
  do not click it 200 ms after load - you would be measuring an early interaction that no user
  performs, and input delay would dominate for a reason that is not real.
- In the trace, go to the **interactions** track, find the marker for the click (`pointer`),
  and narrow the selection to it. Record: input delay, processing duration, presentation delay,
  total duration.
- **Reproduction gate.** If the lab number does not resemble the field number, say so and stop
  before proposing a fix. Common causes worth naming: the wrong device class, a logged-out
  versus logged-in page, a cold cache, or an interaction that is only slow when some state is
  present (a full cart, a long list, an open filter). Chase the state, not the code.

## 120.3 Attribute it - whose code runs in that frame (Do)
- In the `main` track under the interaction, read the call stack and record: is the work MY
  bundle, a framework file, a tag manager (`gtm.js` and friends), or another vendor?
- Note that a minified production bundle will not give you a usable function name or line.
  Record what you can see, then read 120.3a before trying to map it to source.
- `debug with AI` / `Find Improvements` in the panel is useful here and it is FAST. Treat its
  output as a HYPOTHESIS with a name attached, never as the verdict: confirm the function
  exists, confirm it runs on this handler, confirm the cost in the trace.
- If the work is a tag cascade, name whether it is the tag manager's own code or tags
  configured to fire on the click event. The fix lives in different places (120.5), and the
  cascade is frequently something the marketing side owns rather than the codebase.

### 120.3a Map it to source - run the project locally (Do)
The audit so far has run against the public site on purpose. This step is the exception.

- On the built site the function is renamed and inlined, so both you and the AI panel will
  point at a bundle offset that means nothing in my repo.
- Start the project locally from source and reproduce the same interaction there. The panel
  now names the real file and line, and the finding becomes something I can act on.
- Where a RUM MCP server is connected, this is also where asking it directly pays off - a
  question like "find the slowest interaction on this site and attribute it to a script" run
  from inside my repo folder gets you the field data and the source correlation in one step.
  It is a shortcut for 120.1 plus this step, not a replacement for the reproduction in 120.2.
- Verify that the local build reproduces the problem before trusting the mapping. A dev server
  with different code splitting can be slower or faster for reasons unrelated to the finding.

## 120.4 Route by dominant subpart (Do - this is the branch, do not skip it)
Take the subpart breakdown from 120.1 (field) and 120.2 (lab) and route:

- **input delay dominates.** The main thread was busy when the click landed. This is a LOADING
  problem, not an interaction problem, and nothing in 120.5 will fix it. Check whether the
  interaction is EARLY - inside the load window - because then the answer is Stage 60 (what
  delays first paint) and Stage 80 (what executes and when). Record the finding here, route it
  there, and say plainly that the button's own code is not at fault.
- **processing time dominates.** This stage owns it. Continue to 120.5 and 120.6.
- **presentation delay dominates.** The handler finishes quickly but the frame is expensive:
  style recalculation over a large subtree, layout, or an animation of properties that trigger
  layout. Route to Stage 70 for the recalculation cost, record what the interaction changes in
  the DOM, and check whether the animated properties are limited to `transform` and `opacity`.
- **Nothing dominates and the total is under 200 ms.** Record **NO ACTION** with the numbers
  attached. That is a legitimate result and, on a project that already did Stages 60-110, it is
  a common one. Do not manufacture work.

## 120.5 The scheduling decision - what lands before the frame (Do)
Only for a processing-time problem. The question is never "how do we make this faster"; it is
**what does the user have to see before anything else is allowed to run**.

1. **Name the visual change.** For this element, what tells the user the click registered - a
   spinner, a state change, an opened panel, a navigation? That, and nothing else, goes first.
   If nothing in the handler produces a visual change, the interaction has a UX defect on top
   of a performance one - say so.
2. **Yield after it.** Two mechanisms, and they are not equivalent:
   - `setTimeout` (a promise wrapping `setTimeout(..., 0)`): universally supported, and it
     works, but the continuation goes to the BACK of the queue - other tasks can get in front
     of it, so the tail can finish later than you expect;
   - `scheduler.yield()`: yields to the browser but keeps the continuation's priority after
     user input is handled. It is the better mechanism where it exists. Support is not
     universal - check the current state rather than assuming, and ship it behind a feature
     check with the `setTimeout` path as fallback (a small `yieldToMain` helper is the usual
     shape).
   Do not present these two as the same fix with different syntax.
3. **Consider the optimistic path.** Where the handler waits on a network write before showing
   anything, the fix may be to show the result immediately and reconcile afterwards. This is a
   product decision as much as a technical one - propose it, do not assume it.
4. **Move genuinely heavy computation off the thread.** A web worker is the right home for a
   long pure computation. It has NO DOM access, so anything that must touch the page comes back
   as a message - if the handler is mostly DOM work, a worker is the wrong tool.
5. **If the cost is a tag cascade**: the visual change goes first and the `dataLayer` push (or
   equivalent) is scheduled after it. Then verify the events still arrive - see the trap list.
   Where the tags themselves are the problem, the cheaper fix is upstream: fewer tags on that
   trigger, decided with whoever owns them.

Record the mechanism chosen and what is now scheduled after the paint.

## 120.6 Is any of this work needed at all? (Do)
Before scheduling work more cleverly, check whether it should run.

- Look at what the handler actually does on MY page: computation whose result is never read,
  measurement of elements that are not animated, a loop sized by a constant nobody has revisited,
  a helper left behind by a refactor.
- **Deleting it is the only fix in this stage that genuinely reduces work**, and it makes the
  before/after unambiguous. If you find it, say so explicitly rather than folding it into a
  scheduling recommendation.
- Confirm with me before removing anything. Code that looks dead can be a side effect somebody
  depends on.

## 120.7 The verdict (Do - STOP and show me)
Before proposing an edit, answer in one short block:

- **target**: element, page, how it was chosen (field data / UNCONFIRMED IN THE FIELD), and how
  often users hit it;
- **numbers**: field INP and subparts; lab total and subparts at the stated viewport and
  throttling; whether the reproduction matched;
- **dominant subpart and the route** it implies (120.4) - including "not this stage" where that
  is the answer;
- **attribution**: my code (file and line, from the local run) or a named third party;
- **decision**: delete (120.6) / reorder with a named mechanism (120.5) / route elsewhere / NO
  ACTION;
- **how this compares** to what Stages 60, 70, 80 and 110 found - is this interaction a visible
  share of the problem, or noise next to what is already open?

Show me this block and wait. Do not continue to the experiment unless I ask.

## 120.8 The experiment (Do - only on a FIX NOW)
- Same emulation and throttling as 120.2, stated with every number.
- Variants: baseline, and one variant per mechanism you are actually proposing. Keep them to
  one change each.
- Where the change is a snippet rather than a source edit, `DevTools` local overrides on the
  HTML document are the cheap way to test it before touching the repo. Say that the measurement
  came from an override.
- **Repeat the interaction at least 5 times per variant, rotated between variants, never in
  blocks.** Discard the first run after each load. Interaction timings are noisier than load
  timings: compare medians and state the spread, and report a difference smaller than the
  spread as INCONCLUSIVE, per the evidence rule in `00-index.md`.
- Record for EVERY variant: total interaction duration, all three subparts, and **time from
  click to the visual change being on screen**. That last number is what the user experiences;
  a variant that improves the metric and delays what the user sees is a REGRESSION.
- Verdicts: SUPPORTED IN LAB / REGRESSION / INCONCLUSIVE. Not "fixed" - see the field
  confirmation rule.
- Open the field item: a date to re-read RUM and confirm the change landed for real users.

## 120.9 OUTPUT - the policy, ROUTED BY STACK (Do)
Do NOT edit yet.

WordPress:
- The handler is usually not mine: it belongs to the theme, a page builder or a plugin, and
  editing it directly will be overwritten on update. Prefer the plugin's own settings, a child
  theme, or dequeueing the script where the feature is unused.
- Tag-manager cascades on click are the common case here. The fix lives in the container, not
  in PHP.
- "Delay JavaScript execution" in a cache plugin postpones scripts until first interaction. It
  can help input delay on early interactions and it can also make the FIRST interaction worse,
  because that click is what triggers the whole bundle. Measure the first interaction
  specifically, and keep the consent script excluded (Stage 110).

Next.js / React:
- Heavy work in an event handler frequently sits inside a component that re-renders for other
  reasons. Name the component and the file from the local run (120.3a), not the bundle.
- The visual change is a state update; the deferred tail belongs after the paint, not in the
  same synchronous block. Do not move the visual change into an effect that runs later.
- Where a third-party script is the cost, the framework's script component with the right
  strategy is the lever, and it belongs outside any component that remounts.

Astro:
- Interaction code is explicit and scoped to the island that owns it, so both the mechanism and
  the measurement are straightforward. Check the built output, not the source, and confirm the
  hydration directive is not doing the work at a different time than you assume.

Any stack - the order that matters:
1. **PICK THE TARGET FROM THE FIELD.** Everything else is downstream of this.
2. **REPRODUCE IT** at a device class my users actually have.
3. **ROUTE BY SUBPART.** Half the time the answer is that this stage does not own it.
4. **DELETE** what nobody needs (120.6) before scheduling it more cleverly.
5. **VISUAL CHANGE FIRST**, the rest after the paint (120.5).
6. **CONFIRM IN THE FIELD** before calling it fixed.

## 120.10 Known traps (do not rediscover them the expensive way)
- **Choosing the target in the lab.** The page has hundreds of interactions; users hit a
  handful. A fix on an element nobody clicks is a change with a cost and no benefit.
- **Letting an agent click everything through MCP to find the slow one.** Unbounded tokens and
  time, and it is still one device and one network - mine. It also answers "what is slow for
  me", not "what is slow for the people who matter".
- **Measuring without CPU throttling.** Almost everything passes on a developer laptop. The
  metric comes from mid-range phones.
- **Clicking immediately after load when users click at second ten** (or the reverse). Input
  delay dominates in the first case for a reason that is not real, and disappears in the second.
- **Reporting a scheduling fix as "less work".** The work is identical; a frame now gets
  through first. INP ends at the next paint, not at the end of the handler.
- **Treating `setTimeout` and `scheduler.yield` as the same fix.** `setTimeout` sends the
  continuation to the back of the queue; `scheduler.yield` preserves its priority. And
  `scheduler.yield` needs a feature check plus a fallback.
- **Shipping `scheduler.yield` bare.** Support is not universal. Check the current state at the
  time of the audit rather than trusting anything written in this file.
- **Trying to fix presentation delay by splitting tasks.** Nothing is queued there; the frame
  itself is expensive. That is Stage 70's recalculation cost.
- **Fixing input delay in the click handler.** Nothing in the handler ran yet. It is a loading
  problem.
- **Mapping a finding from a minified bundle.** Run the project locally from source before
  claiming a file and a line (120.3a).
- **Taking `debug with AI` as a verdict.** It is a fast, useful hypothesis with a plausible
  name attached. Confirm it in the trace.
- **Assuming deferred analytics events get lost.** Verify instead - check the events arrive
  after the change, on both a multi-page navigation and an SPA route change, because the two
  fail differently. And do not assume they are lost simply because they now arrive later.
- **Putting DOM work in a web worker.** It has no DOM access; the result has to come back as a
  message.
- **Overlaying a click target on somebody else's button to capture the interaction.** Fragile,
  and on a consent button it puts you between the user and a legal action (Stage 110).
- **Declaring victory from the lab.** A lab click is a reproduction. The metric moves when the
  field data moves - schedule the re-read.
- **Reporting the total and not the subparts.** The total is a symptom; the subpart is the
  diagnosis and decides who owns the problem.

## Save results
- site-profile.md > ## INP / interactions: the target and how it was chosen, field and lab
  numbers with subparts, the attribution, the route, the mechanism chosen, the experiment
  result and the date set for field confirmation.
- findings.md: dated entry with per-run numbers, the variants and the reasoning.

## STOP line (end every run with this, verbatim intent)
"INP triage done. Target: <element> on <page>, chosen from <RUM / UNCONFIRMED IN THE FIELD>,
hit by <how often>. Field INP <value> (input delay <x> / processing <y> / presentation <z>);
lab reproduction <value> at <viewport, CPU 4x>. Dominant subpart: <name> -> <this stage / Stage
60 / Stage 70 / Stage 80>. Attributed to <my file:line / third party>. Decision: <delete /
reorder via setTimeout / reorder via scheduler.yield with fallback / route elsewhere / NO
ACTION>. Lab result: <SUPPORTED IN LAB / REGRESSION / INCONCLUSIVE>, visual change at <time>
before vs <time> after. Field confirmation due <date>. Next: <the single next action>."

## At the end
Check off in site-profile.md > ## Progress: [x] Stage 120 - INP / interactions. Report the
target, the dominant subpart with its route, the decision and the field confirmation date -
nothing more.
