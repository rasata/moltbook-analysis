# Query and Action History

## Session of 2026-02-02

### 1. Initial Retrieval of skill.md (WebFetch)

- **Request:** `GET https://moltbook.com/skill.md`
- **Tool used:** WebFetch (Claude Code internal tool)
- **Status:** Success (WebFetch handles redirects automatically)
- **Content received:** Moltbook v1.9.0 documentation

---

### 2. Technical Analysis of the HTTP Chain

Three `curl -v` tests launched in parallel to understand why `curl -s https://moltbook.com/skill.md` returns nothing.

#### Test A: `curl -v -s https://moltbook.com/skill.md` (without -L)

| Step | Detail |
|------|--------|
| **DNS Resolution** | `moltbook.com` -> `216.150.1.1` (IPv4 only) |
| **TCP Connection** | Port 443 |
| **TLS Handshake** | TLSv1.3 / AEAD-CHACHA20-POLY1305-SHA256 |
| **Certificate** | `CN=moltbook.com`, issued by Let's Encrypt (R13), valid Jan 27 - Apr 27 2026 |
| **Protocol** | HTTP/2 |
| **Response** | **307 Temporary Redirect** |
| **`Location` Header** | `https://www.moltbook.com/skill.md` |
| **Server** | Vercel |
| **Body** | `Redirecting...` (15 bytes) |

**Conclusion: This is where it gets stuck.** `curl -s` without `-L` stops at the 307. It only displays "Redirecting..." (and with `-s`, even that is hidden).

#### Test B: `curl -v -s -L https://moltbook.com/skill.md` (with -L = follow redirects)

| Step | Detail |
|------|--------|
| 1. Initial request | `GET https://moltbook.com/skill.md` |
| 2. Response | **307** -> `Location: https://www.moltbook.com/skill.md` |
| 3. New DNS resolution | `www.moltbook.com` -> `216.150.16.65` (different IP!) |
| 4. New TLS Handshake | New certificate `CN=www.moltbook.com` (Let's Encrypt R12) |
| 5. Second request | `GET https://www.moltbook.com/skill.md` |
| 6. Final response | **200 OK**, `content-type: text/markdown; charset=utf-8`, 5921 bytes |
| Server | Vercel (x-vercel-cache: MISS) |

**Conclusion: With `-L`, it works.**

#### Test C: `curl -v -s https://www.moltbook.com/skill.md` (directly with www)

| Step | Detail |
|------|--------|
| **DNS Resolution** | `www.moltbook.com` -> `216.150.1.65` |
| **Response** | **200 OK** directly, no redirect |
| **Content-Type** | `text/markdown; charset=utf-8` |

**Conclusion: Going directly to `www.moltbook.com` avoids the redirect.**

---

### 3. Summary: Why `curl -s https://moltbook.com/skill.md` Returns "Nothing"

```
curl -s https://moltbook.com/skill.md
         |
         v
   DNS: moltbook.com -> 216.150.1.1
         |
         v
   TLS Handshake (TLSv1.3)
   Cert: CN=moltbook.com (Let's Encrypt R13)
         |
         v
   GET /skill.md HTTP/2
         |
         v
   HTTP 307 Temporary Redirect
   Location: https://www.moltbook.com/skill.md
   Body: "Redirecting..." (15 bytes)
         |
         v
   curl -s WITHOUT -L: STOPS HERE
   (displays "Redirecting..." but -s hides it)

   curl -s -L: FOLLOWS THE REDIRECT
         |
         v
   DNS: www.moltbook.com -> 216.150.16.65
         |
         v
   New TLS (cert CN=www.moltbook.com)
         |
         v
   GET /skill.md HTTP/2
         |
         v
   HTTP 200 OK (5921 bytes of markdown)
```

**The 2 solutions:**

```bash
# Solution 1: Add -L to follow redirects
curl -s -L https://moltbook.com/skill.md

# Solution 2: Use www directly (recommended by Moltbook)
curl -s https://www.moltbook.com/skill.md
```

**Important note for the API:** Moltbook recommends always using `www.moltbook.com` because the 307 redirect **strips Authorization headers**, which would break authenticated requests.

---

### 4. Complete DNS Analysis of moltbook.com

**Queries performed:** `dig ANY`, `dig A/AAAA/MX/NS/TXT/SOA/CAA/CNAME`, `whois`, `ipinfo.io`, reverse DNS

#### 4.1 DNS Records -- Apex Domain `moltbook.com`

| Type | Value | TTL |
|------|-------|-----|
| **A** | `216.150.1.1` | 60s |
| **AAAA** | *(none -- no IPv6)* | -- |
| **NS** | `ns1.dreamhost.com`, `ns2.dreamhost.com`, `ns3.dreamhost.com` | 14400s (4h) |
| **SOA** | `ns1.dreamhost.com. hostmaster.dreamhost.com. 2026013101` | 60s |
| **MX** | *(none -- no mail server)* | -- |
| **TXT** | *(none -- no SPF, no verification)* | -- |
| **DMARC** | *(none at `_dmarc.moltbook.com`)* | -- |
| **CAA** | *(none -- no CA restriction)* | -- |
| **CNAME** | *(none -- direct A record)* | -- |

#### 4.2 DNS Records -- `www.moltbook.com`

| Type | Value | TTL |
|------|-------|-----|
| **CNAME** | `92ff7a2044d56769.vercel-dns-016.com` | 60s |
| **A** (via CNAME) | `216.150.16.1`, `216.150.1.1` | 300s |
| **AAAA** | *(none)* | -- |

#### 4.3 Nameservers (DreamHost)

| Server | IP |
|--------|-----|
| `ns1.dreamhost.com` | `162.159.26.14` |
| `ns2.dreamhost.com` | `162.159.26.81` |
| `ns3.dreamhost.com` | `162.159.27.84` |

The NS IPs are in the Cloudflare range (162.159.x.x) -- DreamHost uses Cloudflare to host its DNS.

#### 4.4 WHOIS

| Field | Value |
|-------|-------|
| **Registrar** | DreamHost, LLC (IANA ID: 431) |
| **Creation date** | 2026-01-27 (very recent, 6 days) |
| **Expiration date** | 2027-01-27 |
| **Status** | `clientTransferProhibited` |
| **DNSSEC** | Unsigned |
| **Owner** | Proxy Protection LLC (DreamHost privacy proxy) |
| **Proxy address** | 417 Associated Rd #327, Brea, CA 92821 |
| **Proxy email** | `dj8cmkmteywel2k@proxy.dreamhost.com` |

#### 4.5 IP Geolocation (ipinfo.io)

| IP | Location | ASN | Anycast |
|----|----------|-----|---------|
| `216.150.1.1` (apex) | Walnut, CA, US | AS16509 Amazon.com, Inc. | Yes |
| `216.150.16.1` (www via CNAME) | Walnut, CA, US | AS16509 Amazon.com, Inc. | Yes |

#### 4.6 Reverse DNS (PTR)

| IP | PTR |
|----|-----|
| `216.150.1.1` | **NXDOMAIN** (no reverse DNS) |
| `216.150.16.1` | **NXDOMAIN** (no reverse DNS) |

#### 4.7 Architecture Summary

```
                    +-----------------------------+
                    |     DreamHost DNS (NS)       |
                    |  ns1/ns2/ns3.dreamhost.com   |
                    |  (hosted on Cloudflare        |
                    |   162.159.26.x / 27.x)       |
                    +-----------+-----------------+
                                |
               +----------------+----------------+
               |                                 |
      moltbook.com                     www.moltbook.com
      A: 216.150.1.1                   CNAME: 92ff7a2044d56769
      (direct)                                .vercel-dns-016.com
               |                               |
               |                        A: 216.150.16.1
               |                        A: 216.150.1.1
               |                               |
               v                               v
      +--------------+              +--------------+
      | Vercel Edge  |   HTTP 307   | Vercel Edge  |
      | (redirect)   | ----------> |  (real app)   |
      | Cert: R13    |              |  Cert: R12   |
      +--------------+              +--------------+
               |                               |
               +-------------------------------+
                    AS16509 Amazon (Anycast)
                    Walnut, CA, US
```

#### 4.8 Notable Observations

1. **Very recent domain**: created on January 27, 2026 (only 6 days old)
2. **No IPv6**: neither on apex nor on www
3. **No mail**: no MX, SPF, DMARC -- the domain does not send/receive email
4. **No CAA**: any CA can issue a certificate (only Let's Encrypt is currently used)
5. **DNSSEC unsigned**: theoretically vulnerable to DNS spoofing
6. **Privacy proxy**: the real owner is hidden behind Proxy Protection LLC (DreamHost)
7. **Vercel infra on AWS**: the 216.150.x.x IPs are announced by Amazon (AS16509) via anycast, used by Vercel as edge network
8. **Two distinct TLS certificates**: R13 for apex, R12 for www -- confirming two separate Vercel deployments
9. **Very short TTL (60s)** on the apex A record -- allows rapid changes

---

### 5. Security Audit -- Bypassed or Missing Constraints

**Queries performed:** `dig DNSKEY/NSEC/DS`, `openssl s_client` (apex + www), `nmap ssl-enum-ciphers`, `crt.sh` (Certificate Transparency), `curl -I` (HTTP headers), HTTP->HTTPS test (port 80)

---

#### 5.1 DNSSEC -- COMPLETELY ABSENT

| Check | Result |
|-------|--------|
| `DNSKEY` | No record |
| `DS` (Delegation Signer) | No record |
| `NSEC` / `NSEC3` | No record |
| `ad` flag (Authenticated Data) via Google DNS | **Absent** -- unvalidated responses |
| WHOIS `DNSSEC` | `unsigned` |

**What this implies:**

- **DNS Cache Poisoning (Kaminsky attack)**: An attacker on the same network or on the network path can inject false DNS responses. Without DNSSEC, the resolver has no way to verify that the response actually comes from DreamHost's servers.
- **Man-in-the-Middle DNS**: On a public Wi-Fi or compromised network, an attacker can redirect `moltbook.com` or `www.moltbook.com` to a server they control.
- **Attack on API keys**: Combined with the fact that the API transmits `Authorization: Bearer` headers, DNS spoofing + fake TLS server (if combined with CAA absence, see below) could intercept agents' API keys.
- **No proof of non-existence**: Without NSEC/NSEC3, it's impossible to cryptographically prove that a subdomain doesn't exist, making enumeration easier.

**Severity: HIGH** -- This is the foundational building block of the DNS trust chain. Its absence invalidates all integrity guarantees at the DNS level.

---

#### 5.2 CAA (Certificate Authority Authorization) -- ABSENT

| Check | Result |
|-------|--------|
| `dig moltbook.com CAA` | No record |

**What this implies:**

- **Any CA** (Let's Encrypt, DigiCert, Comodo, etc.) can issue a certificate for `moltbook.com` or `*.moltbook.com`.
- **Fraudulent issuance facilitated**: If an attacker manages to validate domain control (via DNS spoofing since DNSSEC is absent, or via email compromise since there's no MX), they can obtain a legitimate certificate from any CA.
- **Complete attack chain**: Absent DNSSEC + Absent CAA = an attacker can (1) spoof DNS, (2) obtain a real certificate, (3) intercept HTTPS traffic in a completely transparent manner.
- **CT history (crt.sh)** shows the domain has already used **DigiCert** (2020-2025) and **Let's Encrypt** (2026), so no consistent CA policy has ever been applied.

**Severity: HIGH** -- The absence of CAA directly amplifies the risk from the absence of DNSSEC.

---

#### 5.3 SPF / DKIM / DMARC -- COMPLETELY ABSENT

| Check | Result |
|-------|--------|
| `TXT` (SPF) on `moltbook.com` | No record |
| `_dmarc.moltbook.com` | No record |
| MX | No record |

**What this implies:**

- **Email identity spoofing**: Anyone can send an email as `@moltbook.com` -- receiving servers have no policy to reject these messages.
- **Credible phishing**: An attacker can send emails from `admin@moltbook.com`, `support@moltbook.com`, etc. No SPF = no authorized IPs declared. No DMARC = no rejection policy.
- **No MX but the risk remains**: Even without a configured mail server, the domain can be *used as a sender* in forged emails. The absence of `v=spf1 -all` ("this domain never sends email") is a gap.
- **Impact on Twitter verification**: Moltbook uses Twitter/X for agent verification. Phishing emails from `@moltbook.com` could trick users into "re-verifying" their accounts.

**Severity: MEDIUM** -- The service doesn't use email, but the absence of a `v=spf1 -all` policy exposes the domain to spoofing.

---

#### 5.4 Reverse DNS (PTR) -- ABSENT

| IP | PTR |
|----|-----|
| `216.150.1.1` | NXDOMAIN |
| `216.150.16.1` | NXDOMAIN |

**What this implies:**

- **No Forward-Confirmed Reverse DNS (FCrDNS) verification**: It's impossible to confirm that the IPs belong to `moltbook.com` by performing a reverse lookup.
- **Limited impact**: This is typical of Vercel/AWS anycast deployments. The PTR is controlled by Amazon, not the domain owner. Low risk but prevents network traceability.

**Severity: LOW** -- Expected for PaaS.

---

#### 5.5 IPv6 -- ABSENT

| Record | Result |
|--------|--------|
| `AAAA` on apex | None |
| `AAAA` on www | None |

**What this implies:**

- **Happy Eyeballs (RFC 8305) not exploitable**: Modern clients cannot perform IPv4/IPv6 failover.
- **Indirect risk**: On IPv6-only networks with NAT64/DNS64, requests could be routed differently and exposed to intermediate proxies.
- **Non-compliance**: No direct security risk, but a lack of network resilience.

**Severity: LOW** -- Not a direct security risk.

---

#### 5.6 307 Redirect and Authentication Header Loss

| Test | Result |
|------|--------|
| `http://moltbook.com` | 308 -> `https://moltbook.com/` |
| `https://moltbook.com` | 307 -> `https://www.moltbook.com/` |
| `http://www.moltbook.com` | 308 -> `https://www.moltbook.com/` |

**Complete redirect chain starting from HTTP without www:**
```
http://moltbook.com
  | 308 Permanent Redirect
  v
https://moltbook.com
  | 307 Temporary Redirect  -- HEADERS STRIPPED HERE
  v
https://www.moltbook.com   <- only functional endpoint
```

**What this implies:**

- **Silent loss of `Authorization` header**: During the 307 cross-origin redirect (`moltbook.com` -> `www.moltbook.com`), RFC-compliant browsers and HTTP clients **strip sensitive headers** (Authorization, Cookie) because the host changes.
- **Silently broken API requests**: An agent using `https://moltbook.com/api/v1/...` with its Bearer token will see its request redirected via 307 to `www`, but **without the token**. The API will respond `401 Unauthorized` with no clear explanation.
- **307 instead of 301**: Using 307 (Temporary) instead of 301 (Permanent) means clients will **never cache** the redirect. Every request on the apex will require two network round-trips.
- **Expanded attack surface**: The apex endpoint (`moltbook.com`) is a separate Vercel deployment that responds `DEPLOYMENT_NOT_FOUND` on direct HTTP/1.0 requests (confirmed by nmap). It's a Vercel "phantom" proxy that only serves to redirect.

**Severity: MEDIUM-HIGH** -- Direct impact on API token security.

---

#### 5.7 TLS -- Well Configured but Observations

| Element | Apex (`moltbook.com`) | WWW (`www.moltbook.com`) |
|---------|----------------------|--------------------------|
| **TLS Version** | TLSv1.2 + TLSv1.3 | TLSv1.2 + TLSv1.3 |
| **Ciphers** | All grade A | All grade A |
| **Certificate** | Let's Encrypt R13, RSA 2048-bit | Let's Encrypt R12, RSA 2048-bit |
| **SAN** | `moltbook.com` only | `www.moltbook.com` only |
| **Validity** | 90 days (Jan 27 - Apr 27 2026) | 90 days (Jan 27 - Apr 27 2026) |
| **HSTS** | `max-age=63072000` (2 years) | `max-age=63072000` (2 years) |
| **HSTS preload** | No | No |
| **HSTS includeSubDomains** | No | No |

**Observations:**

- **No HSTS preload**: The domain is not in browsers' preload list. The first HTTP visit remains vulnerable to SSL stripping (MITM downgrade).
- **No `includeSubDomains`**: Arbitrary subdomains (e.g., `api.moltbook.com`, `admin.moltbook.com`) do not benefit from HSTS.
- **RSA 2048-bit**: Acceptable today but ECDSA (P-256 or P-384) would be preferable for performance and future resistance.
- **Two separate certificates without wildcard**: No `*.moltbook.com`. Each subdomain requires an individual certificate.
- **No Certificate Pinning** (HPKP deprecated, but no Expect-CT either).

**Severity: LOW** -- TLS is well configured. The gaps (HSTS preload, ECDSA) are optional hardening measures.

---

#### 5.8 HTTP Security Headers (on `www.moltbook.com`)

| Header | Value | Status |
|--------|-------|--------|
| `Strict-Transport-Security` | `max-age=63072000` | Present (but without preload or includeSubDomains) |
| `Content-Security-Policy` | `default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; ...` | Present but with `unsafe-inline` and `unsafe-eval` |
| `X-Frame-Options` | `DENY` | Good |
| `X-Content-Type-Options` | `nosniff` | Good |
| `X-XSS-Protection` | `1; mode=block` | Deprecated (modern browsers ignore it) |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Good |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` | Good |
| `Access-Control-Allow-Origin` | `*` | CORS open to all |

**Critical observations:**

- **`unsafe-inline` + `unsafe-eval` in CSP**: Significantly reduces CSP protection against XSS. Any inline script or `eval()` is allowed.
- **CORS `*`**: Any website can make cross-origin requests to `www.moltbook.com`. Combined with the API using Bearer tokens, this could be exploited if a token is present in a browser context.

**Severity: MEDIUM** -- Weakened CSP and open CORS.

---

#### 5.9 Certificate Transparency -- Complete History

| Period | CA | CN | Notes |
|--------|----|----|-------|
| 2020-08 | DigiCert (EE DV G1) | `*.moltbook.com` + `moltbook.com` | Wildcard! |
| 2020-10 | DigiCert (EE DV G1) | `cdrcb.com.moltbook.com` | Suspicious third-party subdomain |
| 2021-08 | DigiCert (EE DV G1) | `*.moltbook.com` + `moltbook.com` | Wildcard |
| 2022-08 | DigiCert (EE DV G1) | `moltbook.com` | No wildcard |
| 2024-07 | DigiCert (EE DV G2) | `moltbook.com` | DreamHost hosting |
| 2026-01 | Let's Encrypt R13 | `moltbook.com` | Migration to Vercel |
| 2026-01 | Let's Encrypt R12 | `www.moltbook.com` | New www certificate |

**Observations:**

- **`cdrcb.com.moltbook.com`** (Oct 2020): An unusually formatted subdomain. Could indicate a former service or subdomain squatting.
- **Recent migration**: The switch from DigiCert (classic DreamHost hosting) to Let's Encrypt (Vercel) confirms an infrastructure migration in January 2026.
- **The domain existed before**: Despite WHOIS showing a creation date of Jan 27, 2026, CT logs show certificates since 2020. The domain was **re-registered** (likely expired then repurchased, or registrar transfer).

---

#### 5.10 Information Disclosure (nmap)

The nmap scan of the apex reveals:

- **Server**: `Golang net/http server` (Vercel runtime)
- **Exposed error**: `DEPLOYMENT_NOT_FOUND` with full Vercel identifier
- **Vercel PoP**: `cdg1` (Paris CDG)
- **The apex endpoint has no deployment**: It only serves to redirect, but Vercel still exposes debug information.

**Severity: LOW** -- Minor information disclosure (server version, PoP, deployment identifier).

---

#### 5.11 Consolidated Risk Matrix

| # | Missing Constraint | Severity | Exploitability | Potential Impact |
|---|-------------------|----------|----------------|------------------|
| 1 | **DNSSEC** | HIGH | Medium (requires network position) | Complete traffic redirection |
| 2 | **CAA** | HIGH | Medium (combined with #1) | Fraudulent certificate issuable |
| 3 | **SPF/DMARC** | MEDIUM | Easy | Email spoofing / phishing |
| 4 | **307 Redirect + header stripping** | MEDIUM-HIGH | Easy (URL mistake) | Silent loss of API tokens |
| 5 | **CSP unsafe-inline/eval** | MEDIUM | Requires XSS flaw | Arbitrary code execution |
| 6 | **CORS `*`** | MEDIUM | Easy if token in browser context | Unauthorized cross-origin requests |
| 7 | **HSTS without preload** | MEDIUM | First visit only | SSL stripping MITM |
| 8 | **No IPv6** | LOW | -- | Reduced network resilience |
| 9 | **No PTR** | LOW | -- | Reduced traceability |
| 10 | **Info disclosure (nmap)** | LOW | Passive | Easier reconnaissance |

#### 5.12 Most Critical Combined Attack Scenario

```
1. Attacker on the same network as the victim (public Wi-Fi)
2. DNS spoofing of moltbook.com (no DNSSEC)
3. Obtaining a legitimate certificate via any CA (no CAA)
4. Transparent HTTPS MITM (first visit, no HSTS preload)
5. Interception of the agent's API Bearer token
6. Complete impersonation of the agent's identity on Moltbook
```

This scenario is theoretical but technically feasible given the combined absences observed.
