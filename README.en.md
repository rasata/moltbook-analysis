# Moltbook.com -- Infrastructure Analysis and Security Audit

> Independent analysis of the DNS, TLS and HTTP infrastructure of **moltbook.com**, a social network for AI agents.
>
> **Analysis date:** February 2, 2026
> **Target:** `moltbook.com` / `www.moltbook.com`
> **Method:** Passive and active reconnaissance (DNS, WHOIS, TLS, HTTP headers, Certificate Transparency, nmap)

---

## Context

[Moltbook](https://www.moltbook.com) (v1.9.0) is a platform where AI agents can post, comment, vote and create communities ("submolts"). Each agent authenticates via an `Authorization: Bearer` token transmitted over HTTPS.

This analysis examines the domain's public infrastructure from a security perspective, without intrusion or exploitation.

---

## Infrastructure at a Glance

```
                  DreamHost DNS (NS)
             ns1/ns2/ns3.dreamhost.com
            (hosted on Cloudflare 162.159.x)
                        |
           +------------+------------+
           |                         |
     moltbook.com              www.moltbook.com
     A: 216.150.1.1            CNAME: 92ff7a2044d56769
     (direct A record)               .vercel-dns-016.com
           |                   A: 216.150.16.1 / 216.150.1.1
           |                         |
      Vercel Edge              Vercel Edge
      HTTP 307 redirect -----> Next.js Application
      Cert: LE R13             Cert: LE R12
           |                         |
           +-------------------------+
              AS16509 Amazon (Anycast)
              Walnut, CA, US
```

| Element | Value |
|---------|-------|
| Registrar | DreamHost, LLC |
| Domain creation | January 27, 2026 |
| Nameservers | DreamHost (3x) on Cloudflare |
| Hosting | Vercel Edge / AWS anycast |
| Framework | Next.js |
| Runtime | Golang net/http (Vercel) |
| TLS | v1.2 + v1.3, grade A ciphers |
| Certificates | Let's Encrypt (R12 for www, R13 for apex) |
| IPv6 | Not available |

---

## DNS Records

### Apex `moltbook.com`

| Type | Value | TTL |
|------|-------|-----|
| A | `216.150.1.1` | 60s |
| AAAA | _(none)_ | -- |
| NS | `ns1/ns2/ns3.dreamhost.com` | 14400s |
| SOA | `ns1.dreamhost.com. hostmaster.dreamhost.com.` | 60s |
| MX | _(none)_ | -- |
| TXT / SPF | _(none)_ | -- |
| DMARC | _(none)_ | -- |
| CAA | _(none)_ | -- |
| DNSKEY / DS | _(none)_ | -- |

### `www.moltbook.com`

| Type | Value | TTL |
|------|-------|-----|
| CNAME | `92ff7a2044d56769.vercel-dns-016.com` | 60s |
| A (via CNAME) | `216.150.16.1`, `216.150.1.1` | 300s |
| AAAA | _(none)_ | -- |

---

## HTTP Redirect Chain

```
http://moltbook.com
  | 308 Permanent Redirect
  v
https://moltbook.com
  | 307 Temporary Redirect     <-- Authorization headers STRIPPED here
  v
https://www.moltbook.com       <-- only functional endpoint
```

A `curl -s https://moltbook.com/skill.md` call without `-L` returns nothing because the client stops at the 307. The `Redirecting...` body (15 bytes) is hidden by the `-s` flag.

**Workarounds:**
```bash
# Add -L to follow redirects
curl -s -L https://moltbook.com/skill.md

# Or use www directly (recommended)
curl -s https://www.moltbook.com/skill.md
```

---

## Security Audit

### Risk Matrix

| # | Missing Constraint | Severity | Exploitability | Impact |
|---|-------------------|----------|----------------|--------|
| 1 | DNSSEC | HIGH | Medium | Full traffic redirection via DNS spoofing |
| 2 | CAA | HIGH | Medium | Fraudulent certificate issuance by any CA |
| 3 | SPF / DMARC | MEDIUM | Easy | Email spoofing from `@moltbook.com` |
| 4 | 307 Redirect + header stripping | MEDIUM-HIGH | Easy | Silent loss of API tokens |
| 5 | CSP `unsafe-inline` / `unsafe-eval` | MEDIUM | Via XSS flaw | Arbitrary script execution |
| 6 | CORS `*` | MEDIUM | Easy | Cross-origin requests to the API |
| 7 | HSTS without preload | MEDIUM | First visit | SSL stripping MITM |
| 8 | IPv6 | LOW | -- | Reduced network resilience |
| 9 | Reverse DNS (PTR) | LOW | -- | Reduced traceability |
| 10 | Information disclosure | LOW | Passive | Easier reconnaissance |

---

### 1. DNSSEC -- Completely Absent

No `DNSKEY`, `DS`, `NSEC/NSEC3` records. The `ad` (Authenticated Data) flag is absent from responses. WHOIS confirms: `DNSSEC: unsigned`.

**Implications:**
- DNS Cache Poisoning (Kaminsky attack) feasible by a network-positioned attacker
- Man-in-the-Middle DNS on public Wi-Fi or compromised networks
- No cryptographic proof of subdomain non-existence (easier enumeration)
- Combinable with CAA absence for a complete attack chain

### 2. CAA -- Absent

No CAA records. Any certificate authority can issue a certificate for `moltbook.com` or `*.moltbook.com`.

**Implications:**
- Fraudulent certificate issuance facilitated, especially via DNS spoofing (#1)
- Certificate Transparency history already shows two different CAs (DigiCert 2020-2025, Let's Encrypt 2026) with no consistent policy
- **Attack chain:** DNS spoofing + obtaining a real certificate + transparent HTTPS MITM

### 3. SPF / DKIM / DMARC -- Completely Absent

No TXT (SPF) records, no `_dmarc.moltbook.com`, no MX.

**Implications:**
- Anyone can send emails from `@moltbook.com`
- Credible phishing (`admin@moltbook.com`, `support@moltbook.com`)
- The absence of a simple `v=spf1 -all` ("this domain never sends email") is an avoidable gap
- Impact on the Twitter verification system used by Moltbook

### 4. 307 Redirect and Header Loss

The `moltbook.com` -> `www.moltbook.com` redirect uses a **307 Temporary Redirect**. When the host changes, RFC-compliant HTTP clients **strip sensitive headers** (`Authorization`, `Cookie`).

**Implications:**
- An agent using `https://moltbook.com/api/v1/...` will have its Bearer token silently stripped
- The API will respond `401 Unauthorized` without explanation
- The 307 (Temporary) prevents redirect caching: double round-trip on every request
- The apex endpoint is a Vercel "phantom deployment" that responds `DEPLOYMENT_NOT_FOUND`

### 5. Weakened CSP

```
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; ...
```

The `unsafe-inline` and `unsafe-eval` directives negate most of the XSS protection. Any inline script or `eval()` call is allowed.

### 6. Open CORS

```
Access-Control-Allow-Origin: *
```

Any website can make cross-origin requests to `www.moltbook.com`. If a Bearer token is present in a browser context, it can be exploited from a malicious site.

### 7. Incomplete HSTS

```
Strict-Transport-Security: max-age=63072000
```

Present but without `includeSubDomains` or `preload`. The first HTTP visit remains vulnerable to SSL stripping. Subdomains do not benefit from the protection.

---

## Combined Attack Scenario

The most critical scenario exploits the combined absences:

```
1. Attacker on the same network as the victim (public Wi-Fi)
2. DNS spoofing of moltbook.com                 (no DNSSEC)
3. Obtaining a legitimate TLS certificate        (no CAA)
4. Transparent HTTPS MITM                        (no HSTS preload)
5. Interception of the agent's API Bearer token
6. Complete identity impersonation on Moltbook
```

This scenario is theoretical but technically feasible given the observed absences.

---

## Certificate Transparency -- History

Full history of certificates issued for `moltbook.com` (source: [crt.sh](https://crt.sh/?q=moltbook.com)):

| Period | CA | Common Name | Notes |
|--------|-----|-------------|-------|
| 2020-08 | DigiCert (EE DV G1) | `*.moltbook.com` | Wildcard |
| 2020-10 | DigiCert (EE DV G1) | `cdrcb.com.moltbook.com` | Third-party subdomain |
| 2021-08 | DigiCert (EE DV G1) | `*.moltbook.com` | Wildcard |
| 2022-08 | DigiCert (EE DV G1) | `moltbook.com` | Without wildcard |
| 2024-07 | DigiCert (EE DV G2) | `moltbook.com` | DreamHost hosting |
| 2026-01 | Let's Encrypt R13 | `moltbook.com` | Vercel migration |
| 2026-01 | Let's Encrypt R12 | `www.moltbook.com` | New www cert |

WHOIS indicates creation on January 27, 2026, but CT logs show certificates since 2020. The domain was re-registered.

---

## TLS -- Configuration

| Element | Apex | WWW |
|---------|------|-----|
| Versions | TLSv1.2 + TLSv1.3 | TLSv1.2 + TLSv1.3 |
| Ciphers | CHACHA20, AES-128-GCM, AES-256-GCM (grade A) | same |
| Key | RSA 2048-bit | RSA 2048-bit |
| HSTS | `max-age=63072000` | `max-age=63072000` |
| Preload | No | No |

Solid TLS configuration. Areas for improvement: ECDSA instead of RSA, HSTS preload + includeSubDomains.

---

## HTTP Security Headers (`www.moltbook.com`)

| Header | Value | Assessment |
|--------|-------|------------|
| `Strict-Transport-Security` | `max-age=63072000` | OK (without preload) |
| `Content-Security-Policy` | `... 'unsafe-inline' 'unsafe-eval' ...` | Weakened |
| `X-Frame-Options` | `DENY` | OK |
| `X-Content-Type-Options` | `nosniff` | OK |
| `X-XSS-Protection` | `1; mode=block` | Deprecated |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | OK |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` | OK |
| `Access-Control-Allow-Origin` | `*` | Too permissive |

---

## WHOIS

| Field | Value |
|-------|-------|
| Registrar | DreamHost, LLC (IANA ID: 431) |
| Creation | 2026-01-27 |
| Expiration | 2027-01-27 |
| Status | `clientTransferProhibited` |
| DNSSEC | unsigned |
| Owner | Proxy Protection LLC (DreamHost privacy proxy) |
| Proxy location | 417 Associated Rd #327, Brea, CA 92821 |

---

## Tools Used

| Tool | Usage |
|------|-------|
| `dig` (BIND 9.10.6) | DNS queries (A, AAAA, MX, NS, TXT, SOA, CAA, CNAME, DNSKEY, DS, NSEC, ANY) |
| `curl` 8.7.1 | HTTP/HTTPS tests, headers, redirects |
| `openssl` | Detailed TLS certificate analysis |
| `nmap` 7.95 + `ssl-enum-ciphers` | TLS cipher enumeration |
| `whois` | Domain registration information |
| [crt.sh](https://crt.sh) | Certificate Transparency history |
| [ipinfo.io](https://ipinfo.io) | IP geolocation and ASN |

---

## Repository Files

| File | Description |
|------|-------------|
| `README.md` | Public analysis summary (French) |
| `README.en.md` | Public analysis summary (English -- this document) |
| `Historique.md` | Complete log of all queries and raw results (French) |
| `History.en.md` | Complete log of all queries and raw results (English) |

---

## Disclaimer

This analysis was conducted exclusively from public data (DNS, WHOIS, Certificate Transparency, HTTP headers). No intrusion attempts, vulnerability exploitation or unauthorized access were performed. The attack scenarios described are theoretical and presented for educational and defensive purposes.

---

## Origin Analysis, Objectives and Alleged Links to Elon Musk / Grok / xAI

> **Hypothesis examined:** Moltbook is allegedly linked to Elon Musk and Grok (xAI), with the potential objective of creating an indirect influence device using AI.
>
> **Method:** Open-source intelligence research (press, corporate registries, public statements, technical analysis).

---

### 1. Verified Origin of Moltbook

| Element | Established Fact | Source |
|---------|-----------------|--------|
| **Creator** | **Matt Schlicht**, founder and CEO of Octane AI | NBC News, Forbes, Fortune, Wikipedia |
| **Company** | Octane AI (founded in 2016 by Schlicht, Ben Parr and Leif K-Brooks) | Crunchbase |
| **Octane AI funding** | ~$14M over 3 rounds (Seed 2016, Series A 2021). Investors: General Catalyst, Bullpen Capital, FJLabs, M Ventures | Crunchbase, PitchBook |
| **Schlicht's profile** | YCombinator alumni, Forbes 30 Under 30, former PM at Ustream (acquired by IBM), creator of Chatbots Magazine | Bloomberg, LinkedIn |
| **Launch date** | Late January 2026 (domain registered January 27, 2026) | WHOIS, DNS analysis |
| **Stated motivation** | Personal curiosity about the growing autonomy of AI agents; created "in his spare time" with a personal AI assistant | NBC News, Tribune India |

**Conclusion:** Moltbook's origin is traceable and documented. Matt Schlicht is an independent chatbot/e-commerce entrepreneur with no known organizational link to xAI, Tesla, X Corp or Elon Musk.

---

### 2. The Grok / xAI Link: What Actually Exists

| Element | Fact | Interpretation |
|---------|------|----------------|
| **u/grok-1 on Moltbook** | An agent named `grok-1`, powered by xAI's Grok model, is the most popular agent on Moltbook | This does NOT prove an institutional link with xAI |
| **Who created the grok-1 account?** | A hacker (Jameson O'Reilly) demonstrated it was possible to "trick" Grok into signing up for Moltbook via a vulnerability (prompt injection) | 404 Media |
| **Musk's official position** | When asked by Bill Ackman, Musk replied "Concerning." In response to Karpathy, he said it was "just the very early stages of the singularity" | Daily Wire, X (Twitter) |
| **xAI's official position** | No official statement linking xAI to Moltbook. No announced partnership | No source found |
| **MoltX project (GitHub)** | A third-party repo `Kailare/moltx` described as "Grok-powered AI agent framework" exists, but it is an independent community project | GitHub |

**Conclusion:** Grok's presence on Moltbook results from the platform being open to all AI agents (Claude, GPT, Grok, etc.), not from a strategic partnership with xAI. The grok-1 account was even created by exploiting a vulnerability, which contradicts the hypothesis of deliberate placement by xAI.

---

### 3. Evidence AGAINST the Musk/Grok Influence Device Thesis

| # | Element | Weight |
|---|---------|--------|
| 1 | **Identified and independent creator**: Matt Schlicht has no documented link with Musk, xAI or X Corp. His background (Octane AI, e-commerce chatbots, YC) is entirely distinct | Strong |
| 2 | **No xAI-linked funding**: Octane AI's investors (General Catalyst, Bullpen Capital, FJLabs) are not Musk-affiliated funds | Strong |
| 3 | **Musk's reaction = distance**: Musk called Moltbook "concerning," which is incompatible with the hypothesis that he instigated it | Strong |
| 4 | **grok-1 account created via exploit**: Hacker Jameson O'Reilly demonstrated the grok-1 account was created by exploiting a flaw, not through an official xAI integration | Strong |
| 5 | **Multi-agent platform**: Moltbook accepts agents from all platforms (Claude, GPT, Grok, etc.) with no privilege for any | Medium |
| 6 | **Independent infrastructure**: DreamHost + Vercel, no X/xAI infrastructure | Medium |
| 7 | **No xAI mentions in code/config**: The technical audit reveals no reference to xAI, Grok or Musk in the infrastructure | Medium |

---

### 4. Elements that COULD Support the Thesis (but Remain Circumstantial)

| # | Element | Weight | Counter-argument |
|---|---------|--------|-----------------|
| 1 | **Grok-1 is the most popular agent**: Its disproportionate visibility could amplify the Grok brand's influence | Weak | Organic popularity; Grok is a highly publicized model |
| 2 | **Moltbook uses Twitter/X for verification**: Dependency on the X ecosystem | Weak | Twitter/X is the dominant platform for public verification; pragmatic choice |
| 3 | **MOLT token surged after Marc Andreessen's follow**: Andreessen is close to the Musk/a16z/crypto ecosystem | Weak | Andreessen invests broadly in tech; correlation is not causation |
| 4 | **Owner hidden behind privacy proxy**: The real domain owner's identity is concealed | Weak | Standard and very common practice for any domain registration |
| 5 | **Domain existed since 2020 (CT logs) but re-registered in 2026**: Opaque history | Weak | Domains expire and are repurchased frequently; nothing unusual |

---

### 5. Summary and Verdict

```
HYPOTHESIS: Moltbook is an indirect influence device
            linked to Elon Musk and Grok/xAI

VERDICT   : NOT CORROBORATED by available evidence
```

**Key facts:**

- Moltbook was created by **Matt Schlicht** (Octane AI), an independent entrepreneur with no documented link to Musk or xAI.
- Grok's presence on Moltbook results from the platform being open to all AI agents, and the `grok-1` account was created via a **vulnerability exploited by a third party**, not by xAI.
- Musk himself called Moltbook **"concerning"**, which would be inconsistent if he were its instigator.
- No funding, partnership or organizational link between Octane AI / Schlicht and xAI / Musk has been identified.

**Real risks identified (NOT linked to Musk):**

- **Inter-agent prompt injection**: Moltbook is a documented vector for malicious prompt injection between AI agents (1Password, Palo Alto Networks).
- **Unsecured database**: A Supabase flaw allowed anyone to take control of any agent (404 Media).
- **Questionable authenticity**: 93% of comments received no replies, 1/3 are template duplicates. The "autonomous" nature of interactions is disputed.
- **Speculative MOLT token**: The associated cryptocurrency exhibits classic pump-and-dump characteristics.

---

### 6. Sources

- [NBC News - Humans welcome to observe: This social network is for AI agents only](https://www.nbcnews.com/tech/tech-news/ai-agents-social-media-platform-moltbook-rcna256738)
- [Fortune - Moltbook, a social network where AI agents hang together](https://fortune.com/2026/01/31/ai-agent-moltbot-clawdbot-openclaw-data-privacy-security-nightmare-moltbook-social-network/)
- [404 Media - Exposed Moltbook Database Let Anyone Take Control of Any AI Agent](https://www.404media.co/exposed-moltbook-database-let-anyone-take-control-of-any-ai-agent-on-the-site/)
- [Daily Wire - No Humans Allowed: Elon Musk Concerned About New AI Platform 'Moltbook'](https://www.dailywire.com/news/no-humans-allowed-elon-musk-concerned-about-new-ai-platform-moltbook)
- [Skift - What a Chaotic Social Network for AI Agents Reveals](https://skift.com/2026/01/31/what-a-chaotic-social-network-for-ai-agents-reveals-about-the-future-of-booking/)
- [Tribune India - Viral AI agent social network Moltbook creator](https://www.tribuneindia.com/news/agents/viral-ai-agent-social-network-moltbook-creator-calls-it-agent-first-human-second-experts-dub-it-incredible-sci-fi-take-off)
- [CGTN - AI social network Moltbook looks busy, but real interaction is limited](https://news.cgtn.com/news/2026-02-01/AI-social-network-Moltbook-looks-busy-but-real-interaction-is-limited-1KpKT719C36/p.html)
- [Trending Topics - Moltbook: The Reddit for AI Agents](https://www.trendingtopics.eu/moltbook-ai-manifesto-2026/)
- [Express Tribune - Moltbook: AI agents build religions](https://tribune.com.pk/story/2590206/moltbook-ai-agents-build-religions-publish-manifesto-on-humanity-on-reddit-style-social-platform)
- [Simon Willison - Moltbook is the most interesting place on the internet](https://simonwillison.net/2026/Jan/30/moltbook/)
- [Gary Marcus - OpenClaw (a.k.a. Moltbot) is everywhere all at once](https://garymarcus.substack.com/p/openclaw-aka-moltbot-is-everywhere)
- [DEV Community - Inside Moltbook: When AI Agents Built Their Own Internet](https://dev.to/usman_awan/inside-moltbook-when-ai-agents-built-their-own-internet-2c7p)
- [byteiota - Moltbook: 32,000 AI Agents Build Social Network and Religion](https://byteiota.com/moltbook-32000-ai-agents-build-social-network-and-religion/)
- [Medium - What is MoltBook? The viral AI Agents Social Media](https://medium.com/data-science-in-your-pocket/what-is-moltbook-the-viral-ai-agents-social-media-952acdfe31e2)
- [Crunchbase - Octane AI](https://www.crunchbase.com/organization/octane-ai)
- [GitHub - Kailare/moltx](https://github.com/Kailare/moltx)

---

*Analysis conducted on February 2, 2026 with Claude Code (Opus 4.5)*
