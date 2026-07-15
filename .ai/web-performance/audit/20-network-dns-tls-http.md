# Stage 20: Network - DNS, TLS/SSL, HTTP protocol (before the first byte)

> Can be run STANDALONE or from the index (.ai/web-performance/audit/00-index.md).
> Behaviour: this is a DO stage, not an interview. Unlike Stage 10, you run the
> tools yourself and write the results - you do NOT ask me for the data.
> The ONE thing you must NOT invent is the domain: read it from site-profile.md.
> If it is missing/ambiguous, ASK me and WAIT - do not guess a hostname.
> Input:  site-profile.md > ## Project (Public URL) + ## Page types (hostnames).
> Output: site-profile.md > ## Network + findings.md (results + recommendations).
> Idempotent: if ## Network already exists, update it, do not create a second one.

SCOPE: this stage owns the network layer BEFORE the first byte only -
DNS resolution, TLS/SSL negotiation, HTTP protocol version, HSTS, redirects.
DNS/TLS/HTTP/SSL-Labs facts are HOST properties, NOT per-page. Run every check
ONCE PER HOSTNAME. A huge TTFB is a SERVER problem: record it, hand it to a later
stage (loading/backend), do NOT try to fix it here.

## 20.0 Derive the targets (read the profile - do not invent)
- Read site-profile.md > ## Project (Public URL) and ## Page types.
- Derive HOST = the URL's hostname; APEX = the registrable domain (strip any
  subdomain, e.g. www.example.com -> example.com; example.com stays example.com).
- Collect the DISTINCT set of hostnames across all page-type URLs. If they all
  share one host (the usual case), the host checks below run exactly ONCE.
  If a page type lives on a different hostname, repeat 20.1-20.5 for that host too.
- If ## Project has no Public URL, ASK me for it and WAIT. Never assume a domain.
- For this project: HOST = wdi-wow.online, APEX = wdi-wow.online, all 3 page types
  share it -> run the host checks once. (Confirm against the profile at run time.)

SPEED RULES (apply to everything below):
- Cap EVERY curl with `--max-time 8`. Never GET-loop a slow origin to time DNS.
- Run the independent checks (DNS, TLS, HTTP, timing) in PARALLEL, not serially.
- Kick off the SSL Labs scan (20.5) in the BACKGROUND at the very start of the
  stage and collect its result at the end, so its 1-2 min does not block you.

## 20.1 DNS - provider, anycast, TTL, IPv6, latency (Do)
- `dig NS {APEX} +short` -> classify the provider and decide anycast yes/no:
  - Anycast / global: Cloudflare, AWS Route 53, Google Cloud DNS, NS1, DNSMadeEasy,
    Akamai, Azure DNS -> anycast = yes.
  - Registrar / regional (tld.pl, nazwa.pl, home.pl, ovh, dns.hostinger, most
    shared-hosting NS) -> anycast = probably NO.
- `dig A {HOST} +short`, `dig AAAA {HOST} +short`, `dig {HOST} CNAME +short` ->
  record A/AAAA (IPv6 present?), any CNAME (CDN alias, e.g. *.cloudflare / *.vercel /
  *.cloudfront), and the record TTL (`dig {HOST} +noall +answer`).
- DNS latency (do NOT use a curl GET - it waits for the whole page):
  `dig {HOST} +noall +stats` and read the `;; Query time: N msec` line.
  Run it TWICE: first = cold, second = warm (resolver-cached).
- Verdict: NOT anycast, OR cold Query time > 100 ms -> recommend moving DNS to a
  global-anycast provider (Cloudflare DNS is free) and note it in the change list.

## 20.2 TLS - highest protocol version + cipher (Do)
- Probe the handshake: try `-tls1_3`, and if it does NOT negotiate, fall back to
  `-tls1_2`:
  `echo | openssl s_client -connect {HOST}:443 -servername {HOST} -tls1_3 2>/dev/null | grep -E 'Protocol|Cipher'`
- Record the HIGHEST version that negotiates and the negotiated cipher
  (the `Protocol :` and `Cipher :` lines in the s_client output).
- Verdict: TLS 1.2 is the ceiling (1.3 fails) -> recommend enabling TLS 1.3
  (saves ~1 RTT on the handshake). TLS 1.3 present -> good, note it.
- Note: openssl can tell you the negotiated version/cipher; it CANNOT reliably
  judge session resumption - SSL Labs (20.5) is authoritative for that.

## 20.3 HTTP protocol - h2 / h3 / 1.1 (Do)
- `curl --max-time 8 -sI --http2 https://{HOST}/` -> read the status line for
  `HTTP/2`.
- `curl --max-time 8 -sI --http3 https://{HOST}/` (if the curl build supports it)
  -> `HTTP/3`. Also inspect the `alt-svc` response header (`curl --max-time 8 -sI
  https://{HOST}/ | grep -i alt-svc`): `h3=":443"` advertises HTTP/3.
- Record the best protocol offered (h3 > h2 > 1.1).
- Verdict: HTTP/1.1 only -> recommend enabling HTTP/2. h2 but no h3 AND a CDN is in
  front -> recommend enabling HTTP/3 (QUIC). No CDN and h2 -> HTTP/3 is a CDN/edge
  feature, note as info not a blocker.

## 20.4 HSTS, HTTP->HTTPS redirect, full connection timing (Do)
- HSTS: `curl --max-time 8 -sI https://{HOST}/ | grep -i strict-transport-security`
  -> record whether `Strict-Transport-Security` is present + its max-age /
  includeSubDomains / preload. Missing on an HTTPS-only site -> recommend adding it.
- Redirect: `curl --max-time 8 -sI http://{HOST}/` -> confirm a 301/308 to https://.
  No forced upgrade -> flag it.
- Full timing (one call, all phases):
  `curl --max-time 8 -so /dev/null -w 'dns=%{time_namelookup} connect=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total}\n' https://{HOST}/`
  -> record DNS / TCP connect / TLS handshake / TTFB / total.
- IMPORTANT: this stage owns DNS/TCP/TLS only. If TTFB is huge (e.g. this project's
  ~10s+ from Stage 10.3), that is a SERVER problem - record the number, note
  "handed to backend/loading stage", and do NOT attempt a fix here.

## 20.5 SSL Labs - authoritative grade + resumption (Do, run ONCE per host)
- Run ONCE per host, not per URL. Prefer the API, CACHE-FIRST:
  1. Cached (instant, READY in <1s if a recent scan exists):
     `curl --max-time 8 -s "https://api.ssllabs.com/api/v3/analyze?host={HOST}&fromCache=on&maxAge=24"`
  2. If status != READY: kick a fresh scan in the BACKGROUND at the START of the
     stage, then poll at the end (1-2 min):
     start: `...&startNew=on&all=done`  then poll: `...&all=done` until READY.
- From the READY JSON take: `endpoints[].grade`, `protocols`, `alpnProtocols`,
  `hstsPolicy`, `sessionResumption`, `supportsRc4`, `forwardSecrecy`.
- No API access? If you have a browser MCP: open
  `https://www.ssllabs.com/analyze.html?viaform=on&d={HOST}&hideResults=on`,
  wait ~2 min, read the grade + the warning bullets.
- No API AND no browser: ASK me to open that URL and paste the grade + bullets,
  then WAIT.
- SSL Labs is authoritative for the TLS version confirmation, HSTS verdict and
  session resumption (curl cannot judge resumption). Record those from here.

## Save results
- Save to site-profile.md > ## Network (create the section if absent):
  - HOST, APEX, hostnames covered
  - DNS: provider, anycast (yes/no), A/AAAA, CNAME/CDN, TTL, cold/warm Query time
  - TLS: highest version negotiated + cipher
  - HTTP: best protocol (h3/h2/1.1), alt-svc
  - HSTS verdict, http->https redirect, timing (dns/connect/tls/ttfb/total)
  - SSL_LABS_GRADE, confirmed TLS version, HSTS verdict, session-resumption
- Append to findings.md a dated entry (same format as Stage 10): source, the raw
  numbers, observation, and the per-item recommendations.

## Change list (end every run with this)
Write a numbered, PRIORITISED TODO of exactly what I must change in my project,
ordered bad -> info (worst/highest-impact first, informational last). Cover each
axis that came back sub-optimal:
1. DNS provider (move to global anycast, e.g. Cloudflare DNS - free) - if not anycast / cold >100ms
2. TLS 1.3 (enable it) - if TLS 1.2 is the ceiling
3. HTTP/2 (enable) or HTTP/3 (enable at the CDN/edge) - per 20.3 verdict
4. HSTS (add / fix max-age / preload) - if missing or weak
5. Session resumption (enable) - if SSL Labs reports it off
Each line: the change + the one-line why (RTT saved / grade lifted / crawler win).
The GOAL: I walk away knowing precisely which DNS provider / TLS version / HTTP
protocol to switch to.

## At the end
Check off in site-profile.md > ## Progress: [x] Stage 20 - network DNS/TLS/HTTP.
