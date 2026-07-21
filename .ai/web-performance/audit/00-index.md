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

## Stage order
- 10 - profile and baseline       -> .ai/web-performance/audit/10-profile-and-baseline.md
- 20 - network: DNS / TLS / HTTP   -> .ai/web-performance/audit/20-network-dns-tls-http.md
- 30 - preconnect (external domains) -> .ai/web-performance/audit/30-preconnect.md
- 40 - cache & CDN (edge)          -> .ai/web-performance/audit/40-cache-cdn.md
- 50 - backend triage (TTFB)       -> .ai/web-performance/audit/50-backend-triage.md
- (more stages added per workshop: LCP, CLS, INP...)

## How to resume
Check the "## Progress" section in .ai/web-performance/site-profile.md and run the
first unchecked stage: open its file and execute per the rules above.
