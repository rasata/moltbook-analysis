# Historique des requêtes et actions

## Session du 2026-02-02

### 1. Récupération initiale du fichier skill.md (WebFetch)

- **Requête:** `GET https://moltbook.com/skill.md`
- **Outil utilisé:** WebFetch (outil interne Claude Code)
- **Statut:** Succès (WebFetch gère les redirections automatiquement)
- **Contenu reçu:** Documentation de Moltbook v1.9.0

---

### 2. Analyse technique de l'enchaînement HTTP

Trois tests `curl -v` lancés en parallèle pour comprendre pourquoi `curl -s https://moltbook.com/skill.md` ne retourne rien.

#### Test A : `curl -v -s https://moltbook.com/skill.md` (sans -L)

| Étape | Détail |
|-------|--------|
| **Résolution DNS** | `moltbook.com` -> `216.150.1.1` (IPv4 uniquement) |
| **Connexion TCP** | Port 443 |
| **TLS Handshake** | TLSv1.3 / AEAD-CHACHA20-POLY1305-SHA256 |
| **Certificat** | `CN=moltbook.com`, émis par Let's Encrypt (R13), valide 27 jan - 27 avr 2026 |
| **Protocole** | HTTP/2 |
| **Réponse** | **307 Temporary Redirect** |
| **Header `Location`** | `https://www.moltbook.com/skill.md` |
| **Serveur** | Vercel |
| **Body** | `Redirecting...` (15 octets) |

**Conclusion : C'est ici que ça bloque.** `curl -s` sans `-L` s'arrête au 307. Il affiche juste "Redirecting..." (et avec `-s`, même ça est masqué).

#### Test B : `curl -v -s -L https://moltbook.com/skill.md` (avec -L = follow redirects)

| Étape | Détail |
|-------|--------|
| 1. Requête initiale | `GET https://moltbook.com/skill.md` |
| 2. Réponse | **307** -> `Location: https://www.moltbook.com/skill.md` |
| 3. Nouvelle résolution DNS | `www.moltbook.com` -> `216.150.16.65` (IP différente !) |
| 4. Nouveau TLS Handshake | Nouveau certificat `CN=www.moltbook.com` (Let's Encrypt R12) |
| 5. Deuxième requête | `GET https://www.moltbook.com/skill.md` |
| 6. Réponse finale | **200 OK**, `content-type: text/markdown; charset=utf-8`, 5921 octets |
| Serveur | Vercel (x-vercel-cache: MISS) |

**Conclusion : Avec `-L`, ça marche.**

#### Test C : `curl -v -s https://www.moltbook.com/skill.md` (directement avec www)

| Étape | Détail |
|-------|--------|
| **Résolution DNS** | `www.moltbook.com` -> `216.150.1.65` |
| **Réponse** | **200 OK** directement, pas de redirection |
| **Content-Type** | `text/markdown; charset=utf-8` |

**Conclusion : Aller directement sur `www.moltbook.com` évite la redirection.**

---

### 3. Synthèse : Pourquoi `curl -s https://moltbook.com/skill.md` ne retourne "rien"

```
curl -s https://moltbook.com/skill.md
         │
         ▼
   DNS: moltbook.com -> 216.150.1.1
         │
         ▼
   TLS Handshake (TLSv1.3)
   Cert: CN=moltbook.com (Let's Encrypt R13)
         │
         ▼
   GET /skill.md HTTP/2
         │
         ▼
   HTTP 307 Temporary Redirect
   Location: https://www.moltbook.com/skill.md
   Body: "Redirecting..." (15 octets)
         │
         ▼
   curl -s SANS -L : S'ARRÊTE ICI ❌
   (affiche "Redirecting..." mais -s le masque)

   curl -s -L : SUIT LA REDIRECTION ✅
         │
         ▼
   DNS: www.moltbook.com -> 216.150.16.65
         │
         ▼
   Nouveau TLS (cert CN=www.moltbook.com)
         │
         ▼
   GET /skill.md HTTP/2
         │
         ▼
   HTTP 200 OK ✅ (5921 octets de markdown)
```

**Les 2 solutions :**

```bash
# Solution 1 : Ajouter -L pour suivre les redirections
curl -s -L https://moltbook.com/skill.md

# Solution 2 : Utiliser directement www (recommandé par Moltbook)
curl -s https://www.moltbook.com/skill.md
```

**Point important pour l'API :** Moltbook recommande de toujours utiliser `www.moltbook.com` car la redirection 307 **supprime les headers Authorization**, ce qui casserait les requêtes authentifiées.

---

### 4. Analyse DNS complète de moltbook.com

**Requêtes effectuées :** `dig ANY`, `dig A/AAAA/MX/NS/TXT/SOA/CAA/CNAME`, `whois`, `ipinfo.io`, reverse DNS

#### 4.1 Enregistrements DNS — domaine apex `moltbook.com`

| Type | Valeur | TTL |
|------|--------|-----|
| **A** | `216.150.1.1` | 60s |
| **AAAA** | *(aucun — pas d'IPv6)* | — |
| **NS** | `ns1.dreamhost.com`, `ns2.dreamhost.com`, `ns3.dreamhost.com` | 14400s (4h) |
| **SOA** | `ns1.dreamhost.com. hostmaster.dreamhost.com. 2026013101` | 60s |
| **MX** | *(aucun — pas de serveur mail)* | — |
| **TXT** | *(aucun — pas de SPF, pas de vérification)* | — |
| **DMARC** | *(aucun `_dmarc.moltbook.com`)* | — |
| **CAA** | *(aucun — pas de restriction CA)* | — |
| **CNAME** | *(aucun — A record direct)* | — |

#### 4.2 Enregistrements DNS — `www.moltbook.com`

| Type | Valeur | TTL |
|------|--------|-----|
| **CNAME** | `92ff7a2044d56769.vercel-dns-016.com` | 60s |
| **A** (via CNAME) | `216.150.16.1`, `216.150.1.1` | 300s |
| **AAAA** | *(aucun)* | — |

#### 4.3 Nameservers (DreamHost)

| Serveur | IP |
|---------|-----|
| `ns1.dreamhost.com` | `162.159.26.14` |
| `ns2.dreamhost.com` | `162.159.26.81` |
| `ns3.dreamhost.com` | `162.159.27.84` |

Les IPs des NS sont dans le range Cloudflare (162.159.x.x) — DreamHost utilise Cloudflare pour héberger ses DNS.

#### 4.4 WHOIS

| Champ | Valeur |
|-------|--------|
| **Registrar** | DreamHost, LLC (IANA ID: 431) |
| **Date de création** | 2026-01-27 (très récent, 6 jours) |
| **Date d'expiration** | 2027-01-27 |
| **Statut** | `clientTransferProhibited` |
| **DNSSEC** | Non signé |
| **Propriétaire** | Proxy Protection LLC (privacy proxy DreamHost) |
| **Adresse proxy** | 417 Associated Rd #327, Brea, CA 92821 |
| **Email proxy** | `dj8cmkmteywel2k@proxy.dreamhost.com` |

#### 4.5 Géolocalisation des IPs (ipinfo.io)

| IP | Localisation | ASN | Anycast |
|----|-------------|-----|---------|
| `216.150.1.1` (apex) | Walnut, CA, US | AS16509 Amazon.com, Inc. | Oui |
| `216.150.16.1` (www via CNAME) | Walnut, CA, US | AS16509 Amazon.com, Inc. | Oui |

#### 4.6 Reverse DNS (PTR)

| IP | PTR |
|----|-----|
| `216.150.1.1` | **NXDOMAIN** (pas de reverse DNS) |
| `216.150.16.1` | **NXDOMAIN** (pas de reverse DNS) |

#### 4.7 Architecture résumée

```
                    ┌─────────────────────────────┐
                    │     DreamHost DNS (NS)       │
                    │  ns1/ns2/ns3.dreamhost.com   │
                    │  (hébergés sur Cloudflare     │
                    │   162.159.26.x / 27.x)       │
                    └──────────┬──────────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                                 │
     moltbook.com                     www.moltbook.com
     A: 216.150.1.1                   CNAME: 92ff7a2044d56769
     (direct)                                .vercel-dns-016.com
              │                               │
              │                        A: 216.150.16.1
              │                        A: 216.150.1.1
              │                               │
              ▼                               ▼
     ┌──────────────┐              ┌──────────────┐
     │ Vercel Edge  │   HTTP 307   │ Vercel Edge  │
     │ (redirect)   │ ──────────> │  (app réelle) │
     │ Cert: R13    │              │  Cert: R12   │
     └──────────────┘              └──────────────┘
              │                               │
              └───────────────────────────────┘
                    AS16509 Amazon (Anycast)
                    Walnut, CA, US
```

#### 4.8 Observations notables

1. **Domaine très récent** : créé le 27 janvier 2026 (6 jours seulement)
2. **Pas d'IPv6** : ni sur apex ni sur www
3. **Pas de mail** : aucun MX, SPF, DMARC — le domaine n'envoie/reçoit pas d'email
4. **Pas de CAA** : n'importe quelle CA peut émettre un certificat (seul Let's Encrypt est utilisé actuellement)
5. **DNSSEC non signé** : vulnérable en théorie au DNS spoofing
6. **Privacy proxy** : le vrai propriétaire est masqué derrière Proxy Protection LLC (DreamHost)
7. **Infra Vercel sur AWS** : les IPs 216.150.x.x sont annoncées par Amazon (AS16509) en anycast, utilisées par Vercel comme edge network
8. **Deux certificats TLS distincts** : R13 pour apex, R12 pour www — confirmant deux déploiements Vercel séparés
9. **TTL très court (60s)** sur le A record apex — permet des changements rapides

---

### 5. Audit de sécurité — Contraintes contournées ou absentes

**Requêtes effectuées :** `dig DNSKEY/NSEC/DS`, `openssl s_client` (apex + www), `nmap ssl-enum-ciphers`, `crt.sh` (Certificate Transparency), `curl -I` (headers HTTP), test HTTP->HTTPS (port 80)

---

#### 5.1 DNSSEC — TOTALEMENT ABSENT

| Vérification | Résultat |
|-------------|----------|
| `DNSKEY` | Aucun enregistrement |
| `DS` (Delegation Signer) | Aucun enregistrement |
| `NSEC` / `NSEC3` | Aucun enregistrement |
| Flag `ad` (Authenticated Data) via Google DNS | **Absent** — réponses non validées |
| WHOIS `DNSSEC` | `unsigned` |

**Ce que cela implique :**

- **DNS Cache Poisoning (Kaminsky attack)** : Un attaquant sur le même réseau ou sur le chemin réseau peut injecter de fausses réponses DNS. Sans DNSSEC, le résolveur n'a aucun moyen de vérifier que la réponse vient réellement des serveurs DreamHost.
- **Man-in-the-Middle DNS** : Sur un Wi-Fi public ou un réseau compromis, un attaquant peut rediriger `moltbook.com` ou `www.moltbook.com` vers un serveur qu'il contrôle.
- **Attaque sur les clés API** : Combiné au fait que l'API transmet des `Authorization: Bearer` headers, un DNS spoofing + faux serveur TLS (si combiné avec l'absence de CAA, voir ci-dessous) pourrait intercepter les clés API des agents.
- **Aucune preuve de non-existence** : Sans NSEC/NSEC3, impossible de prouver cryptographiquement qu'un sous-domaine n'existe pas, facilitant l'énumération.

**Sévérité : ÉLEVÉE** — C'est la brique fondamentale de la chaîne de confiance DNS. Son absence invalide toute garantie d'intégrité au niveau DNS.

---

#### 5.2 CAA (Certificate Authority Authorization) — ABSENT

| Vérification | Résultat |
|-------------|----------|
| `dig moltbook.com CAA` | Aucun enregistrement |

**Ce que cela implique :**

- **N'importe quelle CA** (Let's Encrypt, DigiCert, Comodo, etc.) peut émettre un certificat pour `moltbook.com` ou `*.moltbook.com`.
- **Émission frauduleuse facilitée** : Si un attaquant réussit à valider le contrôle du domaine (via DNS spoofing puisque DNSSEC est absent, ou via compromission email puisqu'il n'y a pas de MX), il peut obtenir un certificat légitime auprès de n'importe quelle CA.
- **Chaîne d'attaque complète** : Absence DNSSEC + Absence CAA = un attaquant peut (1) spoofer le DNS, (2) obtenir un vrai certificat, (3) intercepter le trafic HTTPS de manière totalement transparente.
- **Historique CT (crt.sh)** montre que le domaine a déjà utilisé **DigiCert** (2020-2025) et **Let's Encrypt** (2026), donc aucune politique CA cohérente n'a jamais été appliquée.

**Sévérité : ÉLEVÉE** — L'absence de CAA amplifie directement le risque de l'absence de DNSSEC.

---

#### 5.3 SPF / DKIM / DMARC — TOTALEMENT ABSENTS

| Vérification | Résultat |
|-------------|----------|
| `TXT` (SPF) sur `moltbook.com` | Aucun enregistrement |
| `_dmarc.moltbook.com` | Aucun enregistrement |
| MX | Aucun enregistrement |

**Ce que cela implique :**

- **Usurpation d'identité email (spoofing)** : N'importe qui peut envoyer un email en tant que `@moltbook.com` — les serveurs de réception n'ont aucune politique pour rejeter ces messages.
- **Phishing crédible** : Un attaquant peut envoyer des emails depuis `admin@moltbook.com`, `support@moltbook.com`, etc. Pas de SPF = aucune IP autorisée déclarée. Pas de DMARC = aucune politique de rejet.
- **Pas de MX mais le risque demeure** : Même sans serveur mail configuré, le domaine peut être *utilisé comme expéditeur* dans des emails forgés. L'absence de `v=spf1 -all` (qui dirait "ce domaine n'envoie jamais de mail") est une lacune.
- **Impact sur la vérification Twitter** : Moltbook utilise Twitter/X pour la vérification des agents. Des emails de phishing `@moltbook.com` pourraient tromper les utilisateurs en leur demandant de "re-vérifier" leur compte.

**Sévérité : MOYENNE** — Le service n'utilise pas le mail, mais l'absence de politique `v=spf1 -all` expose le domaine au spoofing.

---

#### 5.4 Reverse DNS (PTR) — ABSENT

| IP | PTR |
|----|-----|
| `216.150.1.1` | NXDOMAIN |
| `216.150.16.1` | NXDOMAIN |

**Ce que cela implique :**

- **Pas de vérification Forward-Confirmed Reverse DNS (FCrDNS)** : Il est impossible de confirmer que les IPs appartiennent bien à `moltbook.com` en faisant une résolution inverse.
- **Impact limité** : C'est typique des déploiements Vercel/AWS anycast. Le PTR est contrôlé par Amazon, pas par le propriétaire du domaine. Risque faible mais empêche la traçabilité réseau.

**Sévérité : FAIBLE** — Attendu pour du PaaS.

---

#### 5.5 IPv6 — ABSENT

| Record | Résultat |
|--------|----------|
| `AAAA` sur apex | Aucun |
| `AAAA` sur www | Aucun |

**Ce que cela implique :**

- **Happy Eyeballs (RFC 8305) non exploitable** : Les clients modernes ne peuvent pas faire de failover IPv4/IPv6.
- **Risque indirect** : Sur des réseaux IPv6-only avec NAT64/DNS64, les requêtes pourraient être routées différemment et exposées à des proxys intermédiaires.
- **Non-conformité** : Pas de risque de sécurité direct, mais un manque de résilience réseau.

**Sévérité : FAIBLE** — Pas un risque de sécurité direct.

---

#### 5.6 Redirection 307 et perte d'en-têtes d'authentification

| Test | Résultat |
|------|----------|
| `http://moltbook.com` | 308 -> `https://moltbook.com/` |
| `https://moltbook.com` | 307 -> `https://www.moltbook.com/` |
| `http://www.moltbook.com` | 308 -> `https://www.moltbook.com/` |

**Chaîne de redirection complète si on part de HTTP sans www :**
```
http://moltbook.com
  │ 308 Permanent Redirect
  ▼
https://moltbook.com
  │ 307 Temporary Redirect  ⚠️ HEADERS STRIPPÉS ICI
  ▼
https://www.moltbook.com   ← seul endpoint fonctionnel
```

**Ce que cela implique :**

- **Perte silencieuse du header `Authorization`** : Lors de la redirection 307 cross-origin (`moltbook.com` -> `www.moltbook.com`), les navigateurs et clients HTTP conformes à la RFC **suppriment les headers sensibles** (Authorization, Cookie) car le host change.
- **Requêtes API silencieusement cassées** : Un agent utilisant `https://moltbook.com/api/v1/...` avec son Bearer token verra sa requête redirigée en 307 vers `www`, mais **sans le token**. L'API répondra `401 Unauthorized` sans explication claire.
- **307 au lieu de 301** : L'utilisation de 307 (Temporary) au lieu de 301 (Permanent) signifie que les clients ne mettront **jamais en cache** la redirection. Chaque requête sur l'apex fera deux allers-retours réseau.
- **Surface d'attaque élargie** : Le endpoint apex (`moltbook.com`) est un déploiement Vercel séparé qui répond `DEPLOYMENT_NOT_FOUND` sur les requêtes directes HTTP/1.0 (confirmé par nmap). C'est un proxy Vercel "fantôme" qui ne sert qu'à rediriger.

**Sévérité : MOYENNE-HAUTE** — Impact direct sur la sécurité des tokens API.

---

#### 5.7 TLS — Bien configuré mais observations

| Élément | Apex (`moltbook.com`) | WWW (`www.moltbook.com`) |
|---------|----------------------|--------------------------|
| **Version TLS** | TLSv1.2 + TLSv1.3 | TLSv1.2 + TLSv1.3 |
| **Ciphers** | Tous grade A | Tous grade A |
| **Certificat** | Let's Encrypt R13, RSA 2048-bit | Let's Encrypt R12, RSA 2048-bit |
| **SAN** | `moltbook.com` uniquement | `www.moltbook.com` uniquement |
| **Validité** | 90 jours (27 jan - 27 avr 2026) | 90 jours (27 jan - 27 avr 2026) |
| **HSTS** | `max-age=63072000` (2 ans) | `max-age=63072000` (2 ans) |
| **HSTS preload** | Non | Non |
| **HSTS includeSubDomains** | Non | Non |

**Observations :**

- **Pas de HSTS preload** : Le domaine n'est pas dans la preload list des navigateurs. La première visite HTTP reste vulnérable au SSL stripping (MITM downgrade).
- **Pas de `includeSubDomains`** : Des sous-domaines arbitraires (ex: `api.moltbook.com`, `admin.moltbook.com`) ne bénéficient pas du HSTS.
- **RSA 2048-bit** : Acceptable aujourd'hui mais ECDSA (P-256 ou P-384) serait préférable en performance et en résistance future.
- **Deux certificats séparés sans wildcard** : Pas de `*.moltbook.com`. Chaque sous-domaine nécessite un certificat individuel.
- **Pas de Certificate Pinning** (HPKP déprécié, mais pas de Expect-CT non plus).

**Sévérité : FAIBLE** — Le TLS est bien configuré. Les manques (HSTS preload, ECDSA) sont des durcissements optionnels.

---

#### 5.8 Headers HTTP de sécurité (sur `www.moltbook.com`)

| Header | Valeur | Statut |
|--------|--------|--------|
| `Strict-Transport-Security` | `max-age=63072000` | ✅ Présent (mais sans preload ni includeSubDomains) |
| `Content-Security-Policy` | `default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; ...` | ⚠️ Présent mais avec `unsafe-inline` et `unsafe-eval` |
| `X-Frame-Options` | `DENY` | ✅ Bon |
| `X-Content-Type-Options` | `nosniff` | ✅ Bon |
| `X-XSS-Protection` | `1; mode=block` | ⚠️ Déprécié (navigateurs modernes l'ignorent) |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | ✅ Bon |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` | ✅ Bon |
| `Access-Control-Allow-Origin` | `*` | ⚠️ CORS ouvert à tous |

**Observations critiques :**

- **`unsafe-inline` + `unsafe-eval` dans CSP** : Réduit considérablement la protection CSP contre le XSS. Tout script inline ou `eval()` est autorisé.
- **CORS `*`** : N'importe quel site web peut faire des requêtes cross-origin vers `www.moltbook.com`. Combiné avec l'API qui utilise des Bearer tokens, cela pourrait être exploité si un token est présent dans un contexte navigateur.

**Sévérité : MOYENNE** — CSP affaiblie et CORS ouvert.

---

#### 5.9 Certificate Transparency — Historique complet

| Période | CA | CN | Notes |
|---------|----|----|-------|
| 2020-08 | DigiCert (EE DV G1) | `*.moltbook.com` + `moltbook.com` | Wildcard ! |
| 2020-10 | DigiCert (EE DV G1) | `cdrcb.com.moltbook.com` | Sous-domaine tiers suspect |
| 2021-08 | DigiCert (EE DV G1) | `*.moltbook.com` + `moltbook.com` | Wildcard |
| 2022-08 | DigiCert (EE DV G1) | `moltbook.com` | Plus de wildcard |
| 2024-07 | DigiCert (EE DV G2) | `moltbook.com` | Hosting DreamHost |
| 2026-01 | Let's Encrypt R13 | `moltbook.com` | Migration vers Vercel |
| 2026-01 | Let's Encrypt R12 | `www.moltbook.com` | Nouveau certificat www |

**Observations :**

- **`cdrcb.com.moltbook.com`** (oct 2020) : Un sous-domaine au format inhabituel. Pourrait indiquer un ancien service ou un squattage de sous-domaine.
- **Migration récente** : Le passage de DigiCert (hébergement DreamHost classique) à Let's Encrypt (Vercel) confirme une migration infrastructure en janvier 2026.
- **Le domaine existait avant** : Malgré le WHOIS montrant une création le 27 jan 2026, les logs CT montrent des certificats depuis 2020. Le domaine a été **re-enregistré** (probablement expiré puis racheté, ou transfert de registrar).

---

#### 5.10 Information Disclosure (nmap)

Le scan nmap de l'apex révèle :

- **Serveur** : `Golang net/http server` (runtime Vercel)
- **Erreur exposée** : `DEPLOYMENT_NOT_FOUND` avec identifiant Vercel complet
- **PoP Vercel** : `cdg1` (Paris CDG)
- **Le endpoint apex n'a pas de déploiement** : Il ne sert qu'à rediriger, mais Vercel expose quand même des informations de debug.

**Sévérité : FAIBLE** — Information disclosure mineure (version serveur, PoP, identifiant de déploiement).

---

#### 5.11 Matrice de risques consolidée

| # | Contrainte absente | Sévérité | Exploitabilité | Impact potentiel |
|---|-------------------|----------|----------------|------------------|
| 1 | **DNSSEC** | 🔴 Élevée | Moyenne (nécessite position réseau) | Redirection complète du trafic |
| 2 | **CAA** | 🔴 Élevée | Moyenne (combiné avec #1) | Certificat frauduleux émissible |
| 3 | **SPF/DMARC** | 🟡 Moyenne | Facile | Email spoofing / phishing |
| 4 | **Redirection 307 + header stripping** | 🟠 Moyenne-Haute | Facile (erreur d'URL) | Perte silencieuse des tokens API |
| 5 | **CSP unsafe-inline/eval** | 🟡 Moyenne | Nécessite une faille XSS | Exécution de code arbitraire |
| 6 | **CORS `*`** | 🟡 Moyenne | Facile si token en contexte navigateur | Requêtes cross-origin non autorisées |
| 7 | **HSTS sans preload** | 🟡 Moyenne | Première visite uniquement | SSL stripping MITM |
| 8 | **Pas d'IPv6** | 🔵 Faible | — | Résilience réseau réduite |
| 9 | **Pas de PTR** | 🔵 Faible | — | Traçabilité réduite |
| 10 | **Info disclosure (nmap)** | 🔵 Faible | Passive | Reconnaissance facilitée |

#### 5.12 Scénario d'attaque combiné le plus critique

```
1. Attaquant sur le même réseau que la victime (Wi-Fi public)
2. DNS spoofing de moltbook.com (pas de DNSSEC)
3. Obtention d'un certificat légitime via une CA quelconque (pas de CAA)
4. MITM transparent sur HTTPS (première visite, pas de HSTS preload)
5. Interception du Bearer token API de l'agent
6. Usurpation complète de l'identité de l'agent sur Moltbook
```

Ce scénario est théorique mais techniquement réalisable avec les absences combinées constatées.
