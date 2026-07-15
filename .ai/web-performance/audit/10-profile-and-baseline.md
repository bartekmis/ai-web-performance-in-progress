# Stage 10: Profile and baseline

> Can be run STANDALONE or from the index (.ai/web-performance/audit/00-index.md).
> Behaviour (IMPORTANT): question-driven audit. Every "Ask for" = ask me the question,
> STOP, and wait for my answer. Do NOT derive this data yourself (from code, MCP, the
> browser) EVEN IF you can - you must ASK me and wait for me to paste it.
> Do not pre-fill the profile with your own research. "Do" items you perform yourself.
> Ask one at a time; write only after my answer.
> Input: none (this is the start of the audit).
> Output: site-profile.md (## Project, ## Page types, ## Measurement) + wpt/ + findings.md.
> Idempotent: if a section already exists, update it, do not create a second one.

START: ask me question 10.1 and wait for the answer. Do not scan the code to fill
fields for me - you must ask me.

## 10.1 Address and stack
- Ask for: (1) public site URL, (2) stack, (3) whether the source code is local (and path).
- Save to: .ai/web-performance/site-profile.md > ## Project.
- Done when: URL + stack are saved.

## 10.2 Page types
- Goal: we audit the template, not a single page.
- Ask for: what page types exist (e.g. home, article, listing) and one representative URL per type.
- Save to: .ai/web-performance/site-profile.md > ## Page types (table).
- Done when: at least 1 type with a URL.

## 10.3 WebPageTest baseline
- Ask for: whether WPT tests are already done. If not, ask me to run mobile + desktop for each type and upload the JSON files to .ai/web-performance/wpt/mobile/ and .ai/web-performance/wpt/desktop/.
- Do: once the JSONs are in place, extract per type: Start Render, First Contentful Paint (FCP), LCP, CLS, INP, TTFB. Include Start Render and FCP where available.
- Save to: .ai/web-performance/wpt/mobile/, .ai/web-performance/wpt/desktop/ (raw) + .ai/web-performance/findings.md (summary table).
- Done when: each type has a mobile + desktop baseline.

## 10.4 RUM access
- Ask for: whether RUM exists (which one), and whether you have access / MCP to it.
- Save to: .ai/web-performance/site-profile.md > ## Measurement.
- Done when: RUM status established (yes / none / TODO).

## 10.5 CrUX / public field data
- Goal: free field data (p75 from real Chrome users) for the URLs from 10.2.
- Ask for: a Google API key (Chrome UX Report API or PageSpeed Insights) if not configured yet. No key = suggest CrUX Vis (cruxvis.withgoogle.com) manually.
- Do: for each URL from 10.2 (and origin as a fallback) query CrUX:
  - CrUX API: POST https://chromeuxreport.googleapis.com/v1/records:queryRecord?key=KEY
    body {"url":"<URL>","formFactor":"PHONE"} and "DESKTOP"; fallback {"origin":"<origin>"}.
  - or PageSpeed Insights API (loadingExperience field) in one call.
  - Metrics: FCP, LCP, INP, CLS, TTFB (p75). CrUX has no Start Render.
- Note: fresh / low-traffic URLs often have no data (404 / "CrUX data not found"). Try the origin level; if still none, record "no CrUX (insufficient traffic)" and move on.
- Save to: .ai/web-performance/findings.md (p75 per URL, mobile + desktop) + .ai/web-performance/site-profile.md > ## Measurement (CrUX status).
- Done when: each URL has a CrUX result or a "no data" note.

## 10.6 RUM - pull data per URL
- Goal: field data from your own RUM for the page types / URLs from 10.2.
- Input: the RUM source established in 10.4 (e.g. DebugBear RUM MCP or API). No RUM - skip with a note.
- Do: for each page type / URL from 10.2 pull p75 from RUM: FCP, LCP, INP, CLS, TTFB (mobile + desktop if available; include FCP where available) plus volume (session count) so we know whether the sample is enough.
- Note: fresh project with no traffic = few / zero sessions; record "insufficient RUM data" and move on.
- Save to: .ai/web-performance/findings.md (p75 per URL + volume) + .ai/web-performance/site-profile.md > ## Measurement (RUM data status).
- Done when: each URL has RUM data or an "insufficient data" note.

## At the end
Check off in .ai/web-performance/site-profile.md > ## Progress: [x] Stage 10.
