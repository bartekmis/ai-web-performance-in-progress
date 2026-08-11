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
- `wpt/mobile`, `wpt/desktop` - drop your WebPageTest JSON files here

More stages (CLS, INP...) are added as we go, one per workshop.

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
