# Stage 140: Navigation - the page you already paid for, and the one you are about to

> Can be run STANDALONE or from the index (.ai/web-performance/audit/00-index.md).
> Behaviour: DO stage, ending in A POLICY. Every stage before this one optimised ONE page load.
> This one is about the visit: the page the user already loaded and comes back to, and the page
> they are about to open. Both have a native browser mechanism, both are decided by field data,
> and one of them is free.
> Read domain/URLs/stack from site-profile.md; if missing, ASK me and WAIT - do not guess.
> Input:  site-profile.md > ## Project, ## Page types, ## Measurement, ## Cache & CDN (Stage 40),
>         ## Consent / CMP (Stage 110)
>         + CrUX (or any tool exposing navigation types) + the chrome-devtools MCP server
> Output: site-profile.md > ## Navigation + findings.md, ending with A BFCACHE VERDICT with its
>         named blockers, AN ANALYTICS CHECK, and A SPECULATION POLICY sized by field data.
> Idempotent: if ## Navigation exists, update it, do not create a second one.

THIS RUNS AGAINST MY PROJECT, NOT AN EXAMPLE (read first):
The origin, the pages and the navigation data are MINE. Never substitute a demo or a site
mentioned anywhere in this package. If the URL, page types or stack are missing from the
profile, ASK me and WAIT.

Any figure in this file describes a MECHANISM or a threshold - not a measurement of my site.

## SCOPE AND HARD BOUNDARY (read first - IMPORTANT)

**THE FIELD-SIZING RULE.** Both halves of this stage are sized by how real users move around
the site, and both numbers are already published for any origin with enough traffic. Navigation
types tell you what share of visits arrive by back/forward, by reload, from cache, or as a
prerender. Do not start work here on a hunch: read the share first, then decide whether it is
worth an afternoon. A back/forward share of 1% and a share of 15% are different projects.

**THE FREE-MECHANISM RULE.** Back/forward cache is not something you build - it is something you
avoid breaking. The browser stores the whole page, including the JavaScript heap, and restores
it instantly. So the finding here is never "implement bfcache". It is: is it working, what is
blocking it, and what does my analytics do when a restored page never fires a page view.

**THE SPA BOUNDARY.** Speculation Rules act on DOCUMENT navigations. On a site where moving
between pages is client-side routing inside a single-page application, they do not apply -
the framework's own prefetch is the lever. A project can be both: a framework app for one
section and separate document navigations for the rest. Establish which before recommending
anything.

**THE SERVER-COST RULE.** Prefetch and prerender move work earlier, they do not remove it. Every
speculated page is a real request to my origin for a page the user may never open. A default
that speculates every visible link turns one visit into dozens of requests. This stage's output
is therefore a BUDGET, not a switch.

## 140.0 Read what earlier stages already found (Do)
- From site-profile.md > ## Cache & CDN (Stage 40): the caching headers. `Cache-Control:
  no-store` used to disqualify a page from bfcache; current Chrome versions have relaxed that,
  so check the browser's own verdict (140.2) rather than reasoning from the header.
- From site-profile.md > ## Consent / CMP (Stage 110): consent scripts are a frequent source of
  both bfcache blockers and duplicated analytics on restore.
- From site-profile.md > ## Measurement: the RUM tool, if any. Its session data answers which
  links users actually click, which is what sizes the prefetch policy in 140.4.

## 140.1 Size it from the field (Do)
- Read the NAVIGATION TYPES for my origin from CrUX (any tool that exposes the CrUX dataset
  will do, including the official visualiser under its All Metrics view). Record the share of:
  navigate, navigate cache, back/forward, **back/forward cache**, reload, restore, prerender.
- The comparison that matters is BACK/FORWARD against BACK/FORWARD CACHE. The first says how
  many visits use the back and forward buttons; the second says how many of those got the
  instant restore. A healthy origin has the second close to the first. A large first and a near
  zero second is the finding.
- Split by device where the tool allows it. Back and forward are disproportionately a mobile
  behaviour - the gesture is the primary way of moving back in a mobile browser - so a desktop
  developer's own habits are a bad proxy for the number.
- **If the origin has no CrUX data** (too little traffic), say so, and fall back to what the RUM
  tool reports about repeated views and navigation. Mark the sizing UNCONFIRMED IN THE FIELD.

## 140.2 Is bfcache working, and what blocks it (Do)
- Through the chrome-devtools MCP server, or by hand: open the page, navigate away, come back,
  and read **DevTools > Application > Back/forward cache**, using its own Test action. It states
  either that the page was served from bfcache or that it was not, with the blockers listed by
  category.
- Record every blocker verbatim. The ones that recur:
  - an `unload` handler anywhere on the page, mine or a vendor's - this is the classic, and it
    is frequently inside a third-party script rather than my code;
  - headers or cookie flags reported by the browser as disqualifying;
  - anything holding an open connection the browser refuses to freeze.
- Check the blockers against the script inventory from Stage 80. If the offender is a vendor
  tag, the fix is a conversation with whoever owns that tag, not a code change.
- Run the test on each page type, not only the homepage. Blockers are frequently per-template.

## 140.3 What happens to analytics on restore (Do - do not skip this)
A restored page does not load. The heap comes back as it was, no scripts re-run, and a page view
that fires once per load will not fire at all.

- Establish what my analytics does today on a back/forward restore. Test it: navigate away,
  come back, and watch whether an event is sent.
- The standard fix is a `pageshow` listener that checks the persisted flag and sends the view
  when the page came from the cache. Record whether it exists.
- **Then check the opposite failure**: a consent or tag script that re-initialises on restore
  and double-counts. Both directions are real and they are found the same way - by watching the
  network on a restore.
- This matters beyond reporting: if fixing bfcache silently reduces recorded traffic, somebody
  will read it as a drop in visits and reverse a correct change.

## 140.4 The prefetch policy - what my own framework already does (Do)
- Establish what the project prefetches by DEFAULT. In a framework with a link component this
  is frequently every link visible in the viewport, including everything inside a drop-down
  menu the moment it opens. Prove it: open the page, open the navigation, and count the
  requests that fire before any click.
- Cost the default: how many requests per visit, against how many pages a visitor actually
  opens. Where the prefetched payload is a server-rendered component response rather than a
  static file, every one of them is real origin work.
- Decide a policy with me, and size it from the field:
  - prefetch on HOVER rather than on visibility, where the framework allows it;
  - or prefetch only the handful of routes that analytics, heatmaps or RUM show people actually
    take next;
  - the honest default for a small site with cheap pages is to leave it alone - say so if that
    is the answer.
- Record the decision and the expected request count after the change.

## 140.5 Speculation Rules - the native mechanism (Do)
Only for document navigations (see the SPA boundary above).

- Establish whether rules are already present. Some platforms ship them by default - WordPress
  among them - which means the site may already be speculating on a policy nobody chose.
- If they are, read them and say whether the policy is defensible. A default that prefetches
  broadly is usually harmless; a default that PRERENDERS broadly is not, because a prerender
  loads and renders the whole target page, scripts included.
- Understand the difference before writing a finding: **prefetch** fetches the document;
  **prerender** loads and renders the page in the background, so the click shows a page that is
  already painted. The second is dramatically better for the user and dramatically more
  expensive for the origin.
- The eagerness setting is the budget: immediate and eager speculate aggressively, moderate
  waits for a hover of roughly 200 ms, conservative waits for the pointer to go down. **Moderate
  on prerender for key routes** is the setting that buys the most for the least - the user has
  already signalled intent, and the work is spent on a page they are probably about to open.
- Analytics and consent apply here too: a prerendered page runs its scripts. Check that a page
  which is prerendered and never visited does not count as a visit, and that nothing is written
  before consent. Route anything found back to Stage 110.
- Where the project is a framework app, say plainly which of the two mechanisms owns which part
  of the site rather than recommending both everywhere.

## 140.6 The verdict (Do - STOP and show me)
- **field sizing**: navigation type shares, back/forward against back/forward cache, by device
  where available;
- **bfcache**: served or not, per page type, with the blockers named and attributed to my code
  or a vendor;
- **analytics on restore**: undercounting, double counting, or correct - and how it was tested;
- **prefetch today**: what fires before any click, and what it costs the origin;
- **policy proposed**: prefetch scope, speculation rules with their eagerness, and why that
  budget and not a bigger one;
- **how this compares** to what is already open from earlier stages. On a site where back and
  forward is 1% of visits, this stage can legitimately end in NO ACTION.

Show me this block and wait.

## 140.7 The experiment (Do - only on a FIX NOW)
- bfcache: remove or route the blocker, re-run the Application test, and confirm the restore is
  instant on a throttled connection. Then re-read the navigation types in the field after the
  change has been live long enough to collect data - this is a FIELD confirmation, like Stage
  120, and until it lands the result is SUPPORTED IN LAB.
- prefetch or speculation: count requests before and after on a realistic visit, and measure
  the time from click to painted target page on the routes you speculate. State the emulation.
- Watch the origin. If the policy widened rather than narrowed, check the effect on server
  response times under normal traffic before calling it a win.

## 140.8 Known traps
- **Reasoning about bfcache from headers instead of asking the browser.** The rules have
  changed; the Application panel is authoritative and takes seconds.
- **Testing only the homepage.** Blockers are usually per-template.
- **Fixing bfcache and silently losing analytics.** A restore fires no page view. Add the
  `pageshow` path in the same change, or the correct fix will look like a traffic drop.
- **Assuming a developer's own habits represent users.** Back and forward is mostly mobile
  behaviour; the field data is not optional here.
- **Treating prefetch as free.** Every speculated page is a real request to my origin for a page
  nobody may open.
- **Prerendering broadly because the platform ships it that way.** A default is not a decision.
- **Recommending Speculation Rules for client-side routing.** They act on document navigations.
- **Forgetting that a prerendered page runs its scripts.** Analytics and consent both apply.
- **Confusing prefetch with prerender in the finding.** One fetches a document, the other
  renders a page. The costs are not comparable.

## Save results
- site-profile.md > ## Navigation: navigation type shares, the bfcache verdict per page type
  with blockers, the analytics-on-restore result, the prefetch policy and the speculation policy
  with its eagerness, plus the date set for field confirmation.
- findings.md: dated entry with the numbers and the reasoning.

## STOP line (end every run with this, verbatim intent)
"Navigation triage done. Field: back/forward <x>%, back/forward cache <y>%, reload <z>%
(<source>). bfcache: <served / not served> on <page types>, blockers: <list, attributed>.
Analytics on restore: <correct / undercounts / double counts>. Prefetch today: <what fires
before a click, count>. Policy: <prefetch scope> + <speculation rules and eagerness / none - SPA
routing / none - NO ACTION>. Field confirmation due <date>. Next: <the single next action>."

## At the end
Check off in site-profile.md > ## Progress: [x] Stage 140 - navigation and bfcache. Report the
field sizing, the bfcache verdict, the analytics result and the policy - nothing more.
