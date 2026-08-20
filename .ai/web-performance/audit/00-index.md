# Performance audit - index

Orchestrator for the `web-performance` package. You do not need this file to run a
single stage - each stage in audit/ is self-contained and can be run on its own.

## Execution rules (read first - IMPORTANT)
- This audit is QUESTION-DRIVEN. You interview me; you do not do research on my behalf.
- Every "Ask for" item = ask me the question, STOP, and wait for my answer.
  Do NOT proceed or write anything until I reply.
- Do NOT derive this data yourself - not from the project code, not from MCP, not from
  the browser/network - EVEN IF you can and you "see" the answer. You must ASK me.
- At most you may PROPOSE a value ("from the code it looks like X - confirm?"), but do
  NOT write it until I confirm it or paste my own.
- Fill TODO fields ONLY with my answers or with the output of "Do" items. Do not guess,
  do not pull from your own research, do not pre-fill the profile.
- Ask ONE question at a time. After my answer, write it to
  .ai/web-performance/site-profile.md, then the next question.
- "Do" items (e.g. querying the CrUX API, parsing JSON) you perform yourself.
- Write results and measurements to .ai/web-performance/findings.md.
- Be tool-agnostic. Idempotent: update existing sections, do not duplicate.
- EVIDENCE RULE (applies from Stage 60 on, where the audit starts testing hypotheses
  instead of reading values): compare medians of repeated runs, state the spread, and
  report a difference smaller than that spread as INCONCLUSIVE - never as a win. A
  Lighthouse or DevTools "estimated savings" is a prediction, not a measurement; never
  present one as the other. If the experiment cannot be run, stop at the hypothesis and
  say so instead of substituting an estimate.

## Stage order
- 10 - profile and baseline       -> .ai/web-performance/audit/10-profile-and-baseline.md
- 20 - network: DNS / TLS / HTTP   -> .ai/web-performance/audit/20-network-dns-tls-http.md
- 30 - preconnect (external domains) -> .ai/web-performance/audit/30-preconnect.md
- 40 - cache & CDN (edge)          -> .ai/web-performance/audit/40-cache-cdn.md
- 50 - backend triage (TTFB)       -> .ai/web-performance/audit/50-backend-triage.md
- 60 - render start (first paint)  -> .ai/web-performance/audit/60-render-start.md
- 70 - DOM size (recalc cost)      -> .ai/web-performance/audit/70-dom-size.md
- 80 - scripts & third party       -> .ai/web-performance/audit/80-scripts-and-third-party.md
- 90 - images and video            -> .ai/web-performance/audit/90-images-and-video.md
- 100 - fonts                      -> .ai/web-performance/audit/100-fonts.md
- 110 - consent / CMP              -> .ai/web-performance/audit/110-consent-cmp.md
- 120 - INP / interactions         -> .ai/web-performance/audit/120-inp-interactions.md
- 130 - JS runtime after load      -> .ai/web-performance/audit/130-js-runtime.md
- 140 - navigation and bfcache     -> .ai/web-performance/audit/140-navigation-bfcache.md
- (more stages added per workshop: CLS...)

## What each stage produces (the shape changes as you go up the stack)
- 10-40 read a VALUE: the answer exists and can be measured directly.
- 50 produces a CLASS and a ROUTE: the cause lives inside the server, so the audit
  classifies it from outside and hands you the tool that can see in.
- 60 produces a VERDICT: the value (Start Render) is easy to read but the cause is not,
  and no tool will settle it - only an experiment will. Hence the evidence rule above.
- 70 produces a VERDICT PLUS A PRIORITY: the browser hands you both the cost and the
  culprit, so the hard part is no longer finding the problem - it is deciding whether the
  problem is worth an afternoon next to what Stage 60 found. A stage can legitimately end
  in "real, but not now".
- 80 produces a DECISION PER RESOURCE: the cost is easy to attribute (the trace names the
  script) but the fix is rarely deletion - it is a loading strategy, chosen script by
  script, and constrained by dependencies and by the business. The proof is also cheaper
  than in 60/70: a request can be blocked and re-measured before any code is touched.
- 90 produces TWO SEPARATE VERDICTS from one inventory. Here the measurement is the easy
  part - every image reports its own intrinsic size - and the trap is analytical: the LCP
  element is a LATENCY problem (discovery and priority) while every other image is a
  BANDWIDTH problem (dimension, format, deferral), and the treatment that helps one hurts
  the other. A stage that returns a single verdict for "the images" has got it wrong.
- 100 produces A BUDGET AND A CHOSEN COST. The inventory is trivial here - the browser lists
  the loaded faces in one call - and the difficulty moves to arithmetic and to a trade-off
  that cannot be optimised away. Every lever in the stage competes for the same connection,
  so preload is zero-sum: the stage must output a NUMBER (how many faces may be preloaded,
  usually 0 or 1) and prove it, because preloading all of them frequently measures worse
  than preloading none. And while a font is in flight the page must either hide text or
  shift it - the stage names which cost the project is choosing rather than recommending
  swap by reflex.
- 110 produces THREE SEPARATE VERDICTS AND A GATE. One component fails in three unrelated
  ways - it blocks the parser (loading), it gets classified as the LCP element
  (classification), and its consent click releases a tag cascade before the next paint
  (scheduling) - and two of the fixes work against each other, because whatever stops the
  script blocking also makes the banner appear later. A single verdict for "the cookie
  banner" is the failure mode. This is also the first stage whose output is constrained by
  something that is not a metric: the consent state must still be correct after the change,
  and a loading fix that starts writing cookies before consent is withdrawn no matter what
  it did for Start Render.

- 120 produces A TARGET CHOSEN FROM THE FIELD, A ROUTE AND A SCHEDULING DECISION. Every stage
  before this one measured a PAGE, and the page told you what was wrong. This one measures an
  INTERACTION, and a page has hundreds of them while real users only ever hit a handful - so the
  lab cannot choose the target and RUM has to. Driving a browser to click everything is not a
  substitute: it costs a fortune, and it still measures one device, mine. The diagnosis is then
  the SUBPART, not the total, and it decides who owns the problem: input delay is a loading
  problem (Stages 60/80), presentation delay is a recalculation problem (Stage 70), and only
  processing time belongs to this stage. The fix that follows does not remove work - INP ends at
  the next paint, so it lets a frame through and finishes the rest afterwards. The only fix here
  that genuinely reduces work is deleting work nobody needed. And because a lab click is a
  reproduction rather than the metric, the stage ends SUPPORTED IN LAB, with a date to re-read
  the field.

- 130 produces A COST AFTER DOCUMENT COMPLETE, AN OWNER PER CHUNK AND A DEFERRAL DECISION.
  Every stage before this one asked what delays something the user is waiting for. This one
  starts where the page already looks finished and the main thread is still busy - work nobody
  is waiting for, which is why nobody measures it. The trap here is not measurement but
  inference: a component that APPEARS on scroll is not a component whose CODE runs on scroll,
  and a library that arrived in a shared vendor chunk executed at startup no matter how lazily
  its section fades in. So the stage reads the TRIGGER from source and confirms it in the trace
  rather than guessing it from where the component sits on the page. It also counts the bill
  properly: bytes, parse and compile, execution, memory - four costs, of which the quoted
  transfer size is the cheapest. Deleting beats deferring the code, deferring the code beats
  deferring the render, and `content-visibility` belongs in the last of those groups, not the
  first - it skips layout and paint, not JavaScript.
- 140 produces A POLICY SIZED BY FIELD DATA. It is the first stage that is not about one page
  load but about the visit: the page the user comes back to, and the page they are about to
  open. Both halves are decided by published field data (navigation types), and one of them is
  free - back/forward cache is not built, it is merely not broken, so the finding is which
  blocker is breaking it. Its second half is a budget rather than a switch, because prefetch and
  prerender move origin work earlier without removing it, and a default that speculates every
  visible link turns one visit into dozens of requests. The stage also owns a reporting failure
  that looks like a performance win: a restored page fires no page view, so fixing bfcache
  without a `pageshow` path shows up as a drop in traffic.

## How to resume
Check the "## Progress" section in .ai/web-performance/site-profile.md and run the
first unchecked stage: open its file and execute per the rules above.
