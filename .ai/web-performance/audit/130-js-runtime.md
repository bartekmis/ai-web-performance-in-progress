# Stage 130: JS runtime after load - work that keeps running when the page already looks ready

> Can be run STANDALONE or from the index (.ai/web-performance/audit/00-index.md).
> Behaviour: DO stage, ending in an EXPERIMENT. What is new here is WHEN you look. Every stage
> up to 120 asked what delays something the user is waiting for - the first paint, the largest
> image, the response to a click. This one starts AFTER Document Complete, at the moment the
> page looks finished and the main thread is still busy. Nobody is waiting for this work, which
> is exactly why nobody measures it.
> Read domain/URLs/stack from site-profile.md; if missing, ASK me and WAIT - do not guess.
> Input:  site-profile.md > ## Project, ## Page types, ## Scripts (Stage 80),
>         ## DOM size (Stage 70), ## Render start (Stage 60), ## INP / interactions (Stage 120)
>         + the chrome-devtools MCP server + a WebPageTest run + MY SOURCE CODE
> Output: site-profile.md > ## JS runtime + findings.md, ending with A COST AFTER DOCUMENT
>         COMPLETE, AN OWNER PER CHUNK, and A DEFERRAL DECISION that distinguishes deferring
>         the RENDER from deferring the CODE.
>         Runs AFTER Stage 80 (it names the scripts and their loading strategy) and AFTER
>         Stage 70 (it owns recalculation cost).
> Idempotent: if ## JS runtime exists, update it, do not create a second one.

THIS RUNS AGAINST MY PROJECT, NOT AN EXAMPLE (read first):
The page, the chunks and the components are MINE - the URL in site-profile.md > ## Project.
Never substitute a demo, a documentation example, or a site mentioned anywhere in this package.
If the URL, the page types or the stack are missing from the profile, ASK me and WAIT.

Any figure in this file describes a MECHANISM or a threshold - not a measurement of my site.
Never copy a figure from here into findings.md and never treat one as a target.

The deliverable is not "use dynamic import". It is: THIS chunk, executing THIS long after the
page was visually complete, owned by THIS component, deferred by THIS mechanism, measured
before and after.

## SCOPE AND HARD BOUNDARY (read first - IMPORTANT)

**THE LAZY-APPEARANCE RULE. This is the rule that makes or breaks the stage.**
A component that APPEARS on scroll is not the same thing as a component whose CODE runs on
scroll. A section can fade in on an intersection observer, or slide in on a scroll-driven
animation, while the library that powers it was downloaded, parsed and executed at startup -
inside a shared vendor chunk, from a `DOMContentLoaded` handler, in module scope. It looks lazy
and it is not.

This is also why the cost hides. Work triggered at startup is SMEARED across the load: a bit of
parse, a bit of compile, a bit of execution, spread over several tasks, none of which is
obviously "the chart". You will not find it by looking for a long task at the moment the widget
appears, because nothing happens at that moment - it already happened.

**So the question this stage asks is never "when does it appear". It is "when does it execute".**
Prove it, do not infer it: block the chunk, or breakpoint the constructor, or read the trace.

**THE BOUNDARY RULE (what this stage does NOT own).**
| symptom | owner |
|---|---|
| a script blocks the parser or is fetched too early / too late | Stage 80 (loading strategy) |
| first paint is late | Stage 60 |
| style recalculation over a large subtree, layout thrash | Stage 70 |
| a specific click is slow | Stage 120 (and its subpart rule) |
| main thread busy AFTER the page is visually complete | THIS stage |

The overlap with Stage 80 is real and worth stating: Stage 80 decides WHEN a script is fetched
and whether it blocks. This stage decides whether the code inside it should RUN AT ALL on this
page load. A script can be perfectly `defer`red, arrive at exactly the right time, and still
execute 300 ms of work nobody needed.

**THE FOUR-COSTS RULE.** Bytes are the first item on the bill, not the whole bill. Every
kilobyte of JavaScript is paid four times: download, parse and compile, execution on the main
thread, and memory plus the GC that follows. A stage that reports only transfer size has
measured one quarter of the problem - and usually the cheapest quarter, because compression
flatters the number that gets quoted and does nothing for the three that follow.

**THE THREE QUESTIONS.** Every finding in this stage answers one of them, and they are asked in
this order:
1. **Does this code have to be here?** (duplicated libraries, a barrel file pulling a whole
   package, a dependency used on one page and shipped on all of them)
2. **Does it have to run now?** (module-scope work, `DOMContentLoaded` handlers, hydration of a
   component nobody has scrolled to)
3. **Does it have to run again?** (re-renders, scroll and resize handlers, animation frames that
   never stop)

Deleting beats deferring, and deferring beats optimising. Do not reorder this.

## 130.0 Read what earlier stages already found (Do, no new measurement yet)
- From site-profile.md > ## Scripts (Stage 80): the inventory, the loading strategy per script
  and the main-thread cost already attributed. This stage does not re-inventory - it asks what
  the already-named code DOES after the page is complete.
- From site-profile.md > ## DOM size (Stage 70): recalculation cost and any ForcedReflow
  insight. If the runtime cost turns out to be layout rather than script, that stage owns it.
- From site-profile.md > ## INP / interactions (Stage 120): if input delay dominated there, the
  main thread was busy when users clicked - and what was keeping it busy is very likely what
  this stage is about to find. Carry the finding in.
- From site-profile.md > ## Render start (Stage 60): note whether the page paints fast. A fast
  paint with a long busy tail afterwards is the exact profile this stage exists for, and it is
  invisible in every metric that stops at LCP.
- GATE: if Stage 50 says TTFB is the wall or Stage 60 says the page paints in seconds, run this
  stage anyway (reading a trace is cheap) but the verdict must say whether this is next or
  noise.

## 130.1 Find the tail - what runs after the page is complete (Do)
Everything here runs through the chrome-devtools MCP server.

- Call `emulate` ONCE: mobile viewport ~412x765x2.6 (mobile, touch), "Fast 4G", **CPU throttling
  4x**. Do not change it for the rest of the stage. State it next to every number.
- `navigate_page`, then `performance_start_trace` with `reload: true`, and **let it run several
  seconds past the point where the page looks finished. Do not scroll, do not click.** That is
  the whole trick of this stage: the trace has to cover the period when nobody is waiting.
- Record, from the trace:
  - **Total Blocking Time**, and the individual long tasks that make it up (a task counts from
    50 ms; the blocking part is the excess over 50 ms, which is why five 60 ms tasks and one
    300 ms task are not the same finding even when the totals look similar);
  - **where the last long task ends relative to Document Complete and to LCP.** This single
    number - "the main thread was still busy N seconds after the page looked done" - is the
    headline of the stage;
  - for each long task in that tail: the top frames, and which file they belong to.
- Cross-check on a WebPageTest run of the same page: the JS execution bands on the waterfall,
  read against Document Complete. The lab trace tells you what; the waterfall tells you how it
  looks in a shape most people find easier to argue with.
- **If the tail is empty** - the thread goes quiet within a second of the page being complete
  and TBT is small - record **NO ACTION** with the numbers and stop. That is a legitimate
  result and the fastest possible run of this stage.

## 130.2 Attribute it - which chunk, and what does it contain (Do)
- Name the file for each long task in the tail. On a built site it will be a hashed bundle
  (`vendor-<hash>.js` and friends), which tells you nothing yet.
- **Block the request and reload.** In DevTools Network, block that one URL, reload, and watch
  what visually breaks or disappears. That is the cheapest possible mapping from an anonymous
  chunk to a feature I can name. It is a DIAGNOSTIC, not a fix - never report a blocked request
  as an improvement.
- Ask what else travels with it. A vendor chunk frequently carries several unrelated libraries
  because the build put everything shared into one file. Two libraries that have nothing to do
  with each other, loaded on every page because one of them is needed on one page, is a
  finding in its own right - and its fix is a build-config change, not a code change.
- **Map it to source by running the project LOCALLY.** The public site is minified; neither you
  nor the AI panel can turn a bundle offset into a file and a line. Start the project from
  source, reproduce the same trace, and give me the real file. Verify the local build actually
  reproduces the finding before trusting the mapping - different code splitting in dev can move
  the cost somewhere else entirely.
- `debug with AI` / `Find Improvements` in the Performance panel is fast and useful here. Treat
  what it says as a HYPOTHESIS with a plausible name attached, and confirm it in the trace.
  Its recurring recommendations in this area - reduce DOM complexity, optimise style
  recalculation, defer non-critical content - are all real, and the first two belong to Stage
  70, not here. Route them, do not re-solve them.

## 130.3 When does it execute, and who asked for it (Do - the branch)
For each named chunk or component in the tail, establish the TRIGGER. Read it in the source,
confirm it in the trace:

- **module scope** - runs the moment the module is evaluated, i.e. as early as the chunk
  arrives. Data generation, instance construction and configuration parsing at module top level
  are the classic ones;
- **`DOMContentLoaded` / `load` handler** - runs at startup regardless of where the component
  sits on the page. This is the trigger behind most "but it is below the fold" surprises;
- **an intersection observer** - genuinely deferred, IF what it triggers is the IMPORT and not
  merely the render. Read which one it is. An observer that only reveals a section, while the
  library is already in the main bundle, is the lazy-appearance trap;
- **a framework directive** - `client:visible` and friends in Astro, a dynamic import in Next.js
  or React. Confirm in the BUILT OUTPUT that the code really landed in a separate chunk. A
  directive that was configured wrong fails silently and looks correct in source;
- **hydration** - see 130.4;
- **a repeating source** - scroll or resize handler, `requestAnimationFrame` loop, timer,
  observer that never disconnects. Anything still ticking when the page is idle belongs here.

Then ask, per component, the question the trigger cannot answer: **does the user ever see this
on this page?** Work done for a section that 90% of visits never scroll to is not "fast enough
to leave alone" - it is work with no audience.

## 130.4 Hydration and re-renders - framework stacks only (Do)
Skip this if the project is not component-hydrated. Record why it was skipped.

- **Hydration cost is proportional to the size of the component tree, not to what is on
  screen.** The server sent markup that looks finished; the framework then walks the tree,
  recreates components and attaches listeners. Measure it as part of the tail in 130.1 rather
  than assuming a number.
- Ask what could be excluded from it: components that are static in practice, sections below
  the fold, anything that could be an island or arrive behind a suspense boundary instead of
  being hydrated with everything else at once.
- **Re-renders are the cost that does not appear on any waterfall.** A state update high in the
  tree re-renders the subtree below it; a new object, a new inline function or a new reference
  in props does the same for a child that did not need it. Long lists without virtualisation
  multiply the effect on every filter change.
- To see them, use a re-render highlighter (React Scan is the usual choice, and it can be
  injected through DevTools local overrides as a script in `<head>` before any other script -
  no install in the project needed). Then record a Performance trace WITH CPU throttling while
  performing a realistic interaction, so the highlights turn into main-thread time rather than
  a light show.
- **Highlighted re-renders are not automatically a defect.** Ask which of them had to happen. A
  parent re-rendering because its own state changed is correct; twenty children re-rendering
  because a prop identity changed is not.
- A deterministic repo scanner (React Doctor and similar) is worth a run here: it costs no
  tokens, it reports state, effect and performance categories against known patterns, and it can
  sit in CI. Use it for the INVENTORY and hand the shortlist to an agent for the FIXES - that
  split is cheaper than asking a model to both find and fix.
- **A compiler that automates memoisation** (React Compiler and equivalents) moves the decision
  from a human to the build. Two things follow, and both belong in the finding: the code gets
  simpler and a class of stale-memo bugs disappears; and the size of the win is a function of
  how bad the code was to begin with. On a codebase that already memoises deliberately, expect
  little. On an older one, expect a lot. Never present it as a substitute for the measurement -
  measure before and after like anything else.

## 130.5 The deferral decision - render or code (Do)
For every candidate from 130.3, state which of these you are proposing. They are not
interchangeable and they do not cost the same:

1. **Delete.** The dependency is unused, duplicated, or pulled in by a barrel import that takes
   the whole package for one function. This is the only option here that removes all four costs.
   Confirm with me before removing anything.
2. **Defer the CODE.** Split it into its own chunk and import it at the moment it is needed -
   inside an intersection observer, behind a framework directive, on the interaction that
   reveals it. Where a shared vendor chunk carries an unrelated heavy library, the split itself
   is a build-config change (`manualChunks` or its equivalent) and is often the highest-value
   line in the whole stage.
3. **Defer the WORK, keep the code.** Sometimes the module has to be there but its expensive
   initialisation does not have to run at startup: construct on first view, compute on demand,
   size the dataset to what is actually rendered.
4. **Defer the RENDERING to the browser.** `content-visibility: auto` with
   `contain-intrinsic-size` lets the browser skip layout and paint for offscreen sections. It is
   CSS, it needs no build step, and on a page built from someone else's components it is
   frequently the only lever available. Be precise in the finding: **it defers rendering, not
   JavaScript.** A heavy script inside that section still downloads, parses and executes.
5. **Stop the repetition.** Disconnect observers, throttle or passive-ify scroll handlers, end
   animation frames when nothing is animating, memoise deliberately where a compiler is not in
   play.

Record the mechanism per component and what will now happen instead of what happens today.

## 130.6 The verdict (Do - STOP and show me)
Before proposing an edit, answer in one short block:

- **the tail**: TBT, the long tasks that make it up, and how long the main thread stayed busy
  after Document Complete, at the stated viewport and throttling;
- **the owners**: per chunk - what it contains, what broke when you blocked it, the source file
  from the LOCAL run;
- **the triggers**: per component - module scope / `DOMContentLoaded` / observer / directive /
  hydration / repeating, and whether the trigger matches what a reader of the UI would assume;
- **lazy appearance vs lazy code**: name explicitly any component that appears deferred but
  executes at startup. If there is none, say so - it is a good result and worth recording;
- **the decision**: delete / defer the code / defer the work / defer the rendering / stop the
  repetition / NO ACTION, one per candidate;
- **how this compares** to what Stages 60, 70, 80 and 120 found. Is the runtime tail a visible
  share of the problem or noise next to what is already open?

Show me this block and wait. Do not continue to the experiment unless I ask.

## 130.7 The experiment (Do - only on a FIX NOW)
- Same emulation and throttling as 130.1, stated with every number.
- One change per variant. Baseline first.
- **Record all four costs, not just the first**: transfer size of the critical chunks, parse and
  compile time, script evaluation time, and the position of the last long task relative to
  Document Complete. A change that moves bytes without moving the tail has not fixed what this
  stage is about.
- Also record TBT and, where the change touches a component the user interacts with, the
  interaction cost from Stage 120. Deferral can trade one for the other: code that now loads on
  scroll has to execute at some point, and if that point is the moment the user reaches the
  section, you have moved a load-time cost into an interaction-time cost. Measure it there too.
- **Repeat at least 5 runs per variant**, compare medians, state the spread, and report a
  difference smaller than the spread as INCONCLUSIVE, per the evidence rule in `00-index.md`.
- Verify the FEATURE still works, on a mid-range device and a slow connection. A chunk that now
  arrives when the user reaches the section can arrive visibly late. If it does, the answer is
  usually a margin on the observer, not a rollback.
- Verdicts: IMPROVED / REGRESSION / INCONCLUSIVE, with the numbers.

## 130.8 OUTPUT - the policy, ROUTED BY STACK (Do)
Do NOT edit yet.

WordPress:
- The heavy component usually belongs to a theme, a page builder or a plugin, and editing it
  directly will be overwritten on update. In order: check whether the feature can be turned off
  or configured lighter; dequeue the script on page types that do not use it; replace the
  component with a lighter one; only then override it.
- `content-visibility: auto` with `contain-intrinsic-size` on offscreen sections is the minimum
  that is always available and needs nobody's permission. Say plainly that it defers rendering,
  not script execution.
- A custom theme is not a free pass. Hand-built frontends land here too: one shared bundle,
  every library in it, on every page, because that is what the default build config does.

Next.js / React:
- `next/dynamic` (or a bare dynamic import) plus an intersection observer is the standard
  mechanism, and the observer is the part people skip - a dynamic import that still runs at
  mount has moved nothing.
- Check what the vendor chunk actually contains before splitting anything. The fix is
  frequently in the build config, not in a component.
- Re-renders: measure with a highlighter, fix by identity (stable references, state placed
  where it is used) and by virtualising long lists. A memoising compiler is a legitimate lever,
  measured like any other.

Astro:
- Islands make this stage easy: the directive is explicit, and `client:visible` defers the code,
  not just the render. Verify in the BUILT output that the component really landed in its own
  chunk and check the observer margin.
- Being on Astro is not immunity. A `client:load` island with a heavy library behaves exactly
  like a WordPress bundle.

Any stack - the order that matters:
1. **MEASURE THE TAIL AFTER DOCUMENT COMPLETE.** No trace, no finding.
2. **NAME THE OWNER** by blocking the chunk, then map it to source from a LOCAL run.
3. **READ THE TRIGGER**, do not infer it from where the component sits on the page.
4. **DELETE** before deferring, **DEFER THE CODE** before deferring the render.
5. **MEASURE ALL FOUR COSTS** before and after, medians with spread.

## 130.9 Known traps (do not rediscover them the expensive way)
- **Mistaking lazy appearance for lazy code.** The section fades in on scroll; the library ran
  at startup. This is the single most common finding in this stage and the easiest to miss,
  because the moment the component appears is exactly the moment nothing happens.
- **Stopping the trace when the page looks finished.** The whole point is what happens next.
- **Measuring without CPU throttling.** A modern laptop chews through a startup tail that
  freezes a mid-range phone.
- **Reading only transfer size.** Compression flatters bytes and does nothing for parse,
  execute and memory.
- **Treating a blocked request as a fix.** It is a diagnostic. The feature is gone, not
  deferred.
- **Mapping a finding from a minified bundle.** Run the project locally from source first.
- **Assuming below the fold means not executed.** The trigger decides, not the position.
- **Assuming a dynamic import is enough.** If it still runs at mount, nothing moved.
- **Splitting a chunk without looking inside it.** Two unrelated libraries in one vendor file is
  a build-config finding; splitting the wrong one costs an afternoon and changes nothing.
- **Selling `content-visibility` as a JavaScript fix.** It defers layout and paint. The script
  inside still runs.
- **Reporting highlighted re-renders as defects without asking which had to happen.** Some of
  them are the framework working correctly.
- **Expecting a memoising compiler to rescue good code.** The win scales with how bad the code
  was. Measure, do not assume - in either direction.
- **Trading a load-time cost for an interaction-time cost silently.** Deferred code executes
  when the user arrives. If that is at the moment of a click, Stage 120 now owns a new problem.
- **Forgetting the repeating work.** A scroll handler or an animation frame loop that never
  stops does not show up as one long task - it shows up as a thread that is never idle.
- **Optimising a page nobody profiled.** The tail on the homepage and the tail on a listing page
  are different problems. Say which page type the numbers came from.

## Save results
- site-profile.md > ## JS runtime: the tail measured after Document Complete, the chunks with
  their contents and owners, the trigger per component, the lazy-appearance verdict, the
  deferral decision per candidate, and the experiment result.
- findings.md: dated entry with per-run numbers, variants and reasoning.

## STOP line (end every run with this, verbatim intent)
"JS runtime triage done on <page type>. Main thread busy for <N> s after Document Complete, TBT
<value> at <viewport, CPU 4x>, longest task <value> in <chunk>. Owner: <chunk> contains
<libraries>, mapped to <file from local run>. Trigger: <module scope / DOMContentLoaded /
observer / directive / hydration / repeating>. Lazy appearance without lazy code: <yes - which /
no>. Hydration and re-renders: <finding or skipped - why>. Decision: <delete / defer code /
defer work / defer rendering / stop repetition / NO ACTION>. Experiment: <IMPROVED / REGRESSION
/ INCONCLUSIVE>, last long task <before> -> <after>. Next: <the single next action>."

## At the end
Check off in site-profile.md > ## Progress: [x] Stage 130 - JS runtime after load. Report the
tail, the owner, the trigger, the decision - nothing more.
