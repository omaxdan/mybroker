# mybroker — Architecture Audit & Recommendation (Phase 0)

> **Status:** Proposal for approval. No implementation has started.
> **Date:** 2026-10-07
> **Scope:** This is the Phase 0 deliverable required before any code is written.
> It states one recommended architecture, the reasoning, the data ownership model,
> the domain model, the API contract shape, the MVP boundary, the cost model, the
> security model, and the migration path.

---

## 0. Repository audit (what exists today)

The repository is effectively empty: a single `README.md` and one initial commit.
There is **no existing code, dependency manifest, database, migration, environment
configuration, authentication, API boundary, or test suite** to preserve or work
around. This is a genuine inception. Nothing below is constrained by legacy; every
decision is free, and the discipline rule (§42 of the brief) is satisfied trivially
— there is nothing to overwrite.

---

## A. Architecture recommendation

### The decision

> **OPTION 2 — Headless WordPress as the content/property system of record,
> plus a lightweight operational database introduced only when assignments go live.**

Phrased against the brief's four options:

| Option | Verdict |
|---|---|
| A — WordPress only | ❌ Rejected as the **end-state**. ✅ but **sufficient for Phase 1 alone**. |
| **B — WordPress + lightweight operational DB** | ✅ **Recommended.** |
| C — PostgreSQL primary + WordPress as CMS | ❌ Rejected — pays for and builds too much, too early. |
| D — PostgreSQL only | ❌ Rejected — rebuilds a CMS and media pipeline we get free. |

### Why the two workloads must not be forced into one technology

The product has two genuinely different workloads:

- **Content / property domain** — properties, listings, media, taxonomies, SEO,
  editorial, broker/firm profiles, verification display. This is **read-heavy,
  editorially managed, media-heavy, SEO-sensitive**. WordPress is excellent at it:
  admin UI, media library, image handling, hierarchical taxonomies, roles, a mature
  SEO plugin ecosystem, and a REST API all arrive for free. Rebuilding this in a
  custom Postgres app before product-market fit is wasted months.

- **Operational / transactional domain** — assignments, a state machine, dispatch,
  broker availability, location, events, audit, matching, notifications. This is
  **write-heavy, state-machine-driven, integrity-critical, eventually geospatial and
  realtime**. WordPress is a *bad* fit here: `wp_postmeta` is an untyped key-value
  (EAV) table with no constraints, poor filtered-query performance, no transactional
  state-machine support, and the brief itself forbids polling WP REST for realtime
  (§6, §23). Putting this in WordPress is the single biggest avoidable mistake.

Because the two workloads have opposite shapes, the right answer is to use each tool
for what it is good at — **Option B**.

### Why not each of the others

- **Option A (WordPress only) as the end-state** fails the operational domain for the
  reasons above. It *does* succeed for **Phase 1**, where there are no assignments,
  no dispatch, and no realtime yet — see §30.
- **Option C (Postgres primary + WP as pure CMS)** means you run *and pay for* a full
  custom application backend **and** WordPress from day one, and you must build a
  sync/ETL pipeline to copy property content out of WP into Postgres, creating a
  duplication and dual-source-of-truth risk. That is more infrastructure, more code,
  and more operational surface — the opposite of asset-light (§2).
- **Option D (Postgres only)** throws away the CMS, media library, admin, and SEO
  ecosystem and forces us to build all of it. Slowest path to a launchable product;
  directly contradicts the resource-constrained reality (§5).

### What "Option B" means concretely across phases

- **Phase 1 runs on WordPress alone** (plus the Next.js frontend). The operational DB
  is **not deployed and not paid for**, because no feature needs it yet. This honours
  the cost principle: *only pay for infrastructure when a user action requires it* (§2).
- **Phase 2 introduces one managed Postgres database** as the system of record for the
  operational domain, the moment assignments become real.

**Recommended operational store: managed PostgreSQL.** Postgres is chosen because it
gives relational integrity for the state machine and audit log, and **PostGIS** for the
proximity matching the assignment network needs (§24). A **managed** instance (not
self-hosted) matches the asset-light, low-ops goal and Uganda connectivity realities.

A strong default is **Supabase** (managed Postgres + PostGIS + row-level security +
built-in realtime over WebSockets + auth). It folds the Phase 4 realtime requirement
and client/broker auth into the same managed service instead of us building a realtime
stack later. This is a recommendation with a reason, not a lock-in: any managed
Postgres works; Supabase simply reduces what we build. We stay on standard SQL and
avoid vendor-only features where practical, so the store remains portable.

---

## B. Source of truth (data ownership)

The rule (§29): **every field has exactly one system of record.** No field is
authored in two systems.

| Entity / field | System of record | Reason |
|---|---|---|
| Property (asset: title, description, type, location text, plot ref, size, tenure) | **WordPress** | Editorial content, SEO, admin-managed. |
| Property coordinates (lat/lng) | **WordPress** | Authored once with the listing; read-only to the app; map stays intent-triggered. |
| Listing (price, currency, availability, cover, status) | **WordPress** | Market-facing content managed by staff. |
| Property images / floor plans / documents | **WordPress media** (offloaded to object storage + CDN) | WP media library + CDN; see §9/cost model. |
| Walkthrough video | **YouTube** (URL/ID stored in WordPress) | Zero video-storage cost; doubles as SEO/marketing (§9). |
| Property type, tenure, amenity, location/area | **WordPress taxonomies** | Hierarchical, filterable, admin-managed. |
| Broker **profile** (name, photo, bio, service areas, firm link) | **WordPress** | Public-facing content. |
| Firm profile | **WordPress** | Public-facing content. |
| Broker **verification status** (public badge) | **WordPress in Phase 1**, → **Operational DB from Phase 2** | Company-controlled early; becomes a governed operational workflow when external brokers join. Handoff is explicit (see migration). |
| Broker **availability / online status** | **Operational DB** | High-frequency operational state; never in WP (§6, §13). |
| Broker **live location** | **Operational system** (ephemeral) | Sensitive, high-frequency; never in WP (§6, §33). |
| Broker workload / performance / acceptance rate | **Operational DB** | Derived operational metrics. |
| Client (identity, contact) | **Operational DB / auth service** | End users; kept **out of `wp_users`** to shrink WP's attack surface. |
| Assignment + all its fields | **Operational DB** | Core transactional entity with a state machine. |
| Assignment status & transitions | **Operational DB** | Auditable state machine (§12, §34). |
| Viewing event | **Operational DB** | Operational timeline. |
| Meeting | **Operational DB** | Operational coordination. |
| Audit / event log | **Operational DB** | Append-only operational history (§34). |
| Notifications | **Operational DB** + delivery provider | Operational. |
| SEO metadata, editorial pages | **WordPress** | Native CMS strength. |

---

## C. Domain model

### WordPress (content/property) — Custom Post Types & taxonomies

Deliberately **few** CPTs (§6, §7 — WordPress must not become a dumping ground):

- **CPT `property`** — the real-world asset.
- **CPT `listing`** — the market-facing offer. **Related to** a property; never merged
  into it (§8). A property can have many listings. In Phase 1 (controlled inventory)
  it is typically 1 property → 1 listing, but we model them separately now so Phase 5
  (external/partner listings on the same asset) needs **no rewrite**.
- **CPT `broker`** — public broker profile.
- **CPT `firm`** — public firm profile.

**Taxonomies (not CPTs):** `property_type`, `tenure`, `amenity`, and `location_area`
(hierarchical: region → district → area/suburb). Locations are a **taxonomy, not a
CPT** — they are classification, not content.

**Why not CPTs for everything in the brief's candidate list:** `assignment_profile` is
really an assignment → operational DB. `location_area` → taxonomy. `amenity` →
taxonomy. This keeps WordPress lean.

**Property fields** live in post meta in Phase 1 (controlled, low-volume inventory, so
`meta_query` filtering is fine). The *search scaling path* (if/when meta queries get
slow) is documented in §D — the frontend never sees it change.

### Operational application (transactional) — relational tables

- **Client** — identity + contact (phone/WhatsApp-first).
- **Assignment** — structured client intent (§12), with a **state machine** (§C.1).
- **Dispatch** — which brokers were offered an assignment, when, and their response.
- **BrokerOperational** — availability, service-area mapping, workload, performance
  (keyed to the WP `broker` profile by stable ID).
- **Viewing**, **Meeting** — operational timeline entities.
- **Event / AuditLog** — append-only record of every transition (§34).
- **LiveLocation** — ephemeral, short-TTL, Phase 4 only; never persisted to WP.

### Entity → system ownership summary

| Entity | WordPress | Operational app |
|---|---|---|
| Property | ✅ | |
| Listing | ✅ | |
| Media (images/docs) | ✅ | |
| Video (YouTube ref) | ✅ (ref only) | |
| Broker (profile) | ✅ | |
| Firm | ✅ | |
| Location / Amenity / Type / Tenure | ✅ (taxonomy) | |
| Verification | ✅ P1 → ➡️ P2 | ✅ from P2 |
| Client | | ✅ |
| Assignment | | ✅ |
| Viewing | | ✅ |
| Meeting | | ✅ |
| Broker availability / location / performance | | ✅ |
| Audit / events / notifications | | ✅ |

### C.1 Assignment state machine

A real state machine, not a free list of statuses (§12). Every transition is written to
the append-only audit log with actor, timestamp, and reason.

```
DRAFT ──submit──▶ SUBMITTED ──match──▶ MATCHING ──offer──▶ OFFERED
                                                             │
                           ┌──────────accept────────────────┤
                           ▼                                 │ (no broker accepts / timeout)
                       ACCEPTED ──start──▶ IN_PROGRESS       ▼
                           │                              (re-offer / EXPIRED)
                           ▼
                   MEETING_PENDING ──▶ VIEWING ──▶ COMPLETED

Terminal/side transitions (allowed from most live states):
  CANCELLED  (client or admin)
  EXPIRED    (timeout, no acceptance)
  REJECTED   (admin / policy)
```

Allowed transitions are explicit and enforced server-side; illegal transitions are
rejected. Phase 2 implements `DRAFT → … → COMPLETED` with **manual/semi-automatic**
matching and offering; Phase 3 automates the `MATCHING → OFFERED` dispatch ranking.

---

## D. API architecture

**The application owns its API contract. The frontend never talks to WordPress
directly and never sees `/wp-json/wp/v2/...`** (§27, §28).

```
          Next.js frontend (public web / mobile web)
                        │   talks ONLY to /api/v1/*
                        ▼
          Application API  (the stable domain contract)
                   │                      │
   (Phase 1: this layer is Next.js        │
    server routes / server components)    │
                   │                      │
                   ▼                      ▼
     WordPress REST (custom,        Operational DB  (Phase 2+)
     namespaced endpoints)         assignments, dispatch,
     properties/listings/media     availability, events, realtime
```

- **Phase 1:** the Application API is simply **Next.js server-side code** (Route
  Handlers / Server Components) calling WordPress. **No separate backend service runs
  or is paid for.** The frontend already calls only `/api/v1/*`, so the contract is
  stable from day one.
- **Phase 2+:** assignment/operational endpoints are **added** to the same `/api/v1`
  surface, backed by the operational DB. Property endpoints are untouched. (If the API
  outgrows Next.js routes it can be extracted to its own service without the frontend
  noticing — the contract is the seam.)

**WordPress exposure:** we publish **custom, namespaced REST endpoints**
(e.g. `mybroker/v1/...`) that return clean domain objects and expose only public,
published fields — never raw post objects. The app reaches WordPress **server-side
only**, with application credentials; browsers never hit WordPress.

### Request routing (which request goes where)

| Request | Served from |
|---|---|
| Search / list properties, filters | WordPress (via app API). Scaling path below. |
| Property / listing detail | WordPress (via app API). |
| Map tiles / coordinates render | **Nothing on page load.** Map JS + tiles load only on explicit "View Map" click (§10, §11). |
| Nearby amenities | On explicit "Explore Nearby" only. |
| Create / view assignment | Operational DB (via app API), Phase 2+. |
| Broker availability / dispatch / location | Operational DB (via app API), Phase 3/4. |
| Realtime updates | Operational realtime channel (Supabase/WebSocket), Phase 4. **Never WP polling.** |

### Example domain object (the contract, independent of WordPress internals)

```json
{
  "id": "property_123",
  "title": "Residential Plot in Kira",
  "type": "residential_plot",
  "price": { "amount": 95000000, "currency": "UGX" },
  "location": {
    "region": "Central", "district": "Wakiso", "area": "Kira",
    "latitude": 0.0, "longitude": 0.0, "has_map": true
  },
  "media": {
    "cover": "https://cdn.example/...jpg",
    "gallery": ["https://cdn.example/...jpg"],
    "video": { "provider": "youtube", "id": "abc123" }
  },
  "attributes": { "land_size": 25, "land_size_unit": "decimals", "tenure": "mailo" },
  "verification": { "status": "verified" },
  "availability": "available"
}
```

Note `location.latitude/longitude` are delivered in the payload, but **rendering a map
from them is deferred to a user click** — the data being present costs nothing; a map
API call is what we gate.

### Search scaling path (so Option B is not naive about WP search)

Phase 1 search = WordPress `meta_query` over controlled, low-volume inventory: cheap and
adequate. If/when it gets slow, the Application API projects property data into a read
model — the operational Postgres (once it exists) or a lightweight search index — and
serves search from there. Because the frontend only calls `/api/v1/search`, this change
is invisible to it.

---

## E. MVP boundary — build now vs. wait

### Build now

- **Phase 0 (this document):** architecture, data ownership, domain model, API contract,
  security model, cost model, migration plan. ← *awaiting approval.*
- **Phase 1 — Property platform (WordPress-only backend):**
  - Headless WordPress with the CPTs/taxonomies above.
  - Custom namespaced WP REST domain endpoints.
  - Next.js public site: **list-based** search + filters (**no map on load**), property
    detail (photos, **lazy/click-to-load** YouTube, location summary, **button-gated**
    interactive map), strong SEO, mobile-first.
  - **"No results → Ask a nearby broker to find one"** CTA (§19). In Phase 1 this
    captures a **lead** delivered to the company (email/WhatsApp), **not** a
    WP-managed operational entity — so there is nothing to migrate when the operational
    DB arrives (keeps source-of-truth clean).
  - Controlled, company-owned inventory; designated brokers as content.
  - Object-storage + CDN media offload; YouTube video embedding.

### Explicitly wait

Operational DB, assignment state machine, dispatch engine, broker availability/live
location, realtime, subscriptions, premium placement, broker analytics, native mobile
apps, AI valuation/recommendation, advanced GIS. These arrive in Phases 2–5, each
turning on its cost only when its feature ships.

The first thing to prove (§36): **can we reliably connect a person looking for property
with the right local property/broker?** Phase 1 + the lead-capture CTA tests the demand
side before we build dispatch.

---

## F. Cost model

Approximate categories (per §F — not invented provider prices), with the key point being
**when each cost turns on**.

| Cost | Phase it turns on | Notes |
|---|---|---|
| Frontend hosting (Next.js) | Phase 1 | Free→low tier (Vercel/Netlify) or a small VPS. |
| WordPress hosting | Phase 1 | Managed WP host or small VPS + MySQL. Low, fixed. |
| Image storage + CDN | Phase 1 | Offload WP media to object storage; **Cloudflare R2 has no egress fees** — attractive for bandwidth-sensitive Uganda traffic. |
| Video storage | **Never (≈ $0)** | YouTube-hosted; we store only the URL/ID. Doubles as marketing/SEO. |
| Maps API | Phase 1 but **near-zero** | Intent-triggered: billed only on explicit "View Map"/"Explore Nearby" clicks, not on page load or crawls (§10, §11). |
| Operational database | **Phase 2** | $0 until assignments exist; then managed Postgres free/low tier. |
| Realtime | **Phase 4** | $0 until live dispatch/tracking; bundled if Supabase. |
| Transactional email | Phase 1–2 | Free tier initially (e.g. SES/Resend-class). |
| WhatsApp | Phase 1 | **Click-to-chat deep links ≈ $0**; WhatsApp Business API (metered) only when automated messaging is needed. |
| SMS | Later | Pay-per-message; add only if WhatsApp coverage is insufficient. |
| Monitoring | Phase 1 | Free tiers (error tracking + uptime). |

Net: **Phase 1 recurring cost ≈ frontend hosting + WP hosting + media CDN**, with maps,
operational DB, realtime, and SMS all at zero until their feature is live. This directly
realizes §2 and §31.

---

## G. Security model

- **Clients:** public read of **published** listings only. Lightweight auth
  (phone/OTP or WhatsApp) **only** to create/track an assignment (Phase 2+). Clients
  are **not** WordPress users.
- **Brokers:** authenticated against the operational/auth service; a broker sees only
  assignments dispatched to them and only the client contact info an accepted assignment
  requires. Brokers never touch WP admin.
- **Admins / company staff:** WordPress admin with scoped capabilities; this is the
  internal content/property CMS (§26).
- **WordPress hardening:** admin behind 2FA and login throttling; XML-RPC and author
  enumeration locked down; the app authenticates to WP REST with application
  credentials/app passwords used **server-side only**; custom endpoints expose only
  public published fields; admin ideally not on the public listing domain.
- **API authentication:** the Application API owns auth (sessions/JWT for clients and
  brokers). Browsers call only `/api/v1/*`; WordPress and the operational DB are never
  reachable from the client.
- **Rate limiting & abuse prevention** at the Application API edge; assignment creation
  rate-limited to deter spam.
- **Private-data boundaries (§32–§34):**
  - **Broker live location is used server-side for matching only.** Clients see
    *"broker found nearby,"* never raw coordinates (§33). Meeting location is shared
    only after acceptance.
  - Client phone numbers are never public; exposed to a broker only on an accepted
    assignment.
  - Assignment contents are private to the client, the matched broker(s), and admins.
  - Every state transition is written to an **append-only audit log** (§34) for
    dispute resolution, fraud detection, and performance tracking.

---

## H. Migration strategy

The whole design is arranged so introducing the operational database is **additive, not
a rewrite** — the public property platform is never rebuilt.

1. **The Application API contract is the seam.** The frontend only ever calls
   `/api/v1/*`. Whatever sits behind it can change without touching the frontend.
2. **Phase 1:** `/api/v1/*` (property/search/detail) is served from WordPress. No
   operational DB exists. The "Ask a broker" CTA is a lead, not a WP operational entity,
   so **nothing operational is trapped in WordPress to migrate later.**
3. **Phase 2:** stand up one managed Postgres. **Add** `/api/v1/assignments`,
   `/api/v1/brokers/{id}/availability`, etc., backed by it. Property endpoints are
   untouched. This is pure addition.
4. **Verification handoff (the one field that changes owner):** in Phase 1 the broker
   verification badge is authored in WordPress (company-controlled brokers). When
   external brokers join (Phase 2+), the operational DB becomes the system of record for
   verification state; WordPress then *displays* a value it reads from the app, it does
   not author it. This handoff is explicit and one-directional, so the dual-source-of-
   truth trap (§29) is avoided.
5. **Stable identifiers:** domain IDs (`property_123`, `broker_45`) are exposed in the
   contract and kept stable, so operational records can reference WordPress content by a
   durable key across the migration.
6. **Search:** if WP search becomes a bottleneck, project property data into a read model
   (Postgres/index) behind the same `/api/v1/search` endpoint — invisible to the
   frontend.

---

## §30 — The explicit database question, answered

> **Can this platform launch successfully with WordPress as the primary
> database/content system and no separate PostgreSQL database?**

**For Phase 1: yes.** The property-discovery platform — search, listings, media,
YouTube video, SEO, controlled inventory, designated brokers, and even the "ask a
broker" *lead* capture — can launch on WordPress alone with the Next.js frontend. No
PostgreSQL is required or paid for in Phase 1.

**For the full product (Phase 2 onward): no.** The assignment network, dispatch,
broker availability/location, the auditable state machine, and realtime cannot live in
WordPress without violating the brief's own constraints (§6, §23) and will not perform
or stay correct there. The **minimum additional infrastructure** is exactly **one
managed PostgreSQL database** (with PostGIS for proximity, and — if Supabase — bundled
realtime and auth). That single addition, introduced only when assignments ship, is the
whole of Option B.

This is the smallest architecture that reliably operates the business and evolves into
the larger platform (§45): **WordPress for everything it is great at, one managed
Postgres added exactly when the operational workload demands it, and a stable
Application API contract so that addition never forces a rewrite.**

---

## Final recommendation

**Adopt Option B.** Build Phase 1 on headless WordPress + Next.js with no operational
database. Add one managed PostgreSQL (Supabase recommended) at Phase 2 when assignments
go live. Keep the Application API as the permanent, stable contract so WordPress
internals are never exposed and the operational DB can be added without rebuilding the
public platform.

Implementation sequence: **Phase 0 (this doc, approve first) → Phase 1 property platform
→ Phase 2 assignment MVP → Phase 3 dispatch → Phase 4 realtime → Phase 5 marketplace.**

*Awaiting approval of this architecture before any implementation begins (§44, §46).*
