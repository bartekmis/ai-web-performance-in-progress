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
- `wpt/mobile`, `wpt/desktop` - drop your WebPageTest JSON files here

More stages (LCP, CLS, INP, images, fonts...) are added as we go, one per workshop.

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
