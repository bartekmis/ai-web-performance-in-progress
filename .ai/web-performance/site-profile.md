# Site profile

This file holds the audit DATA and STATE. The AI fills it in while running stages
(asking you, writing here). You may also fill it in manually.

Note: the fields below are filled ONLY from the user's answers (or from "Do" step
outputs). The AI must not auto-derive them from code, MCP or the network.

## Project
- Public URL: <TODO>
- Stack: <TODO e.g. Next.js / WordPress / Astro>
- Code local: <TODO yes/no, path>
- Hosting / server: <TODO>
- Deploy: <TODO e.g. CI/CD>

## Page types
| Type | Representative URL |
|------|--------------------|
| <TODO> | <TODO> |

## Measurement
- WebPageTest baseline: <TODO status>
- CrUX (public field): <TODO status / no data>
- RUM: <TODO which / none>
- RUM access / MCP: <TODO>
- RUM data (p75 per URL): <TODO status>

## Network
Filled by Stage 20 (audit/20-network-dns-tls-http.md). Leave <TODO> until run.

- Host / Apex: <TODO>
- DNS provider / nameservers: <TODO>
- DNS anycast: <TODO yes / probably not>
- DNS lookup cold / warm: <TODO ms / ms>
- A record TTL / IPv6 (AAAA): <TODO>
- TLS version / cipher: <TODO e.g. TLS 1.3>
- HTTP protocol: <TODO 1.1 / h2 / h3>
- HSTS: <TODO present max-age / absent>
- HTTP->HTTPS redirect: <TODO>
- SSL Labs grade / resumption: <TODO>

### Change list
- <TODO filled by Stage 20.7: what to change in the project>

### Preconnect
Filled by Stage 30 (audit/30-preconnect.md). Leave <TODO> until run.
- External origins used (count): <TODO>
- Preconnect (max 4, earliest-critical): <TODO>
- Dns-prefetch / dropped: <TODO>
- Dead hints to remove: <TODO>

## Cache & CDN
Filled by Stage 40 (audit/40-cache-cdn.md). Leave <TODO> until run.
- Edge / CDN in front: <TODO which / none>
- HTML cacheable per page type: <TODO cached (HIT/Age) / dynamic (MISS/no-store)>
- Cache-bust: cached TTFB vs ORIGIN TTFB: <TODO n / n + gap>
- Static assets long+immutable & edge-cached: <TODO yes / origin-bound assets>

## Backend
Filled by Stage 50 (audit/50-backend-triage.md). Leave <TODO> until run.
- Origin TTFB (cache-bypassed, cold / warm): <TODO n / n>
- Static-floor TTFB vs app-route TTFB: <TODO n / n>
- Server-Timing present: <TODO yes (spans) / no - add it>
- Bottleneck class (A compute/DB, B cold-start, C N+1/scale, D infra, E false-alarm): <TODO>
- Next step (tool to open): <TODO routed by stack>

## Progress
- [ ] Stage 10 - profile and baseline
- [ ] Stage 20 - network: DNS / TLS / HTTP
- [ ] Stage 30 - preconnect (external domains)
- [ ] Stage 40 - cache & CDN (edge)
- [ ] Stage 50 - backend triage (TTFB)
