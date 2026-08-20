# AI Web Performance Audit - work in progress

This is the **live, evolving** version of the AI-driven web performance audit we build
during the *Zoptymalizowany Frontend AI Labs* workshops. It tracks the real state of
the audit package as it grows session by session, so you can always see the latest
stages, prompts and structure - not a frozen snapshot.

## How this relates to the starter kit

There are two repos:

- **Starter kit** (what you begin with): https://github.com/bartekmis/ai-web-performance
  A clean, self-contained package you copy into your own project's `.ai/` folder and
  run against your site.
- **This repo** (`ai-web-performance-in-progress`): the same package, but kept in sync
  with what we actually do live. New stages, refined prompts and fixes land here first.

Use this repo as your **reference**: whenever your local copy starts to drift from what
you see in the workshop, compare it against this one and pull in the newer stages.

## What's inside (`.ai/web-performance/`)

- `site-profile.md` - data + audit state (the AI fills it in, asking you)
- `findings.md` - log of results, measurements and hypotheses
- `PROMPTS.md` - ready-to-paste prompts: full audit / resume / single stage / new stage
- `audit/00-index.md` - orchestrator (stage order, execution rules)
- `audit/10-profile-and-baseline.md` - stage 1: profile, page types, WPT baseline, CrUX, RUM
- `audit/20-network-dns-tls-http.md` - stage 2: network - DNS / TLS / HTTP
- `audit/30-preconnect.md` - stage 3: preconnect for external domains
- `audit/60-render-start.md` - stage 6: what delays the first paint, proven by a
  controlled A/B experiment (the first stage that tests a hypothesis instead of reading
  a value - see the evidence rule in `audit/00-index.md`)
- `audit/70-dom-size.md` - stage 7: what the size and shape of the DOM cost in style and
  layout recalculation, which script forces it, and - explicitly - whether it is worth
  fixing yet. Node count is not the criterion; a 40 ms+ recalculation is.
- `audit/80-scripts-and-third-party.md` - stage 8: when each script executes and what it
  blocks while it does. Weight is not the criterion; measured main-thread time is. Covers
  loading modes, third-party SPOF, the tag-manager cascade, and why a coverage report tells
  you what to DEFER, not what to delete.
- `audit/90-images-and-video.md` - stage 9: images and video, treated as **two separate
  populations**. The LCP element is a latency problem (how late the browser discovers it,
  what priority it gets); every other image is a bandwidth problem (dimension, format,
  deferral) - and the treatment that helps one hurts the other. Levers are applied in a
  fixed order: dimension, format, compression, loading strategy. Covers `srcset`/`sizes`,
  `img` vs `background-image`, SVG and base64, and video including third-party embeds.
- `audit/100-fonts.md` - stage 10: fonts, treated as the END OF A CHAIN rather than as
  files. Nothing about a font request is decided by the font - it is decided by how many
  hops (HTML, CSS, an `@import`, CSSOM, a matched selector) happened before the browser knew
  the file existed. The stage produces a **preload budget**: a number, usually 0 or 1,
  because preload is zero-sum and preloading every face routinely measures worse than
  preloading none. It also forces the trade-off nobody can optimise away - hide the text
  (`block`) or shift it (`swap`) - and makes the project choose one on purpose, with metric
  matching as the price of `swap`. Bytes (weights, subsetting, `unicode-range`, WOFF2,
  variable fonts) come last, and are reported as transfer saved, not as a metric win - the
  stage ends with a real regenerated file, not a recommendation. The inventory is delegated
  to the **WebPerf Snippets** skill (nucliweb) where it is installed, with an explicit note
  on where this stage overrules its preload advice.
- `audit/120-inp-interactions.md` - stage 12: INP, and the first stage whose TARGET cannot be
  chosen in the lab. A page offers dozens of interactions and real users only ever hit a handful,
  so the element under audit comes from **RUM** and is only then reproduced in the browser -
  driving an agent to click everything costs a fortune and still measures one device, yours. The
  diagnosis is the **subpart**, not the total, and it decides who owns the problem: input delay is
  a loading problem (Stages 60/80), presentation delay is a recalculation problem (Stage 70), and
  only processing time belongs here. The fix then REORDERS work rather than removing it, because
  INP ends at the next paint and not at the end of your JavaScript - the visual change lands
  first and the rest runs after the frame, via `setTimeout` or `scheduler.yield` (not the same
  mechanism, and the second one needs a fallback). The only fix in the stage that genuinely
  reduces work is deleting work nobody needed. Ends SUPPORTED IN LAB with a date to re-read the
  field, because a lab click is a reproduction, not the metric.
- `audit/130-js-runtime.md` - stage 13: what still runs when the page ALREADY LOOKS READY. Every
  stage before it asked what delays something the user is waiting for; this one starts after
  Document Complete, where the main thread is still busy and nobody is waiting - which is exactly
  why nobody measures it. Its central distinction is that **a component that appears on scroll is
  not a component whose code runs on scroll**: a library sitting in a shared vendor chunk executed
  at startup no matter how lazily its section fades in, and the cost is smeared across the load
  rather than sitting in one obvious task. So the stage reads the TRIGGER from source (module
  scope, a `DOMContentLoaded` handler, an observer, a framework directive, hydration, or something
  repeating) and confirms it in the trace. It counts the bill in four parts - bytes, parse and
  compile, execution, memory - and orders the fixes: delete, then defer the CODE, then defer the
  work, then defer the RENDERING (`content-visibility` lives in that last group, because it skips
  layout and paint, not JavaScript). Covers hydration cost, re-renders and what a memoising
  compiler does and does not buy.
- `audit/140-navigation-bfcache.md` - stage 14: the first stage that is not about one page load
  but about the VISIT. Both halves are sized by published field data (navigation types), and one
  of them is free: **back/forward cache is not built, it is merely not broken**, so the finding is
  which blocker breaks it and whether it is yours or a vendor's. It also owns a reporting failure
  that looks like a performance win - a restored page fires no page view, so fixing bfcache
  without a `pageshow` path reads as a drop in traffic and gets reverted. The second half is a
  BUDGET rather than a switch: framework link prefetching and Speculation Rules move origin work
  earlier without removing it, and a default that speculates every visible link turns one visit
  into dozens of requests.
- `wpt/mobile`, `wpt/desktop` - drop your WebPageTest JSON files here

More stages (CLS...) are added as we go, one per workshop.

## How to use it

The audit is **question-driven**: the AI interviews you and does not derive data on your
behalf. Each stage is self-contained - it reads what it can from `site-profile.md`, asks
you for anything missing (one question at a time), and writes results to `findings.md`.

Three ways to run (full prompts in `.ai/web-performance/PROMPTS.md`):

1. **Full audit** at once, from `audit/00-index.md`.
2. **Resume** from the first unchecked stage (per the `## Progress` section in
   `site-profile.md`).
3. **Single stage** standalone (e.g. just profile and baseline).

Start your AI assistant (Claude Code / Codex / Antigravity...) in the folder where
`.ai/web-performance/` lives, then paste a prompt from `PROMPTS.md`.

## Keeping up to date

Pull this repo before/after each session to get the newest stages and prompts, then
copy the updated `.ai/web-performance/` into your own project.

```
git pull
```
