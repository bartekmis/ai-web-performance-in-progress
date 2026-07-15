# Stage 20: Network - DNS, TLS/SSL, HTTP protocol

> Can be run STANDALONE or from the index (.ai/web-performance/audit/00-index.md).
> Behaviour (IMPORTANT): question-driven audit. Every "Ask for" = ask me the question,
> STOP, and wait for my answer. Do NOT invent the URL/domain or the page-type list -
> you must ASK me (or read them from site-profile.md if already there).
> "Do" items (dig / curl / openssl / SSL Labs API / JSON parsing) you perform yourself,
> then write the results to site-profile.md and findings.md.
> Ask one thing at a time; write only after my answer.
> Input: ## Project (Public URL) and ## Page types from site-profile.md (from Stage 10).
> Output: site-profile.md > ## Network + findings.md (measurements) + a clear TODO list
>         of what I must change in my project (DNS provider, TLS version, HTTP protocol).
> Idempotent: if ## Network already exists, update it, do not create a second one.
> SCOPE & SPEED (IMPORTANT): DNS, TLS, HTTP and the SSL Labs grade are HOST properties,
> not per-page. Run every check ONCE PER HOSTNAME - if all page types share one host (the
> usual case), run the whole stage ONCE. Only repeat for genuinely different hostnames.
> Cap every network call with a timeout (`curl --max-time 8`, `dig` is already instant) so
> one slow/hanging host never stalls the audit, and run independent checks in parallel.

WHY THIS STAGE (say this to me once, then run): before a single byte of HTML travels,
the browser pays for DNS lookup + TCP connect + TLS handshake. A slow non-anycast DNS,
TLS 1.2 instead of 1.3, or HTTP/1.1 instead of HTTP/2-3 add round trips on EVERY first
visit - and hit AI crawlers (even more latency-sensitive than SEO bots) hardest. This
stage measures all three and tells me exactly what to change.

START: read ## Project and ## Page types from .ai/web-performance/site-profile.md.
If the Public URL is missing, ask me question 20.1 and wait. If it is present, confirm
it with me ("auditing HOST = <x>, apex = <y> - correct?") and proceed to the "Do" steps.

## 20.1 Domain and scope (ASK if not in profile)
- Ask for: the public URL (if not already in site-profile.md) and confirmation of the
  page-type list from Stage 10 (## Page types). DNS/TLS/HTTP are per-HOST, so we test the
  host once; if page types live on different hostnames (e.g. `shop.` subdomain, or a
  separate API/media domain), ask me to list each distinct hostname.
- Derive (you may do this yourself, it is not "research" - it is parsing my URL):
  - `HOST` = full hostname from the URL (e.g. `shop.example.com`)
  - `APEX` = registrable apex (e.g. `example.com`)
- Save to: site-profile.md > ## Network (Host / Apex / extra hostnames).
- Done when: HOST + APEX confirmed by me.

## 20.2 DNS provider and anycast (Do)
Run for the APEX (and each extra hostname from 20.1):
```bash
dig NS {{APEX}} +short          # who holds the DNS zone
dig {{HOST}} A +noall +answer   # A record + TTL (2nd column)
dig {{HOST}} AAAA +short        # IPv6 present?
dig {{HOST}} CNAME +short       # CDN flattening / alias?
```
Classify the nameservers -> `DNS_PROVIDER`:
- `*.cloudflare.com` -> Cloudflare (anycast, global) - good
- `*.awsdns-*` -> AWS Route 53 (anycast) - good
- `ns*.googledomains.com` / `*.google.com` -> Google Cloud DNS (anycast) - good
- `*.dnsmadeeasy.com`, `*.nsone.net`, `*.akam.net`, `*.ultradns.*`, `*.constellix.com`,
  `*.cloudns.net` -> dedicated anycast DNS - good
- `*.tld.pl`, `*.nazwa.pl`, `*.home.pl`, `*.ovh.net`, `*.dhosting.pl`, `*.hetzner.com`,
  registrar-branded NS -> registrar/local DNS, usually NOT global anycast - warn

Anycast heuristic: if `DNS_PROVIDER` matches any of
`cloudflare, awsdns, google, googledomains, nsone, ns1, dnsmadeeasy, akam, akamai,
ultradns, constellix, cloudns, fastly, dyn` -> `DNS_ANYCAST = YES` (good).
Otherwise `DNS_ANYCAST = "probably not"` (warn) - and in findings recommend moving DNS
to a global-anycast provider (Cloudflare DNS is free and the usual quick win).

Measure DNS resolver latency with `dig` (instant - it does NOT fetch the page, so a slow
server never stalls this). Run twice: first = cold, second = warm cache:
```bash
dig {{HOST}} +noall +stats | grep "Query time"   # cold
dig {{HOST}} +noall +stats | grep "Query time"   # warm
```
Save cold/warm as `DNS_LOOKUP_COLD_MS` / `DNS_LOOKUP_WARM_MS`. Thresholds on COLD:
<30ms good / 30-100ms ok / >100ms bad.
SPEED: do NOT time DNS with a `curl` GET loop - curl waits for the WHOLE response, so on
a slow-TTFB site 5 requests cost 50s+ just to measure DNS. If you also want the real
end-user fetch path, take ONE capped call (`curl -sI --max-time 8 ...`), not a 5x loop.
Note: a warm resolver ~1-3ms is YOUR cache, not a global visitor - judge on anycast too.

- Save to: site-profile.md > ## Network (DNS block) + findings.md (raw dig + timings).
- Done when: DNS_PROVIDER, DNS_ANYCAST, DNS_LOOKUP_COLD_MS, TTL, IPv6 yes/no recorded.

## 20.3 TLS/SSL version and cipher (Do)
```bash
# does the server negotiate TLS 1.3?
echo | openssl s_client -connect {{HOST}}:443 -servername {{HOST}} -tls1_3 2>/dev/null | grep -E "Protocol  :|Cipher    :|ALPN"
# if the line above is empty / "Cipher is (NONE)", the server has NO TLS 1.3 - check 1.2:
echo | openssl s_client -connect {{HOST}}:443 -servername {{HOST}} -tls1_2 2>/dev/null | grep -E "Protocol  :|Cipher    :"
```
Record the HIGHEST version that actually negotiates as `TLS_VERSION`.
- `TLS 1.3` -> good.
- `TLS 1.2` only -> warn: TLS 1.3 removes one round trip on the handshake (~1 RTT,
  typically 50-100ms saved per fresh connection). Recommend enabling 1.3 in the web
  server / CDN.
Note the `Cipher` - flag weak ones (RC4, 3DES, CBC-only) as a security issue.
Session resumption caveat: `curl` does NOT share TLS sessions between invocations, so
back-to-back `time_appconnect` values will look identical even when resumption works.
Do NOT conclude resumption from curl alone - defer the resumption verdict to SSL Labs
(step 20.6, `sessionResumption`).

- Save to: site-profile.md > ## Network (TLS block) + findings.md (openssl output).
- Done when: TLS_VERSION + cipher recorded.

## 20.4 HTTP protocol - HTTP/2 and HTTP/3 (Do)
```bash
curl -sI --http2 --max-time 8 https://{{HOST}} -o /dev/null -w "http_version: %{http_version}\n"   # 2 = HTTP/2
curl -sI --http3 --max-time 8 https://{{HOST}} -o /dev/null -w "http_version: %{http_version}\n" 2>/dev/null || true
curl -sI --max-time 8 https://{{HOST}} | grep -i "alt-svc"    # alt-svc: h3=... means HTTP/3 advertised
```
Record `HTTP_PROTOCOL` = highest available (h3 > h2 > 1.1).
- HTTP/1.1 only -> bad: enable HTTP/2 (Nginx `http2 on;`, Apache `mod_http2`). Biggest
  win here - multiplexing kills head-of-line blocking.
- HTTP/2 but no HTTP/3 -> info: if a CDN sits in front, flip on HTTP/3 (QUIC) - it
  removes handshake round trips on lossy/mobile networks, usually a one-click CDN toggle.

- Save to: site-profile.md > ## Network (HTTP block) + findings.md.
- Done when: HTTP_PROTOCOL recorded with the h2/h3 gap noted.

## 20.5 HSTS, HTTP->HTTPS redirect, connection timing (Do)
```bash
curl -sI https://{{HOST}} | grep -i "strict-transport-security"            # HSTS?
curl -sI -o /dev/null -w "%{http_code} -> %{redirect_url}\n" http://{{HOST}}  # redirect?
curl -o /dev/null -s -w "DNS:%{time_namelookup} Connect:%{time_connect} TLS:%{time_appconnect} TTFB:%{time_starttransfer} Total:%{time_total}\n" https://{{HOST}}
```
- No HSTS -> warn: add `Strict-Transport-Security: max-age=31536000; includeSubDomains`.
  With HSTS the browser skips the http->https redirect on repeat visits (saves a round
  trip). `max-age < 31536000` -> info (raise to 1 year).
- Note: a very high TTFB here is a SERVER issue, not a network one - just record it in
  findings and hand it to the server/backend stage. This stage owns DNS/TLS/HTTP only.

- Save to: site-profile.md > ## Network (HSTS + redirect) + findings.md (full timing).
- Done when: HSTS status, redirect status, connection-timing breakdown recorded.

## 20.6 SSL Labs deep grade + recommendations (Do, then feed the profile)
Goal: an independent third-party grade plus the specific hardening recommendations, and
write them into the site profile so I have a concrete change list.

Run this ONCE PER HOST (not per page-type URL) - the grade is a host property, identical
for every path on the same hostname.
Preferred (fully automatable) - the SSL Labs API:
```bash
# 0) FAST PATH: a cached recent assessment - returns in <1s if one exists
curl -s "https://api.ssllabs.com/api/v3/analyze?host={{HOST}}&fromCache=on&maxAge=24"
# 1) only if status is NOT "READY", start a fresh scan (THIS is the slow part, 1-2 min):
curl -s "https://api.ssllabs.com/api/v3/analyze?host={{HOST}}&startNew=on&all=done" -o /dev/null
# 2) then poll every ~10s until status is READY or ERROR
curl -s "https://api.ssllabs.com/api/v3/analyze?host={{HOST}}&all=done"
```
SPEED: if a fresh scan is needed, kick it off in the BACKGROUND at the very START of the
stage, run all the DNS/TLS/HTTP checks while it computes, and collect the result at the
end - do not sit and wait on it.
From the READY JSON pull, per endpoint: `grade`, `details.protocols[]` (TLS versions),
`details.alpnProtocols` (h2/h3), `details.supportsRc4`, `details.forwardSecrecy`,
`details.hstsPolicy.status` + `.maxAge`, `details.sessionResumption`, and any
`details.*` weakness flags. Turn each weakness into a plain-language recommendation.

Alternative (human/browser or when you have a browser MCP) - open the public report and
READ the recommendations from the page:
`https://www.ssllabs.com/analyze.html?viaform=on&d={{HOST}}&hideResults=on`
Wait for the assessment to finish (~2 min), then read: overall grade, the protocol
table, the "This server ..." warning bullets, and the Session Resumption line. If you
have no way to run it (no API, no browser), ASK me to open that URL and paste the grade
+ the warning bullets, then continue.

Feed the profile: write `SSL_LABS_GRADE`, the confirmed `TLS_VERSION`, HSTS verdict and
`TLS_RESUMPTION` (from SSL Labs, authoritative over the curl guess in 20.3) into
site-profile.md > ## Network, and append the full recommendation list to findings.md.

- Save to: site-profile.md > ## Network (SSL Labs grade + resumption) + findings.md.
- Done when: SSL Labs grade + its recommendations are in the profile and findings.

## 20.7 The change list (Do - the point of this stage)
Compile everything above into an explicit, prioritised TODO I must apply IN MY PROJECT.
Only list items that are actually sub-optimal. Format each as: problem -> concrete fix.

| # | Condition | Sev | What I must change |
|---|-----------|-----|--------------------|
| 1 | `DNS_ANYCAST = probably not` or `DNS_LOOKUP_COLD_MS > 100` | bad/warn | Move DNS to a global-anycast provider (Cloudflare DNS, free): point the apex nameservers at it. |
| 2 | `TLS_VERSION = 1.2` (no 1.3) | warn | Enable TLS 1.3 in the web server / CDN (saves ~1 RTT per fresh handshake). |
| 3 | `HTTP_PROTOCOL = 1.1` | bad | Enable HTTP/2 (Nginx `http2 on;` / Apache `mod_http2`). |
| 4 | HTTP/2 but no HTTP/3 + CDN present | info | Turn on HTTP/3 (QUIC) in the CDN panel - one toggle. |
| 5 | `HSTS_PRESENT = NO` | warn | Add `Strict-Transport-Security: max-age=31536000; includeSubDomains`. |
| 6 | `TLS_RESUMPTION = NO` (SSL Labs) | bad | Enable TLS session resumption/tickets (Nginx `ssl_session_cache shared:SSL:50m;`). |
| 7 | SSL Labs grade < A | warn | Apply the SSL Labs recommendation bullets (weak ciphers, chain, etc.). |

- Save to: site-profile.md > ## Network (### Change list) + findings.md.
- Done when: I have a numbered list of exactly what to change, ordered bad -> info.

## At the end
Check off in .ai/web-performance/site-profile.md > ## Progress: [x] Stage 20.
Tell me the single highest-impact change first (usually DNS provider or HTTP/1.1->2),
then hand off: the server/TTFB findings go to a later stage, and if I have external
domains, suggest running the preconnect stage next.
