# Stage 80: Scripts and third party - when JavaScript runs, and what it blocks while it does

> Can be run STANDALONE or from the index (.ai/web-performance/audit/00-index.md).
> Behaviour: DO stage. Like Stage 70 you measure and attribute; unlike Stage 70 the cheap
> proof comes BEFORE any code change - you can delete a script from the page at request
> level and re-measure, without touching the repo.
> Read domain/URLs/stack from site-profile.md; if missing, ASK me and WAIT - do not guess.
> Input:  site-profile.md > ## Project, ## Page types, ## Render start (Stage 60 verdict),
>         ## DOM size (Stage 70 attribution)
>         + the chrome-devtools MCP server + WebPageTest + MY SOURCE CODE
> Output: site-profile.md > ## Scripts + findings.md, ending with a priority verdict and a
>         per-script decision table that names MY files. Runs AFTER Stage 60.
> Idempotent: if ## Scripts exists, update it, do not create a second one.

THIS RUNS AGAINST MY PROJECT, NOT AN EXAMPLE (read first):
The page you measure is MY page - the URL in site-profile.md > ## Project, for the page
type I am auditing. Never substitute a demo, a documentation example, or a site mentioned
anywhere in this package. If the URL, the page types or the stack are missing from the
profile, ASK me and WAIT.

Any figure that appears in this file is there to describe a MECHANISM or a threshold
defined by the tooling - not a measurement of my site. My project's numbers will be
different. Never copy a figure from this file into findings.md and never treat one as a
target.

The deliverable is not "you have too much JavaScript". It is: THIS script, loaded THIS way,
from THIS file in my repo (or from THIS vendor), costs THIS much main-thread time at THIS
point in the load, and here is the change to when it runs.

SCOPE AND HARD BOUNDARY (read first - IMPORTANT):
This stage answers ONE question: which scripts on this page cost measurable time, and can
their EXECUTION be moved, deferred or removed without breaking the page?

Three boundaries:

1. **Weight is not the finding - timing is.** A JavaScript file and an image of identical
   size are not the same cost: the image is decoded largely off the main thread, while the
   script must be transferred, parsed, compiled and executed. Two of those four happen on
   the main thread, where nothing else can run. So never report a script as a problem
   because of its transfer size alone. The question is always: when does it execute, and
   what is waiting behind it?

2. **This stage measures the LOAD, but the fix often lives in INTERACTION.** A tag that
   runs on every click (see 80.5) never shows up in a load trace. If the page has
   interactive analytics, say that the load number is a LOWER BOUND and route the rest to
   the INP stage - do not try to measure INP here.

3. **A script you cannot remove is still in scope.** Business-critical third party stays.
   The output for such a script is a LOADING STRATEGY, not a deletion. A report that only
   says "remove Google Tag Manager" has not done the work.

## 80.0 Read what earlier stages already found (Do, no new measurement yet)
- From site-profile.md > ## Render start: Start Render / FCP / LCP for the slow page type,
  and which resources Stage 60 already deleted or deferred. Scripts Stage 60 dealt with are
  not re-litigated here - carry the decision forward.
- From site-profile.md > ## DOM size: the caller named by the Stage 70 ForcedReflow
  attribution. If a script forces layout, it belongs in this stage's table too - the two
  findings are the same script seen from two sides.
- From site-profile.md > ## Preconnect (Stage 30): which external origins are already
  warmed. You will need this in 80.8.
- GATE: if Stage 50 says TTFB is the wall, or LCP is still measured in seconds, note it and
  carry it into 80.6. Run the stage anyway - the inventory is cheap and useful - but the
  verdict must say out loud whether script work is the next thing to do.

## 80.1 Inventory - what loads, how, and from where (Do)
Everything here runs through the chrome-devtools MCP server.

- Call `emulate` ONCE: mobile viewport ~412x765x2.6 (mobile, touch), "Fast 4G", CPU
  throttling 4x. Do not change it for the rest of the stage. CPU throttling is not optional
  here: parse and compile are the costs this stage is about, and on an unthrottled desktop
  they are close to invisible. An audit run without throttling will conclude that
  JavaScript is free.
- `navigate_page` to the page type you are auditing, then `performance_start_trace` with
  `reload: true` and `autoStop: true`.
- Also `curl` the page and keep the raw server HTML. You need it in 80.3 and 80.5: a script
  present in the DOM but absent from the server HTML was injected by another script, and
  that changes who owns it and which fixes are even possible.

Build ONE row per script, from the server HTML plus the network list:

| script | origin | 1st/3rd party | mode | position | transfer | in server HTML? |
|---|---|---|---|---|---|---|

- **mode** is one of `sync` / `async` / `defer` / `module` / `module+async` / `injected`.
  Read it from the actual tag in the server HTML, not from what the framework claims.
- **position** is `head` or `end of body`, plus the order among other scripts.
- `injected` means the tag is not in the server HTML at all - some other script created it.
  Record its injector if you can see it in the initiator chain.

Then state, in one line each, what the modes actually mean for this page - these are the
mechanics the rest of the stage depends on:
- `sync` in head: the HTML parser stops, fetches, executes, and only then resumes. Worst
  case, and the first thing to look at.
- `async`: downloads in parallel, but executes THE MOMENT it arrives, interrupting the
  parser at an unpredictable point. Order between multiple async scripts is not guaranteed.
- `defer`: downloads in parallel, executes after parsing, in document order.
- `module`: behaves like defer by default; `module` plus `async` executes as soon as it is
  fetched, regardless of its position in the document.

## 80.2 Execution cost - who owns the main thread (Do)
An inventory tells you what loads. It does not tell you what it cost.

- From the trace, read the main-thread work attributed to each script URL. Where the trace
  offers a script-level breakdown, record per script: parse/compile time, execution time,
  and the point in the load at which it ran (before or after First Paint / LCP).
- If the trace lists **`ThirdParties`** or an equivalent third-party insight, call
  `performance_analyze_insight` on it and record the per-vendor main-thread time. This is
  the single most useful number in the stage: it converts "we have a lot of tags" into "this
  vendor owns N ms of the main thread".
- If the trace lists **`ForcedReflow`** (Stage 70 already ran it), cross-reference: a script
  that both executes long AND forces layout is your first candidate regardless of size.
- Repeat the trace 3 times, record the per-script execution time from each, and work out
  YOUR spread. That spread is the noise floor for my page and the only bar that counts.
  Script timing is noisier than structural data - expect it to move between runs.
- Sort the table from 80.1 by MEASURED MAIN-THREAD TIME, not by transfer size. If the two
  orders disagree - and they usually do - say so explicitly in the report. That disagreement
  is the point of this stage.

## 80.3 Third party and SPOF (Do)
Every script from an origin I do not control gets treated as a suspect until proven
innocent. That is the working assumption, not a conclusion.

- For each third-party origin, note the extra cost that a first visit pays before a single
  byte of the script arrives: DNS lookup, TCP connect, TLS handshake. Check against Stage 30
  which of these origins are already warmed with `preconnect`.
- **SPOF check.** Flag every third-party script that sits in `head` WITHOUT `async` and
  WITHOUT `defer`. That is a single point of failure: if the vendor's host is slow or down,
  the parser stops and the page may never render. This is not a theoretical risk and it is
  the highest-severity finding this stage can produce - report it above any timing result.
- Prove it rather than asserting it: run a WebPageTest test with the vendor's domain
  blackholed (WebPageTest's SPOF tab does exactly this - it is a DIFFERENT test from
  DevTools request blocking, which only removes the request). Record what happened to Start
  Render. A page whose first paint is tied to a vendor's availability is a business risk,
  not just a performance one.
- For sizing the opportunity, `third-party-web` (the public dataset of real-world third-
  party costs) is useful for ARGUMENT and prioritisation - "this category of tag typically
  costs this much" - but it is a population median, not a measurement of my page. Never put
  a number from it in findings.md as if I had measured it. My number comes from 80.2.

## 80.4 Coverage - what can be DEFERRED, not what can be deleted (Do)
Use the DevTools coverage data (or the equivalent unused-bytes report) for each script.

READ THIS BEFORE CONCLUDING ANYTHING. Coverage reports bytes not executed UP TO THE MOMENT
OF MEASUREMENT. It does not report dead code. A modal handler, a form validator, a carousel
below the fold and an off-canvas menu will all appear ~100% unused on a load trace, because
nobody has clicked anything yet. Deleting them does not optimise the page - it breaks it.

So split every "unused" figure into three buckets, and never report the raw total as a
saving:
- **dead** - not reachable at all on this page type (a feature that does not exist here, a
  library kept for a removed component). This one can be deleted; prove it by searching my
  repo, not by trusting coverage.
- **needed later** - reachable only after an interaction or after scrolling. This is the
  bucket the stage exists for: it does not get deleted, it gets LOADED LATER.
- **needed now** - executes during load and is required for what the user first sees. Stays.

A Lighthouse "Reduce unused JavaScript" saving is a PREDICTION, not a measurement (see the
evidence rule in 00-index.md). Cite it as a hypothesis to test in 80.7, never as a result.

## 80.5 Tag managers - the cascade and the click cost (Do, if a tag manager is present)
A tag manager is not one script, it is a loader for an unknown number of others. Audit it as
a cascade:

1. the container script is fetched from a vendor origin (DNS/TCP/TLS, then transfer);
2. it executes, and only then do its tags fire;
3. tags inject further scripts into the page, each with its own origin and its own
   connection cost;
4. those scripts finally load and execute.

Each step is serialised behind the previous one, so a tag manager loaded early does not mean
its tags run early - and a tag manager loaded late delays an entire tree of vendors.

- Record the cascade for my page: container -> tags -> injected scripts, with the timing of
  each level from 80.2. Scripts at level 3 are the ones that will be marked `injected` in
  80.1 and that will NOT be findable in my repo - say so rather than proposing an edit I
  cannot make.
- **Ask me to export the container configuration** (Admin > Export Container, a JSON file)
  and analyse it: which tags exist, which fire on which trigger, which are duplicated,
  which have not fired in the trace at all. Containers accumulate tags that someone added
  for a test and never removed. This is a question-driven step: ASK, then WAIT for the file.
  Do not guess the container's contents from the page.
- **The click cost (route this to INP, do not measure it here).** If any tag fires on an
  interaction trigger - click, form submit, phone/e-mail link, menu open - then every such
  interaction runs the tag's JavaScript inside the same task as the user's click, before
  the browser can paint the result. That cost does not appear anywhere in a load trace.
  Record which triggers are interaction-based, state that the load-time figure is a LOWER
  BOUND, and hand the list to the INP stage.
- Note for the fix list (80.8): deferring the container also defers analytics. That is a
  business decision, not a performance one - present it as a trade-off with a named owner,
  never as a free win.

## 80.6 The priority verdict (Do - STOP and show me)
Before proposing a single fix, answer in one short block:
- total main-thread time attributed to scripts, split first party vs third party, with the
  spread from 80.2;
- any SPOF findings from 80.3, listed first regardless of their timing cost;
- what else is on this page: LCP, FCP, Start Render from Stage 60, and the Stage 70 verdict;
- the ratio: what share of this page's problem is script execution?
- verdict, one of:
  - **FIX NOW** - a SPOF exists, or script execution is a visible share of the page's cost.
  - **FIX LATER** - real but small next to what Stage 50/60 found. Say what to do first.
  - **NO ACTION** - execution cost inside the noise floor and no SPOF. Record the inventory
    as hygiene and stop.

Show me this block and wait. Do not continue to the experiment on a FIX LATER or NO ACTION
verdict unless I ask.

## 80.7 The experiment (Do - only on FIX NOW)
This stage has a cheaper proof than Stage 60 or 70: you can remove a script from the page
without building anything.

- Pick ONE script with the largest expected effect.
- **Round 1 - block the request.** Use DevTools request blocking (block by URL or by domain)
  or the WebPageTest equivalent, then re-measure. 3 rounds, rotated A,B,A,B - never in
  blocks. Discard the first warm-up load. Compare medians and state the spread.
- **CHECK THE PAGE STILL WORKS while blocked.** This is the trap that makes this experiment
  dangerous: blocking a script that the layout depends on will show a beautiful improvement
  in a page that is now broken - content missing, images unloaded, half the DOM never built.
  A number from a broken render is not a result. Confirm the page renders the same content
  before you record any win.
- **Round 2 - only if Round 1 was SUPPORTED.** Now test the change you would actually ship,
  which is usually NOT deletion but a different loading strategy: move `sync` to `defer`,
  move a tag behind first paint, load a widget on interaction. Build it as a local HTML
  variant exactly as in Stage 60.5 (download the page, inject `<base href="{PAGE_URL}">`,
  one edit per variant, plus an unmodified baseline), serve locally, and measure the same
  way. Do NOT mutate the page at runtime with `initScript` - it does not survive the
  reload that `performance_start_trace` performs, and you will "prove" that your fix does
  nothing (verified in Stage 70).
- Compare local variant against LOCAL baseline only. A local copy loads a different set of
  third-party scripts than production - never put a local number next to a production
  number in the same table.
- Verdicts as in Stage 60: SUPPORTED / REGRESSION / INCONCLUSIVE, where the bar is the
  larger of the two spreads.

## 80.8 OUTPUT - per-script decisions, ROUTED BY STACK (Do)
For each script in the table, one decision. Do NOT edit yet.

The decision tree, in this order:
1. **Is it needed on this page at all?** If no - remove it. Prefer conditional loading over
   a per-URL exception list: a rule that says "not on the contact page" breaks the moment
   the client edits that page. Detect the component or feature, and load accordingly.
2. **Does it render something above the fold?** If yes - it stays early, but with `defer`,
   in `head`, after the CSS.
3. **Does it have dependencies or require order?** If yes - `defer`, never `async`. Async
   between dependent scripts is not a performance trade-off, it is a bug: the order is not
   guaranteed and the scripts will fail.
4. **Is it needed only after an interaction or after scrolling?** If yes - load it on
   demand: a dynamic import on the event, an `IntersectionObserver` for the section, or the
   framework's own primitive.
5. **Is it third party?** Then the levers are loading strategy, configuration, or dropping
   the vendor - not a source edit. Warm the connection with `preconnect` (Stage 30) even
   when you defer the script itself: the connection can be established early while the
   file is fetched late.

WordPress:
- `wp_enqueue_script` takes a loading strategy (`defer` / `async`) in current versions -
  that is the supported way to change a plugin's script, not editing the plugin.
- `wp_dequeue_script` for what a page does not need; confirm the handle ID first.
- Classic candidates for removal after verification: jQuery Migrate (a compatibility shim
  for old jQuery), and polyfills for browsers my users do not run - check my analytics
  before claiming this, do not assume.
- jQuery in `head` with dependent plugin scripts at the end of `body` is a common shape:
  moving the whole chain in front of the closing `body` tag with `defer` preserves order
  while freeing the parser. Test it - it does not always win.
- Minification and concatenation are separate, real wins on bytes. Do not report them as
  execution-time fixes.

Next.js / React:
- `next/script` with `afterInteractive` for third party that must run but not first;
  `lazyOnload` for anything that can wait for idle.
- Framework chunks already load with `defer` in `head` - do not "fix" that.
- `next/dynamic` for components whose code is only needed after an interaction.
- Barrel files (`index.ts` re-exporting a folder) quietly defeat code splitting: importing
  one component pulls the whole module graph into the chunk. Import from the target path,
  or evaluate `optimizePackageImports`.
- Check what the bundle actually contains (`@next/bundle-analyzer`, or
  `rollup-plugin-visualizer` for Vite/Rollup) before optimising a guess.

Astro:
- `client:visible` for anything below the fold, `client:idle` for the rest, `client:only`
  where SSR is pointless. Astro ships no JS by default, so any hydrated island here is a
  deliberate choice - revisit each one.

Any stack - the order that matters:
1. **DELETE** what is not needed on this page type.
2. **MOVE THE EXECUTION** - defer, or push behind first paint.
3. **LOAD ON DEMAND** what only an interaction needs.
4. **WARM THE CONNECTION** for what stays external (`preconnect` for the critical ones,
   `dns-prefetch` for the rest) - but never to an origin nothing is fetched from.

SEO note: crawlers execute JavaScript, so deferring is normally safe - but content that
must be indexed should not sit behind an interaction a crawler will never perform.

## 80.9 Known traps (do not rediscover them the expensive way)
- **`async` plus `defer` on the same tag: `async` wins and `defer` is ignored.** This
  combination is a legacy pattern from when `defer` support was incomplete. If you find it,
  report it as an anti-pattern to clean up, and be aware the tag is behaving as `async` -
  which matters if the script has dependencies.
- **`defer` is not free.** It moves WHEN the script runs; it does not make the work
  disappear. The script still executes on the main thread and can still delay
  interactivity. A report claiming "we deferred it, so it costs nothing now" is wrong.
- **CSS blocks script execution.** A stylesheet blocks the execution of every script that
  follows it in the document, including `async` and `defer` ones, because a script may
  query computed styles. So a heavy stylesheet can delay a script that looks entirely
  independent of it. If the timeline shows a script waiting with no obvious cause, check
  what CSS precedes it.
- **Coverage measures "unused so far", not "dead".** See 80.4. This is the most common way
  to turn this stage into a broken page.
- **Blocking a request can produce a fake win.** See 80.7. Always confirm the page still
  renders the same content while blocked.
- **A script in the DOM may not be in the HTML.** Anything injected by a tag manager or by
  another script cannot be changed by editing my templates. Check the server HTML with
  `curl` before designing a fix, and route such findings to the vendor's configuration.
- **Never conclude from an unthrottled run.** Parse and compile are several times more
  expensive on a mid-range phone than on the machine running the audit. Without CPU
  throttling this stage will report that JavaScript is cheap.
- **`initScript` does not survive the trace's own reload** (verified in Stage 70). Build
  variants as HTML files.
- **A vendor's median cost is not my cost.** `third-party-web` is for prioritising and for
  arguing with stakeholders, not for filling in findings.md.
- **Interaction cost is invisible here.** Tags that fire on click do not appear in a load
  trace. Route them to the INP stage rather than declaring the tag cheap.

## Save results
- site-profile.md > ## Scripts: the inventory table with measured main-thread time, the
  first/third-party split, SPOF findings, the tag-manager cascade, the coverage buckets,
  the priority verdict, and the per-script decision table.
- findings.md: dated entry with the per-run numbers, the experiment rounds and the
  reasoning.

## STOP line (end every run with this, verbatim intent)
"Script triage done. Main-thread time: <first party> / <third party>, spread <x>. SPOF:
<none, or the named vendor>. Heaviest: <script, ms, when it runs>. Coverage split:
<dead / later / now>. Priority: <FIX NOW / FIX LATER / NO ACTION> because <one clause
comparing it to LCP/FCP>. Next: <the single next action>. Remember this measured the LOAD -
tags firing on interaction cost extra and show up in INP, not here."

## At the end
Check off in site-profile.md > ## Progress: [x] Stage 80 - scripts and third party. Report
the priority verdict and the single next action - nothing more.
