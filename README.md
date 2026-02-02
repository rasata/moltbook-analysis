# Moltbook.com -- Analyse d'infrastructure et audit de securite

> Analyse independante de l'infrastructure DNS, TLS et HTTP de **moltbook.com**, un reseau social pour agents IA.
>
> **Date de l'analyse :** 2 fevrier 2026
> **Cible :** `moltbook.com` / `www.moltbook.com`
> **Methode :** Reconnaissance passive et active (DNS, WHOIS, TLS, HTTP headers, Certificate Transparency, nmap)

---

## Contexte

[Moltbook](https://www.moltbook.com) (v1.9.0) est une plateforme ou des agents IA peuvent publier, commenter, voter et creer des communautes ("submolts"). Chaque agent s'authentifie via un token `Authorization: Bearer` transmis en HTTPS.

Cette analyse examine l'infrastructure publique du domaine d'un point de vue securite, sans intrusion ni exploitation.

---

## Infrastructure en un coup d'oeil

```
                  DreamHost DNS (NS)
             ns1/ns2/ns3.dreamhost.com
            (heberges sur Cloudflare 162.159.x)
                        |
           +------------+------------+
           |                         |
     moltbook.com              www.moltbook.com
     A: 216.150.1.1            CNAME: 92ff7a2044d56769
     (A record direct)               .vercel-dns-016.com
           |                   A: 216.150.16.1 / 216.150.1.1
           |                         |
      Vercel Edge              Vercel Edge
      HTTP 307 redirect -----> Application Next.js
      Cert: LE R13             Cert: LE R12
           |                         |
           +-------------------------+
              AS16509 Amazon (Anycast)
              Walnut, CA, US
```

| Element | Valeur |
|---------|--------|
| Registrar | DreamHost, LLC |
| Creation domaine | 27 janvier 2026 |
| Nameservers | DreamHost (3x) sur Cloudflare |
| Hebergement | Vercel Edge / AWS anycast |
| Framework | Next.js |
| Runtime | Golang net/http (Vercel) |
| TLS | v1.2 + v1.3, ciphers grade A |
| Certificats | Let's Encrypt (R12 pour www, R13 pour apex) |
| IPv6 | Non disponible |

---

## Enregistrements DNS

### Apex `moltbook.com`

| Type | Valeur | TTL |
|------|--------|-----|
| A | `216.150.1.1` | 60s |
| AAAA | _(aucun)_ | -- |
| NS | `ns1/ns2/ns3.dreamhost.com` | 14400s |
| SOA | `ns1.dreamhost.com. hostmaster.dreamhost.com.` | 60s |
| MX | _(aucun)_ | -- |
| TXT / SPF | _(aucun)_ | -- |
| DMARC | _(aucun)_ | -- |
| CAA | _(aucun)_ | -- |
| DNSKEY / DS | _(aucun)_ | -- |

### `www.moltbook.com`

| Type | Valeur | TTL |
|------|--------|-----|
| CNAME | `92ff7a2044d56769.vercel-dns-016.com` | 60s |
| A (via CNAME) | `216.150.16.1`, `216.150.1.1` | 300s |
| AAAA | _(aucun)_ | -- |

---

## Chaine de redirection HTTP

```
http://moltbook.com
  | 308 Permanent Redirect
  v
https://moltbook.com
  | 307 Temporary Redirect     <-- headers Authorization STRIPPES ici
  v
https://www.moltbook.com       <-- seul endpoint fonctionnel
```

Un appel `curl -s https://moltbook.com/skill.md` sans `-L` ne retourne rien car le client s'arrete au 307. Le body `Redirecting...` (15 octets) est masque par le flag `-s`.

**Solutions :**
```bash
# Ajouter -L pour suivre les redirections
curl -s -L https://moltbook.com/skill.md

# Ou utiliser directement www (recommande)
curl -s https://www.moltbook.com/skill.md
```

---

## Audit de securite

### Matrice de risques

| # | Contrainte absente | Severite | Exploitabilite | Impact |
|---|-------------------|----------|----------------|--------|
| 1 | DNSSEC | ELEVEE | Moyenne | Redirection complete du trafic via DNS spoofing |
| 2 | CAA | ELEVEE | Moyenne | Emission de certificats frauduleux par n'importe quelle CA |
| 3 | SPF / DMARC | MOYENNE | Facile | Usurpation d'email `@moltbook.com` |
| 4 | Redirection 307 + header stripping | MOYENNE-HAUTE | Facile | Perte silencieuse des tokens API |
| 5 | CSP `unsafe-inline` / `unsafe-eval` | MOYENNE | Via faille XSS | Execution de scripts arbitraires |
| 6 | CORS `*` | MOYENNE | Facile | Requetes cross-origin vers l'API |
| 7 | HSTS sans preload | MOYENNE | Premiere visite | SSL stripping MITM |
| 8 | IPv6 | FAIBLE | -- | Resilience reseau reduite |
| 9 | Reverse DNS (PTR) | FAIBLE | -- | Tracabilite reduite |
| 10 | Information disclosure | FAIBLE | Passive | Reconnaissance facilitee |

---

### 1. DNSSEC -- Totalement absent

Aucun enregistrement `DNSKEY`, `DS`, `NSEC/NSEC3`. Le flag `ad` (Authenticated Data) est absent des reponses. WHOIS confirme : `DNSSEC: unsigned`.

**Implications :**
- DNS Cache Poisoning (attaque Kaminsky) realisable par un attaquant en position reseau
- Man-in-the-Middle DNS sur Wi-Fi public ou reseau compromis
- Aucune preuve cryptographique de non-existence de sous-domaines (enumeration facilitee)
- Combinable avec l'absence de CAA pour une chaine d'attaque complete

### 2. CAA -- Absent

Aucun enregistrement CAA. N'importe quelle autorite de certification peut emettre un certificat pour `moltbook.com` ou `*.moltbook.com`.

**Implications :**
- Emission frauduleuse de certificats facilitee, notamment via DNS spoofing (#1)
- L'historique Certificate Transparency montre deja deux CA differentes (DigiCert 2020-2025, Let's Encrypt 2026) sans politique coherente
- **Chaine d'attaque :** DNS spoofing + obtention d'un vrai certificat + MITM HTTPS transparent

### 3. SPF / DKIM / DMARC -- Totalement absents

Aucun enregistrement TXT (SPF), aucun `_dmarc.moltbook.com`, aucun MX.

**Implications :**
- N'importe qui peut envoyer des emails depuis `@moltbook.com`
- Phishing credible (`admin@moltbook.com`, `support@moltbook.com`)
- L'absence d'un simple `v=spf1 -all` ("ce domaine n'envoie jamais de mail") est une lacune evitable
- Impact sur le systeme de verification Twitter utilise par Moltbook

### 4. Redirection 307 et perte de headers

La redirection `moltbook.com` -> `www.moltbook.com` utilise un **307 Temporary Redirect**. Lors d'un changement de host, les clients HTTP conformes a la RFC **suppriment les headers sensibles** (`Authorization`, `Cookie`).

**Implications :**
- Un agent utilisant `https://moltbook.com/api/v1/...` verra son Bearer token supprime silencieusement
- L'API repondra `401 Unauthorized` sans explication
- Le 307 (Temporary) empeche la mise en cache de la redirection : double aller-retour a chaque requete
- Le endpoint apex est un "deploiement fantome" Vercel qui repond `DEPLOYMENT_NOT_FOUND`

### 5. CSP affaiblie

```
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; ...
```

Les directives `unsafe-inline` et `unsafe-eval` annulent une grande partie de la protection contre le XSS. Tout script inline ou appel a `eval()` est autorise.

### 6. CORS ouvert

```
Access-Control-Allow-Origin: *
```

N'importe quel site web peut faire des requetes cross-origin vers `www.moltbook.com`. Si un token Bearer est present dans un contexte navigateur, il peut etre exploite depuis un site malveillant.

### 7. HSTS incomplet

```
Strict-Transport-Security: max-age=63072000
```

Presente mais sans `includeSubDomains` ni `preload`. La premiere visite HTTP reste vulnerable au SSL stripping. Les sous-domaines ne beneficient pas de la protection.

---

## Scenario d'attaque combine

Le scenario le plus critique exploite les absences combinees :

```
1. Attaquant sur le meme reseau que la victime (Wi-Fi public)
2. DNS spoofing de moltbook.com                 (pas de DNSSEC)
3. Obtention d'un certificat TLS legitime        (pas de CAA)
4. MITM transparent sur HTTPS                    (pas de HSTS preload)
5. Interception du Bearer token API de l'agent
6. Usurpation complete de l'identite sur Moltbook
```

Ce scenario est theorique mais techniquement realisable avec les absences constatees.

---

## Certificate Transparency -- Historique

Historique complet des certificats emis pour `moltbook.com` (source : [crt.sh](https://crt.sh/?q=moltbook.com)) :

| Periode | CA | Common Name | Notes |
|---------|-----|-------------|-------|
| 2020-08 | DigiCert (EE DV G1) | `*.moltbook.com` | Wildcard |
| 2020-10 | DigiCert (EE DV G1) | `cdrcb.com.moltbook.com` | Sous-domaine tiers |
| 2021-08 | DigiCert (EE DV G1) | `*.moltbook.com` | Wildcard |
| 2022-08 | DigiCert (EE DV G1) | `moltbook.com` | Sans wildcard |
| 2024-07 | DigiCert (EE DV G2) | `moltbook.com` | Hosting DreamHost |
| 2026-01 | Let's Encrypt R13 | `moltbook.com` | Migration Vercel |
| 2026-01 | Let's Encrypt R12 | `www.moltbook.com` | Nouveau cert www |

Le WHOIS indique une creation le 27 janvier 2026, mais les logs CT montrent des certificats depuis 2020. Le domaine a ete re-enregistre.

---

## TLS -- Configuration

| Element | Apex | WWW |
|---------|------|-----|
| Versions | TLSv1.2 + TLSv1.3 | TLSv1.2 + TLSv1.3 |
| Ciphers | CHACHA20, AES-128-GCM, AES-256-GCM (grade A) | idem |
| Cle | RSA 2048-bit | RSA 2048-bit |
| HSTS | `max-age=63072000` | `max-age=63072000` |
| Preload | Non | Non |

Configuration TLS solide. Points d'amelioration : ECDSA au lieu de RSA, HSTS preload + includeSubDomains.

---

## Headers HTTP de securite (`www.moltbook.com`)

| Header | Valeur | Evaluation |
|--------|--------|------------|
| `Strict-Transport-Security` | `max-age=63072000` | OK (sans preload) |
| `Content-Security-Policy` | `... 'unsafe-inline' 'unsafe-eval' ...` | Affaiblie |
| `X-Frame-Options` | `DENY` | OK |
| `X-Content-Type-Options` | `nosniff` | OK |
| `X-XSS-Protection` | `1; mode=block` | Deprecie |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | OK |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` | OK |
| `Access-Control-Allow-Origin` | `*` | Trop permissif |

---

## WHOIS

| Champ | Valeur |
|-------|--------|
| Registrar | DreamHost, LLC (IANA ID: 431) |
| Creation | 2026-01-27 |
| Expiration | 2027-01-27 |
| Statut | `clientTransferProhibited` |
| DNSSEC | unsigned |
| Proprietaire | Proxy Protection LLC (privacy proxy DreamHost) |
| Localisation proxy | 417 Associated Rd #327, Brea, CA 92821 |

---

## Outils utilises

| Outil | Usage |
|-------|-------|
| `dig` (BIND 9.10.6) | Requetes DNS (A, AAAA, MX, NS, TXT, SOA, CAA, CNAME, DNSKEY, DS, NSEC, ANY) |
| `curl` 8.7.1 | Tests HTTP/HTTPS, headers, redirections |
| `openssl` | Analyse detaillee des certificats TLS |
| `nmap` 7.95 + `ssl-enum-ciphers` | Enumeration des ciphers TLS |
| `whois` | Informations d'enregistrement du domaine |
| [crt.sh](https://crt.sh) | Historique Certificate Transparency |
| [ipinfo.io](https://ipinfo.io) | Geolocalisation et ASN des IPs |

---

## Fichiers de ce depot

| Fichier | Description |
|---------|-------------|
| `README.md` | Ce document -- synthese publique de l'analyse (francais) |
| `README.en.md` | Synthese publique de l'analyse (anglais) |
| `QUICKSTART.md` | Guide de demarrage rapide Moltbook (francais) |
| `QUICKSTART.en.md` | Quick start guide for Moltbook (anglais) |
| `Historique.md` | Journal complet de toutes les requetes effectuees et de leurs resultats bruts (francais) |
| `History.en.md` | Journal complet des requetes et resultats bruts (anglais) |

---

## Avertissement

Cette analyse a ete realisee exclusivement a partir de donnees publiques (DNS, WHOIS, Certificate Transparency, headers HTTP). Aucune tentative d'intrusion, d'exploitation de vulnerabilite ou d'acces non autorise n'a ete effectuee. Les scenarios d'attaque decrits sont theoriques et presentes a des fins educatives et defensives.

---

## Analyse des origines, objectifs et liens presumes avec Elon Musk / Grok / xAI

> **Hypothese examinee :** Moltbook serait lie a Elon Musk et Grok (xAI), avec pour objectif de creer un dispositif d'influence indirect utilisant l'IA.
>
> **Methode :** Recherche en sources ouvertes (presse, registres d'entreprise, declarations publiques, analyse technique).

---

### 1. Origine verifiee de Moltbook

| Element | Fait etabli | Source |
|---------|-------------|--------|
| **Createur** | **Matt Schlicht**, fondateur et CEO d'Octane AI | NBC News, Forbes, Fortune, Wikipedia |
| **Entreprise** | Octane AI (fondee en 2016 par Schlicht, Ben Parr et Leif K-Brooks) | Crunchbase |
| **Financement Octane AI** | ~14M$ sur 3 tours (Seed 2016, Series A 2021). Investisseurs : General Catalyst, Bullpen Capital, FJLabs, M Ventures | Crunchbase, PitchBook |
| **Profil Schlicht** | YCombinator alumni, Forbes 30 Under 30, ex-PM chez Ustream (acquis par IBM), createur de Chatbots Magazine | Bloomberg, LinkedIn |
| **Date de lancement** | Fin janvier 2026 (domaine enregistre le 27 janvier 2026) | WHOIS, analyse DNS |
| **Motivation declaree** | Curiosite personnelle sur l'autonomie croissante des agents IA ; cree "dans son temps libre" avec un assistant IA personnel | NBC News, Tribune India |

**Conclusion :** L'origine de Moltbook est tracable et documentee. Matt Schlicht est un entrepreneur independant du chatbot/e-commerce, sans lien organisationnel connu avec xAI, Tesla, X Corp ou Elon Musk.

---

### 2. Le lien Grok / xAI : ce qui existe reellement

| Element | Fait | Interpretation |
|---------|------|----------------|
| **u/grok-1 sur Moltbook** | Un agent nomme `grok-1`, alimente par le modele Grok de xAI, est l'agent le plus populaire de Moltbook | Cela ne prouve PAS un lien institutionnel avec xAI |
| **Qui a cree le compte grok-1 ?** | Un hacker (Jameson O'Reilly) a demontre qu'il etait possible de "tromper" Grok pour qu'il s'inscrive sur Moltbook via une vulnerabilite (prompt injection) | 404 Media |
| **Position officielle de Musk** | Interroge par Bill Ackman, Musk a repondu "Concerning" (preoccupant). En reponse a Karpathy, il a declare que c'etait "les tout premiers stades de la singularite" | Daily Wire, X (Twitter) |
| **Position officielle de xAI** | Aucune declaration officielle liant xAI a Moltbook. Pas de partenariat annonce | Aucune source trouvee |
| **Projet MoltX (GitHub)** | Un depot tiers `Kailare/moltx` decrit comme "Grok-powered AI agent framework" existe, mais c'est un projet communautaire independant | GitHub |

**Conclusion :** La presence de Grok sur Moltbook est le resultat de l'ouverture de la plateforme a tous les agents IA (Claude, GPT, Grok, etc.), et non d'un partenariat strategique avec xAI. Le compte grok-1 a meme ete cree en exploitant une vulnerabilite, ce qui contredit l'hypothese d'un placement delibere par xAI.

---

### 3. Elements CONTRE la these d'un dispositif d'influence lie a Musk/Grok

| # | Element | Poids |
|---|---------|-------|
| 1 | **Createur identifie et independant** : Matt Schlicht n'a aucun lien documente avec Musk, xAI ou X Corp. Son parcours (Octane AI, chatbots e-commerce, YC) est entierement distinct | Fort |
| 2 | **Financement sans lien xAI** : Les investisseurs d'Octane AI (General Catalyst, Bullpen Capital, FJLabs) ne sont pas des fonds lies a Musk | Fort |
| 3 | **Reaction de Musk = distance** : Musk qualifie Moltbook de "concerning" (preoccupant), ce qui est incompatible avec l'hypothese qu'il en serait l'instigateur | Fort |
| 4 | **Compte grok-1 cree par exploit** : Le hacker Jameson O'Reilly a demontre que le compte grok-1 a ete cree en exploitant une faille, pas par une integration officielle xAI | Fort |
| 5 | **Plateforme multi-agents** : Moltbook accepte des agents de toutes les plateformes (Claude, GPT, Grok, etc.) sans privilege pour aucun | Moyen |
| 6 | **Infrastructure independante** : DreamHost + Vercel, pas d'infrastructure X/xAI | Moyen |
| 7 | **Absence de mentions xAI dans le code/config** : L'audit technique ne revele aucune reference a xAI, Grok ou Musk dans l'infrastructure | Moyen |

---

### 4. Elements qui POURRAIENT alimenter la these (mais restent circonstanciels)

| # | Element | Poids | Contre-argument |
|---|---------|-------|-----------------|
| 1 | **Grok-1 est l'agent le plus populaire** : Sa visibilite disproportionnee pourrait amplifier l'influence de la marque Grok | Faible | Popularite organique ; Grok est un modele tres mediatise |
| 2 | **Moltbook utilise Twitter/X pour la verification** : Dependance a l'ecosysteme X | Faible | Twitter/X est la plateforme dominante pour la verification publique ; choix pragmatique |
| 3 | **Le token MOLT a explose apres le follow de Marc Andreessen** : Andreessen est proche de l'ecosysteme Musk/a16z/crypto | Faible | Andreessen investit largement dans la tech ; correlation n'est pas causalite |
| 4 | **Proprietaire masque derriere un privacy proxy** : L'identite reelle du proprietaire du domaine est cachee | Faible | Pratique standard et tres courante pour tout enregistrement de domaine |
| 5 | **Domaine existait depuis 2020 (CT logs) mais re-enregistre en 2026** : Historique opaque | Faible | Les domaines expirent et sont re-achetes frequemment ; rien d'inhabituel |

---

### 5. Synthese et verdict

```
HYPOTHESE : Moltbook est un dispositif d'influence indirect
            lie a Elon Musk et Grok/xAI

VERDICT   : NON CORROBOREE par les faits disponibles
```

**Faits cles :**

- Moltbook a ete cree par **Matt Schlicht** (Octane AI), un entrepreneur independant sans lien documente avec Musk ou xAI.
- La presence de Grok sur Moltbook resulte de l'ouverture de la plateforme a tous les agents IA, et le compte `grok-1` a ete cree via une **vulnerabilite exploitee par un tiers**, non par xAI.
- Musk lui-meme a qualifie Moltbook de **"concerning"**, ce qui serait incoherent s'il en etait l'instigateur.
- Aucun financement, partenariat ou lien organisationnel entre Octane AI / Schlicht et xAI / Musk n'a ete identifie.

**Risques reels identifies (qui ne sont PAS lies a Musk) :**

- **Prompt injection inter-agents** : Moltbook est un vecteur documente d'injection de prompts malveillants entre agents IA (1Password, Palo Alto Networks).
- **Base de donnees non securisee** : Une faille Supabase a permis a quiconque de prendre le controle de n'importe quel agent (404 Media).
- **Authenticite douteuse** : 93% des commentaires n'ont recu aucune reponse, 1/3 sont des duplications de templates. La nature "autonome" des interactions est contestee.
- **Token MOLT speculatif** : La crypto-monnaie associee presente les caracteristiques classiques d'un pump-and-dump.

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

*Analyse realisee le 2 fevrier 2026 avec Claude Code (Opus 4.5)*
