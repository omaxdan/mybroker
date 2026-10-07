# mybroker — Deployment & Environments (Architecture Contract)

> **Status:** Proposal for approval. **No production, DNS, hosting, or CDN change has
> been made.** This documents the intended deployment architecture only.
> **Depends on:** `docs/ARCHITECTURE.md` (Option B; production-domain section).
> **Date:** 2026-10-07

---

## 1. Current production state (`www.mybroker.ug`)

| Property | Finding |
|---|---|
| Ownership | Project owns the domain; it is live. |
| DNS | `www.mybroker.ug` **and** apex `mybroker.ug` → `198.54.121.233`. |
| Hosting | **Namecheap shared hosting, cPanel ("Premium")** — reverse DNS `premium68-3.web-hosting.com`. |
| Deployed content | **"Coming soon" placeholder only.** No real site. (Confirmed by owner.) |
| SEO footprint | **None.** `site:mybroker.ug` → zero indexed pages. |
| Existing URLs | **None to preserve.** |
| `cms.` / `staging.` | **Not created yet.** |
| Email (MX) | None observed; to be defined later. |

**Implication:** we start clean. No legacy URL/SEO constraints; the Phase 1 URL structure
(`docs/PHASE-1-DOMAIN-MODEL.md` §12) can be adopted as-is. The only live obligation is to
**not break the placeholder** until the real frontend is ready to cut over.

> The build container's egress policy blocks `mybroker.ug`, so on-page content, SSL
> chain, `robots.txt`, and `sitemap.xml` could not be fetched from here. This is a
> sandbox limitation. Before go-live these should be confirmed directly (or the host
> allow-listed in the environment's network settings) — see §8 risks.

---

## 2. Environments

| Environment | Frontend host | WordPress host | Purpose | Indexable |
|---|---|---|---|---|
| **Local** | `localhost:3000` (Next.js dev) | `localhost:8080` / Local / DDEV / wp-env | Development | No |
| **Staging** | `staging.mybroker.ug` | `cms-staging.mybroker.ug` | Test & approval; mirrors prod | **No** (`noindex` + Basic auth) |
| **Production** | `www.mybroker.ug` (canonical) | `cms.mybroker.ug` | Live | Yes (public frontend only) |

- **Canonical production domain: `www.mybroker.ug`**; apex `mybroker.ug` **301 → www**.
- WordPress is **never** the public site; it is a headless origin + admin at `cms.`.
- `/api/v1/*` is served by the Next.js app (no separate `api.` host in Phase 1).

---

## 3. Topology

```
Public (HTTPS)                 www.mybroker.ug   ─ apex 301 → www
                                      │
                                      ▼
                            Next.js SSR frontend
                        (Vercel / Netlify / Node host)
                                      │  /api/v1/*  (server-side)
                        ┌─────────────┴──────────────┐
                        ▼                             ▼
                cms.mybroker.ug                PostgreSQL (Phase 2+)
            WordPress (cPanel, headless)       managed Postgres/PostGIS
            admin + custom REST only           operational system of record
```

The public frontend runs on a platform built for SSR traffic — **not** the shared cPanel
box, which serves only WordPress at `cms.`.

---

## 4. Recommended hostnames (summary)

| Role | Hostname | Notes |
|---|---|---|
| Public platform (canonical) | `www.mybroker.ug` | Next.js. HSTS once canonical is stable. |
| Apex | `mybroker.ug` | 301 → `www`. |
| WordPress CMS/API (prod) | `cms.mybroker.ug` | Headless origin + admin. Locked down (§6). |
| Public frontend (staging) | `staging.mybroker.ug` | Basic auth + `noindex`. |
| WordPress (staging) | `cms-staging.mybroker.ug` | Basic auth + `noindex`. |
| Future API host (optional) | `api.mybroker.ug` | Only if `/api/v1/*` is extracted from Next.js later. |
| Future CDN | (fronts `www`) | Not configured now. |

---

## 5. DNS & SSL plan (no changes made without approval)

### Intended records (illustrative — exact values set at provisioning time)

| Host | Type | Points to | When |
|---|---|---|---|
| `cms.mybroker.ug` | A | Namecheap cPanel IP (`198.54.121.233`) | When WP install begins (staging first) |
| `cms-staging.mybroker.ug` | A | cPanel IP (or a subfolder/addon domain) | Staging setup |
| `staging.mybroker.ug` | CNAME | Frontend platform's staging target | Staging setup |
| `www.mybroker.ug` | CNAME (or A/ALIAS) | Frontend platform's target | **Cutover only, after approval** |
| `mybroker.ug` (apex) | ALIAS/redirect | → `www` (CNAME flattening or host redirect) | Cutover |
| `mybroker.ug` | MX + TXT (SPF/DKIM/DMARC) | Email provider | When email is added (later) |

### SSL / HTTPS

- HTTPS on every host. Managed TLS on the frontend platform; **cPanel AutoSSL
  (Let's Encrypt)** for `cms.` and `cms-staging.`.
- Enable **HSTS** on `www` only after the canonical + HTTPS behaviour is confirmed stable
  (HSTS is hard to roll back).
- Redirect rules: `http → https` and `apex → www`, everywhere.

### The architecture must support (and does)

HTTPS · www/apex canonicalization · `/api/v1/*` routing · WordPress CMS access at `cms.` ·
staging · a future CDN in front of `www` · future email (MX/SPF/DKIM/DMARC).

### Safe cutover order (production `www`)

1. Build & verify everything on **staging** first.
2. Stand up `cms.mybroker.ug` (prod WordPress) and seed content; keep it non-public.
3. Deploy the production frontend to the platform; verify it over its platform URL.
4. **Lower DNS TTL** on `www` ahead of cutover.
5. Repoint `www` to the frontend platform; set apex 301 → `www`.
6. Verify, then raise TTL and enable HSTS.
7. The coming-soon placeholder keeps serving until step 5 flips `www`.

---

## 6. WordPress hardening at `cms.mybroker.ug`

WordPress is an origin, not a website. Minimum controls:

- **Headless:** no public theme browsing; front-end requests to `cms.` return nothing
  useful (redirect to `www` or 404). Public traffic never lands on `cms.`.
- **Admin protection:** 2FA on all staff; login throttling; restrict `wp-login.php` /
  `wp-admin` (IP allow-list where feasible); unique admin path if practical.
- **REST surface:** expose **only** the custom `mybroker/v1` read endpoints publicly;
  lock default `wp/v2/*` behind authentication; **disable user enumeration**; **disable
  XML-RPC**.
- **App access:** the Next.js server authenticates to WordPress with an **application
  password / scoped credential**, used **server-side only**. The browser never calls
  `cms.`.
- **Private media/data:** title deeds and internal verification evidence are not in the
  public media library (see domain model §13).
- **Staging:** `cms-staging.` behind Basic auth + `noindex`, never indexed.

This enforces the domain model §13 security boundary at the hostname level: WordPress
administrative structures are never reachable through the public domain.

---

## 7. Deployment workflow

```
LOCAL ──▶ STAGING ──▶ TEST / APPROVAL ──▶ PRODUCTION
```

- **Never** develop or test against the live domain (brief requirement).
- Git-based: feature branch → PR → CI (lint, typecheck, tests, build) → deploy to
  **staging** on merge → manual approval → promote to **production**.
- **Frontend** promotes via the hosting platform (immutable build, instant rollback).
- **WordPress** changes (CPTs/fields/config as code where possible, e.g. ACF field groups
  exported to JSON and version-controlled) are applied to `cms-staging.` first, verified,
  then to `cms.`. Content itself is authored in production WordPress by staff.
- **Secrets** (WP app password, DB creds in Phase 2) live in the hosting platform's
  environment config — never committed.
- Each environment is independent; no environment reads another's database.

---

## 8. Risks discovered

| # | Risk | Mitigation |
|---|---|---|
| 1 | Both `www` and apex currently point at shared cPanel; a careless DNS change could break the live placeholder or, later, the real site. | Staged cutover with lowered TTL (§5); placeholder serves until `www` flips. **No DNS change without approval.** |
| 2 | Serving the public SSR app from shared cPanel would not scale and would couple the app to WordPress hosting. | Host the frontend on a platform built for SSR; cPanel hosts only WordPress at `cms.`. |
| 3 | Default WordPress on `cms.` exposes `wp-login`, `wp/v2` REST, user enumeration, XML-RPC to the internet. | Harden per §6 before `cms.` is reachable. |
| 4 | Putting WordPress on `www`/apex would expose WP internals as the public site (violates brief §27). | WordPress is headless at `cms.` only; public is Next.js at `www`. |
| 5 | This container's egress blocks `mybroker.ug`, so live content/SSL/robots/sitemap were not directly verified here. | Confirm directly before go-live, or allow-list the host in the environment's Network settings; low impact given a placeholder-only site. |
| 6 | Apex cookies could leak into `cms.`/`staging.` subdomains if apex were canonical. | Canonical is `www`; scope cookies to `www`. |
| 7 | Staging could get indexed or leak pre-release content. | `noindex` + HTTP Basic auth on all staging hosts. |
| 8 | Email deliverability if MX/SPF/DKIM/DMARC are added hastily later. | Design accommodates them; defined when email is actually introduced. |

---

## 9. Explicit non-actions (per instruction)

Not done, and not to be done without explicit approval: no production change; no
replacement of the current website; no DNS change; no WordPress install on production; no
CDN configured against the live domain. This document is inspection + design only.

*Awaiting approval before any environment is provisioned or any DNS/hosting change is
made.*
