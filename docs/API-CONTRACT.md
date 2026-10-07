# mybroker — API Contract `/api/v1/*` (Architecture Contract)

> **Status:** Proposal for approval. **No implementation has started.**
> **Depends on:** `docs/ARCHITECTURE.md` (Option B), `docs/PHASE-1-DOMAIN-MODEL.md`
> (entities, identifiers, states), `docs/DEPLOYMENT.md` (domains/environments).
> **Date:** 2026-10-07
>
> This document defines the **stable application boundary** between the Next.js frontend
> and the backend systems (WordPress now; PostgreSQL from Phase 2). The contract is
> **domain-oriented, never WordPress-oriented.** The frontend depends only on this
> contract — never on raw WordPress REST, post types, meta keys, ACF, or post IDs.

```
www.mybroker.ug  ──▶  Next.js public app  ──/api/v1/*──▶  { WordPress @ cms. (P1) ,  PostgreSQL (P2+) }
```

---

## 1. API principles

| Concern | Decision |
|---|---|
| **Base URL** | `https://www.mybroker.ug/api/v1` (served by the Next.js app; no public `api.` host in Phase 1). |
| **Versioning** | URI version `/api/v1`. **Additive** changes (new fields, new endpoints, new optional params) are backwards-compatible and do **not** bump the version. **Breaking** changes require `/api/v2` and a deprecation window. |
| **Format** | JSON only, UTF-8, `Content-Type: application/json`. Field names **camelCase** (the WP↔domain mapper converts WP snake_case; see §14). |
| **Identifiers** | Opaque, type-prefixed domain UIDs from the domain model: `prop_…`, `lst_…`, `brk_…`, `frm_…` (ULID body). Path `:id` accepts a **UID or a slug** for properties. **Post IDs, meta keys, and ACF keys are never exposed.** |
| **Dates/money** | Timestamps ISO 8601 UTC (`2026-10-07T12:00:00Z`). Money as `{ "amount": <integer>, "currency": "UGX", "qualifier": "fixed" }`; UGX has no minor unit → whole shillings as integer. |
| **Envelopes** | Collections: `{ "data": [...], "meta": {...} }`. Single resource: the object directly. Errors: `{ "error": {...} }` (§8). |
| **Pagination** | Offset/page in Phase 1; cursor migration path (§9). |
| **Filtering/sorting/search** | Query parameters on `/search` (§3). Unknown query params are **ignored** (forward-compatible); known params are validated. |
| **HTTP status** | `200` ok · `201` created · `304` not modified (ETag) · `400` malformed · `401` unauthorized · `403` forbidden · `404` not found · `409` conflict · `422` validation · `429` rate-limited · `500` server · `503` upstream unavailable. |
| **Caching** | Public GETs are cacheable with `Cache-Control` + `ETag`; on-demand revalidation from WordPress (§10). |
| **Rate limiting** | Per-IP at the app edge. Reads generous; writes strict. `429` + `Retry-After` + `RateLimit-*` headers (§8). |
| **Authentication** | Phase 1 public reads are **unauthenticated** (public data). The app→WordPress call uses a server-side **application password** (never in the browser). Phase 1 writes (`/leads`) are unauthenticated but anti-abuse protected. Phase 2 adds `Authorization: Bearer <token>` for assignment/broker endpoints. |
| **Authorization** | Phase 1: none needed for public reads. Phase 2: role-based (`client`, `broker`, `admin`) + row-level (a broker sees only assignments dispatched to them) — see §12–§13. |
| **Validation** | Params/bodies validated for type, range, and enum membership → `422 VALIDATION_ERROR` with `details.fields`. |
| **Idempotency** | Reads are idempotent. `POST /leads` (and future `POST /assignments`) accept an `Idempotency-Key` header to dedupe retries/double-submits. |
| **Backwards compatibility** | Never remove or repurpose a field within a version; only add. Clients must tolerate unknown fields. Deprecations are announced and dual-served before removal in the next major version. |
| **Request tracing** | Every response carries `X-Request-Id`; it is echoed in `error.requestId` for support. |

---

## 2. Property API — resource strategy (the key decision)

**Decision: the public discovery API is _property-centric_.** The primary public
resources are **`search`** (browse/discover) and **`properties/:id`** (canonical detail).
There is **no public `listings` collection in Phase 1.** The **Property vs Listing
distinction is preserved** by **embedding listings inside the property** (never flattening
them away).

### Reasoning

- The **canonical public URL is the property's** (`PHASE-1-DOMAIN-MODEL.md` §12), and SEO
  wants exactly **one page/card per asset**. A property-centric API yields one result and
  one canonical URL per asset — no duplicate cards when an asset later has several offers.
- The **offer (Listing) is still first-class**: each property response carries a
  `listings` array. Phase 1 has exactly one (company) offer; Phase 5 adds more without any
  contract change — consumers already read an array.
- A plain unfiltered `GET /api/v1/properties` list would duplicate `GET /api/v1/search`
  with no filters. We therefore **do not** ship a separate `properties` collection;
  **`/search` is the canonical collection** (empty query = browse all). This avoids API
  bloat (brief §4, §18).
- A per-offer permalink (`/listings/:id`, `rel=canonical` → property) is **SHOULD-HAVE-
  LATER**, needed only when multiple offers must each be shareable (§18).

### `primaryListing` rule

Where a single headline offer is needed (search cards, OG price), the API uses the
**primary listing** = the top-ranked **published + active** listing for that property
(rank order: premium first [Phase 5], then most recently published). Phase 1: trivially
the one listing.

### Resource shapes

**`PropertySummary`** (search card — compact):
```json
{
  "id": "prop_01J8…",
  "slug": "3-bedroom-home-munyonyo-rd",
  "url": "https://www.mybroker.ug/property/kira/3-bedroom-home-munyonyo-rd",
  "title": "3-Bedroom Home near Munyonyo Road",
  "propertyType": { "slug": "house", "name": "House" },
  "location": { "region": "Central", "district": "Wakiso", "area": "Kira",
                "latitude": 0.0, "longitude": 0.0, "hasMap": true },
  "attributes": { "bedrooms": 3, "bathrooms": 2, "landSize": 25, "landSizeUnit": "decimals" },
  "price": { "amount": 95000000, "currency": "UGX", "qualifier": "negotiable" },
  "availability": "available",
  "verification": { "property": "verified", "listing": "verified" },
  "cover": { "url": "https://cdn…/card.jpg", "width": 640, "height": 427,
             "alt": "Front view", "placeholder": "data:image/…" },
  "media": { "imageCount": 8, "hasVideo": true },
  "listingCount": 1,
  "badges": { "premium": false },
  "updatedAt": "2026-10-01T09:00:00Z"
}
```

**`PropertyDetail`** (full detail / SSR / SEO — superset):
```json
{
  "id": "prop_01J8…",
  "slug": "3-bedroom-home-munyonyo-rd",
  "url": "https://www.mybroker.ug/property/kira/3-bedroom-home-munyonyo-rd",
  "canonicalUrl": "https://www.mybroker.ug/property/kira/3-bedroom-home-munyonyo-rd",
  "title": "3-Bedroom Home near Munyonyo Road",
  "description": "Full marketing description…",
  "propertyType": { "slug": "house", "name": "House" },
  "tenure": { "slug": "mailo", "name": "Mailo" },
  "location": {
    "region": "Central", "district": "Wakiso", "area": "Kira",
    "addressText": "Off Munyonyo Road, Kira",
    "latitude": 0.0, "longitude": 0.0, "hasMap": true
  },
  "attributes": {
    "bedrooms": 3, "bathrooms": 2, "parking": 2,
    "landSize": 25, "landSizeUnit": "decimals",
    "roadAccess": "tarmac",
    "utilities": ["nwsc_water", "grid_power"],
    "amenities": [{ "slug": "fenced", "name": "Fenced" }]
  },
  "media": { "cover": { … }, "gallery": [ … ], "video": { … },
             "floorPlans": [ … ], "documents": [ … ] },
  "listings": [
    {
      "id": "lst_01J8…",
      "price": { "amount": 95000000, "currency": "UGX", "qualifier": "negotiable" },
      "availability": "available",
      "source": { "type": "company", "ref": null },
      "verification": { "status": "verified" },
      "isPremium": false,
      "publishedAt": "2026-09-20T08:00:00Z"
    }
  ],
  "verification": { "property": { "status": "verified" },
                    "listing":  { "status": "verified" } },
  "seo": { … see §11 … },
  "createdAt": "2026-09-01T08:00:00Z",
  "updatedAt": "2026-10-01T09:00:00Z"
}
```

> **Private fields never appear** in these responses: `plotReference`, internal
> verification notes/evidence, admin notes, raw coordinates of anything other than the
> static property location, any PII (`PHASE-1-DOMAIN-MODEL.md` §13). Coordinates are
> present but the **map is intent-triggered** — the API never calls a map provider.

---

## 3. Search API

**`GET /api/v1/search`** — the single discovery/browse resource (property-centric).

### Query parameters

| Param | Type | Notes |
|---|---|---|
| `q` | string | Free text (Phase 1: title + area match). |
| `area` | slug | `location_area` term slug (e.g. `kira`). Repeatable. |
| `district`, `region` | slug | Coarser location filters. |
| `type` | slug | `property_type` (repeatable). |
| `tenure` | slug | `tenure` (repeatable). |
| `amenity` | slug | `amenity` (repeatable; AND semantics). |
| `minPrice`, `maxPrice` | int | UGX. |
| `minSize`, `maxSize` | number | Interpreted with `sizeUnit`. |
| `sizeUnit` | enum | `acres\|decimals\|hectares\|sqm` (default `decimals`). |
| `bedrooms`, `bathrooms` | int | Minimum (`3` = 3+). |
| `availability` | enum | Default `available`; accepts `under_offer`, etc. |
| `verified` | bool | `true` → listing verification = `verified`. |
| `sort` | enum | `newest`(default) `\| price_asc \| price_desc \| size_asc \| size_desc \| relevance`(only with `q`). |
| `page`, `pageSize` | int | Pagination (§9). `pageSize` default 20, max 60. |
| `fields` | csv | Optional projection (e.g. `fields=url,updatedAt` for sitemap builds). |

### Behaviour

- **Default sort:** `newest` (recently published); `relevance` only when `q` is present.
- **Relevance rules (Phase 1):** filter match + recency. **Premium weighting is FUTURE
  (Phase 5)** and, per brief §17, promoted results will be **explicitly labelled**
  (`sponsored` vs `organic`) — paid placement is never disguised as objective relevance.
- **Response:** `{ "data": [ PropertySummary ], "meta": { page, pageSize, total,
  totalPages, sort, appliedFilters } }`.
- **Empty results:** HTTP `200` with `data: []` and `meta.total: 0`. The API returns
  **nothing assignment-related**. The “**No suitable results → Ask a broker**” CTA is a
  **frontend** behaviour triggered off `meta.total === 0`; it submits to
  **`POST /api/v1/leads`** (Phase 1) and, from Phase 2, to `POST /api/v1/assignments`.
  **Search is therefore fully decoupled from the assignment engine** (brief §3, §19) —
  the search contract does not change when assignments arrive.
- **Future extensibility:** `near` (lat,lng + `radius`) geo-filtering is reserved for when
  the search backing store gains PostGIS (`ARCHITECTURE.md` §7); adding it is an additive
  optional param, no breaking change.

---

## 4. Location API

Locations drive the filter UI **and** the indexable area archive pages
(`/properties/<region>/<district>/<area>/`), so they are a genuine public resource — but
only two endpoints, to avoid bloat.

- **`GET /api/v1/locations`** — the hierarchical area tree (region → district → area) with
  published-listing counts. Used for filter menus and the area index.
- **`GET /api/v1/locations/:id`** — one area (by slug or term UID): name, breadcrumb,
  parent/children, listing count, and SEO fields for its archive page.

Other vocabularies needed to render filters (property types, tenures, amenities, price
bounds) come from **`GET /api/v1/facets`** — a single, heavily cacheable endpoint
(can be build-time/ISR generated) returning the controlled vocabularies. This replaces a
scatter of per-taxonomy endpoints.

> **Live broker location is NOT built here.** It is operational, private, Phase 4+, and
> lives in PostgreSQL — never in this public Location API (`PHASE-1-DOMAIN-MODEL.md` §8,
> brief §33).

---

## 5. Media API model

The API returns **ready-to-render media**; the frontend never sees WordPress attachment
IDs, sizes config, or ACF internals (§14). All URLs are **absolute CDN URLs**.

```json
{
  "cover": {
    "url": "https://cdn.mybroker.ug/p/abc/full.jpg",
    "width": 1600, "height": 1067,
    "alt": "Front elevation",
    "placeholder": "data:image/jpeg;base64,…",   // tiny LQIP for fast paint
    "variants": [
      { "label": "card",    "url": "…/card.jpg",    "width": 640,  "height": 427 },
      { "label": "gallery", "url": "…/gallery.jpg", "width": 1024, "height": 683 },
      { "label": "full",    "url": "…/full.jpg",    "width": 1600, "height": 1067 }
    ]
  },
  "gallery": [ { "url": "…", "width": …, "height": …, "alt": "…", "placeholder": "…", "variants": [ … ] } ],
  "video": {
    "provider": "youtube",
    "id": "dQw4w9WgXcQ",
    "url": "https://www.youtube.com/watch?v=dQw4w9WgXcQ",
    "embedUrl": "https://www.youtube-nocookie.com/embed/dQw4w9WgXcQ",
    "thumbnail": { "url": "https://cdn…/poster.jpg", "width": 1280, "height": 720 }
  },
  "floorPlans": [ { "url": "…", "width": …, "height": …, "alt": "Ground floor" } ],
  "documents": [ { "url": "…", "title": "Brochure", "type": "pdf" } ]
}
```

- `variants` carry the responsive sizes the frontend turns into `srcset`.
- `video` is **YouTube-only** in Phase 1; `embedUrl` uses the privacy-enhanced host and is
  loaded **lazily (click-to-load)** — no third-party JS until the user clicks.
- `video.thumbnail` is the custom poster if set, else the YouTube thumbnail.
- **Private documents (e.g. title deeds) are never included** — only public brochures/
  floor plans (`PHASE-1-DOMAIN-MODEL.md` §9, §13).

---

## 6. Listing status — three orthogonal axes (never one field)

The API exposes three **separate** concepts (brief §6; `PHASE-1-DOMAIN-MODEL.md` §1.5).
A listing can be `published` **and** `under_offer` **and** `verified` simultaneously.

| Axis | API field | Values | Meaning |
|---|---|---|---|
| **Editorial / publishing** | *not publicly exposed per se* | internal `draft \| pending \| publish \| private \| trash` | Governs **whether the resource appears at all**. The public API only ever returns **published** content; unpublished items 404. |
| **Market availability** | `availability` (on listing + surfaced on summary) | `available \| under_offer \| reserved \| taken \| withdrawn` | The offer's market state. |
| **Verification** | `verification` (three sub-fields, §7) | see §7 | Trust, independent of the above. |

Notes:
- Publishing state is deliberately **not** a public enum value — the public never sees
  drafts; exposing it would leak editorial workflow. (Admin tooling sees it internally.)
- `property_status` (`active | archived`) is an **asset-lifecycle** concept, also internal;
  archived assets with no published listing simply 404 publicly.

---

## 7. Verification representation

Three independent public badges, each a simple status — **no generic `verified=true`**,
and **no internal evidence** (brief §7; `PHASE-1-DOMAIN-MODEL.md` §10).

```json
"verification": {
  "property": { "status": "verified" },     // unverified | pending | verified | disputed
  "listing":  { "status": "verified" }      // unverified | pending | verified
}
```

- **Broker verification** (`unverified | pending_review | verified | suspended | rejected`)
  appears on the (later) broker resource as `verification.broker`.
- **Public exposes only `status`.** Verification notes, documents, reviewer identity, and
  dates are **private** and never serialized to the public API.
- System of record: property & listing verification = WordPress; **broker verification
  moves to PostgreSQL in Phase 2** and is then read-through (`ARCHITECTURE.md` §H).

---

## 8. Error contract

One structure for every error. **WordPress/PHP/MySQL errors are never leaked** — the API
layer catches upstream failures and returns a clean envelope while logging details
server-side with the `requestId`.

```json
{
  "error": {
    "code": "PROPERTY_NOT_FOUND",
    "message": "Property not found.",
    "details": {},
    "requestId": "req_01J8…"
  }
}
```

| Code | HTTP | When |
|---|---|---|
| `BAD_REQUEST` | 400 | Malformed request / unparseable params. |
| `VALIDATION_ERROR` | 422 | Known param/body fails validation; `details.fields` lists each. |
| `UNAUTHORIZED` | 401 | Missing/invalid auth (Phase 2 endpoints). |
| `FORBIDDEN` | 403 | Authenticated but not permitted. |
| `NOT_FOUND` / `PROPERTY_NOT_FOUND` / `LOCATION_NOT_FOUND` | 404 | Resource absent or unpublished. |
| `CONFLICT` | 409 | State conflict (e.g. duplicate idempotency key with different body). |
| `RATE_LIMITED` | 429 | Rate limit exceeded; `Retry-After` + `RateLimit-*` headers. |
| `UPSTREAM_UNAVAILABLE` | 503 | WordPress/operational backend unavailable and no cache to serve (§15). |
| `INTERNAL_ERROR` | 500 | Unexpected server error (generic message; no stack/DB details). |

---

## 9. Pagination

- **Phase 1: offset/page** — `?page=1&pageSize=20`. Simple and sufficient for controlled
  inventory.
- **Defaults/limits:** `pageSize` default **20**, **max 60** (over-max → clamped, not an
  error).
- **Response metadata:** `meta: { page, pageSize, total, totalPages, sort, appliedFilters }`.
- **Migration path to cursor:** when datasets grow or deep pagination hurts, add an
  **optional** `cursor` param and `meta.nextCursor`/`meta.prevCursor` **alongside** the
  page fields (additive, non-breaking). Offset may later be deprecated for very deep
  ranges while the envelope stays stable. No `v2` required to introduce cursors.

---

## 10. Caching

Property discovery is read-heavy; caching uses the **simplest effective** layering — **no
Redis in Phase 1** (brief §10).

1. **HTTP:** public GETs return `Cache-Control: public, s-maxage=…, stale-while-revalidate=…`
   and a strong `ETag` (→ `304` on revalidation).
2. **Next.js data/route cache (ISR):** property detail and area pages are cached/
   incrementally regenerated. This is the primary server-side cache; no extra
   infrastructure.
3. **On-demand revalidation:** WordPress `save_post`/`transition_post_status` fires a
   signed webhook to an **internal** `POST /api/v1/revalidate` (shared-secret, not public),
   purging exactly the affected property/area so edits appear promptly without blanket TTL
   waits.
4. **WordPress response caching:** a lightweight page/object cache on `cms.` (cPanel-level
   or a cache plugin) shields WordPress from repeated identical reads.
5. **CDN (future):** when a CDN fronts `www`, the same `Cache-Control`/`ETag` headers make
   API and page responses edge-cacheable with no contract change.

Write endpoints (`/leads`) and any Phase 2 authenticated endpoints are **never cached**
(`Cache-Control: no-store`).

---

## 11. SEO support (server-rendered)

`GET /api/v1/properties/:id` returns, **in one server-side call**, everything needed to
render the page's `<head>` — the frontend makes **no browser-side request to obtain
SEO-critical data** (brief §11).

```json
"seo": {
  "title": "3-Bedroom Home near Munyonyo Road, Kira | mybroker",
  "description": "3-bed, 2-bath home on 25 decimals in Kira, Wakiso. UGX 95,000,000.",
  "canonicalUrl": "https://www.mybroker.ug/property/kira/3-bedroom-home-munyonyo-rd",
  "openGraph": {
    "title": "3-Bedroom Home near Munyonyo Road",
    "description": "3-bed, 2-bath home on 25 decimals in Kira.",
    "type": "website",
    "image": "https://cdn.mybroker.ug/p/abc/og.jpg",
    "url": "https://www.mybroker.ug/property/kira/3-bedroom-home-munyonyo-rd"
  },
  "structuredData": {
    "@context": "https://schema.org",
    "@type": "Residence",
    "name": "3-Bedroom Home near Munyonyo Road",
    "address": { "@type": "PostalAddress", "addressRegion": "Central",
                 "addressLocality": "Kira", "addressCountry": "UG" },
    "offers": { "@type": "Offer", "price": 95000000, "priceCurrency": "UGX",
                "availability": "https://schema.org/InStock" }
  }
}
```

- **Sitemap generation:** the frontend enumerates published properties via
  `GET /api/v1/search?fields=url,updatedAt&pageSize=60&page=N` (compact projection) to
  build `sitemap.xml`. A dedicated minimal sitemap endpoint is a possible later
  optimization (§18), not required now.
- Availability, location, images, canonical URL, and OG/structured data are all present in
  `PropertyDetail`, so SSR/SSG needs exactly one upstream fetch per page.

---

## 12. Future Assignment API (documented boundary — **not built**)

Phase 2+. Backed by **PostgreSQL** (`ARCHITECTURE.md` §B; `PHASE-1-DOMAIN-MODEL.md` §3).

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/assignments` | POST | Create an assignment (structured client intent). Auth: client. `Idempotency-Key`. |
| `/api/v1/assignments/:id` | GET | Read one assignment. Auth: owning client / matched broker / admin. |
| `/api/v1/assignments/:id/cancel` | POST | Client/admin cancel → state machine (`PHASE-1-DOMAIN-MODEL.md` §C.1). |
| `/api/v1/assignments/:id/transitions` | GET | Audit trail of state changes (auth-scoped). |

**Coexistence with the Property API:** assignments are a **separate resource** that
*references* content by stable UID (`matched_listing_uid`, `property_uid`). They are
**added** under the same `/api/v1` surface; **nothing in the property/search/location API
changes**. The “ask a broker” CTA merely switches its submit target from `/leads` (Phase 1)
to `/assignments` (Phase 2). The property API therefore needs **no restructuring** when
assignments ship (brief §12).

---

## 13. Future Broker API (documented boundary — **not built**)

Separate **public profile** (WordPress) from **private operational** data (PostgreSQL).

| Endpoint | Method | Phase | Source | Notes |
|---|---|---|---|---|
| `/api/v1/brokers` | GET | SHOULD-LATER | WordPress | Public broker profiles (name, photo, areas, public verification badge). |
| `/api/v1/brokers/:id` | GET | SHOULD-LATER | WordPress | One public profile. |
| `/api/v1/brokers/:id/availability` | GET/PUT | FUTURE (P3/4) | PostgreSQL | **Private/operational.** Auth: broker/admin. |
| `/api/v1/dispatch/*`, `/acceptance`, `/meetings`, `/viewings` | — | FUTURE (P3/4) | PostgreSQL | Operational; auth + row-level scoping. |

**Live broker location is never public**; it is used server-side for matching only, and
the client sees only “broker found nearby” (brief §33). Public broker endpoints expose
**no** availability, location, workload, or client PII.

---

## 14. WordPress boundary (how the translation works)

```
Next.js page / server component
        │  (server-side fetch)
        ▼
/api/v1/*   ← THE CONTRACT (this document). camelCase domain objects.
        │  (Next.js route handler / data layer = the mapper)
        ▼
WordPress custom REST:  https://cms.mybroker.ug/wp-json/mybroker/v1/*
        │  (NOT wp/v2; published, public-safe fields only)
        ▼
WordPress core (CPTs, taxonomies, post meta, ACF)  ← never seen by the frontend
```

- The **mapper** (the only code that knows WordPress exists) converts post/taxonomy/meta/
  ACF into the domain objects above: resolves `*_uid` ↔ post, builds absolute CDN media
  URLs + variants, parses the YouTube id, maps snake_case→camelCase, strips private fields,
  and composes `listings`/`verification`/`seo`.
- **Frontend code must never contain** logic such as `post_type === "property"`,
  `acf.property_price`, `meta._property_location`, or any `post_id`. Such logic lives
  **only** behind `/api/v1`. (This is the explicit prohibition in brief §14.)
- Even the WordPress custom-endpoint shape (`mybroker/v1`) can change without touching the
  frontend, because `/api/v1` — not `mybroker/v1` — is the frontend's contract.

---

## 15. WordPress failure behaviour

A CMS outage must degrade **freshness**, not **availability**, of the public platform
(brief §15).

| Surface | Behaviour when WordPress (`cms.`) is unavailable |
|---|---|
| Property detail & area pages | Served from the **Next.js ISR cache / last good static render** (`stale-while-revalidate`, `stale-if-error`). Users see slightly stale content, not an error. |
| Search | If still WP-backed (Phase 1): cached queries served stale; uncached queries return `503 UPSTREAM_UNAVAILABLE` with `Retry-After` and a soft frontend message. From Phase 2, search backed by the Postgres read-model is **independent of WordPress uptime**. |
| `POST /leads` | **Store-and-forward**: accept, enqueue, retry delivery — a lead is **never lost** to a CMS blip. |
| `/api/v1/health` | Reports degraded upstream so monitoring/alerting fires. |

Principle: static/ISR generation + `stale-if-error` means a `cms.` outage does **not** take
down `www`.

---

## 16. PostgreSQL migration seam

The contract stays stable while the implementation beneath it changes.

```
PHASE 1                               PHASE 2+
/api/v1/search       → WordPress      /api/v1/search       → WordPress  (or Postgres read-model)
/api/v1/properties/* → WordPress      /api/v1/properties/* → WordPress   (unchanged)
/api/v1/locations/*  → WordPress      /api/v1/locations/*  → WordPress   (unchanged)
/api/v1/leads        → email/WhatsApp /api/v1/assignments  → PostgreSQL  (ADDED)
                                      /api/v1/brokers/*/availability → PostgreSQL (ADDED)
```

- Phase 2 is **purely additive**: new endpoints backed by Postgres; existing property/
  search/location endpoints untouched → property pages, search UX, listing URLs, SEO,
  media, and frontend architecture are **not rebuilt** (`PHASE-1-DOMAIN-MODEL.md` §6).
- Cross-system joins use the **stable UIDs**, never WordPress post IDs (§1, domain model
  §4), so operational records reference content durably.
- If `/search` later moves to a Postgres/PostGIS read-model, it is the **same endpoint** —
  invisible to the frontend (`ARCHITECTURE.md` §7).

---

## 17. Per-endpoint specification (Phase 1)

Format per brief §17: Endpoint · Purpose · Method · Auth · Parameters · Request body ·
Response · Errors · Caching · Source of truth · Phase.

### 17.1 `GET /api/v1/search`
- **Purpose:** Browse/discover properties (property-centric cards). The canonical collection.
- **Method:** GET · **Auth:** none (public).
- **Parameters:** §3 (q, area/district/region, type, tenure, amenity, min/maxPrice,
  min/maxSize+sizeUnit, bedrooms, bathrooms, availability, verified, sort, page, pageSize,
  fields).
- **Request body:** none.
- **Response:** `200 { data: [PropertySummary], meta: { page, pageSize, total, totalPages,
  sort, appliedFilters } }`. Empty → `200` with `total: 0`.
- **Errors:** `422 VALIDATION_ERROR` (bad param), `429`, `503`.
- **Caching:** `public, s-maxage` + `stale-while-revalidate` + ETag.
- **Source of truth:** WordPress (Phase 1). **Phase:** MUST HAVE NOW.

### 17.2 `GET /api/v1/properties/:id`
- **Purpose:** Canonical property detail (SSR/SEO). `:id` = property UID **or** slug.
- **Method:** GET · **Auth:** none.
- **Parameters:** path `:id`. Optional `?fields=` projection.
- **Response:** `200 PropertyDetail` (includes `listings`, `media`, `verification`, `seo`).
  Slug that differs from canonical → response carries `canonicalUrl` so the page can 301.
- **Errors:** `404 PROPERTY_NOT_FOUND` (absent/unpublished), `429`, `503`.
- **Caching:** ISR + on-demand revalidation (§10); ETag.
- **Source of truth:** WordPress. **Phase:** MUST HAVE NOW.

### 17.3 `GET /api/v1/locations`
- **Purpose:** Hierarchical area tree + counts (filters, area index).
- **Method:** GET · **Auth:** none · **Parameters:** optional `?region=`/`?district=` to scope.
- **Response:** `200 { data: [AreaNode] }`.
- **Errors:** `503`. **Caching:** long `s-maxage` (vocab changes rarely) + revalidation.
- **Source of truth:** WordPress (`location_area` taxonomy). **Phase:** MUST HAVE NOW.

### 17.4 `GET /api/v1/locations/:id`
- **Purpose:** One area (slug or term UID): breadcrumb, children, count, SEO for the
  archive page.
- **Method:** GET · **Auth:** none · **Parameters:** path `:id`.
- **Response:** `200 Area`. **Errors:** `404 LOCATION_NOT_FOUND`, `503`.
- **Caching:** as 17.3. **Source of truth:** WordPress. **Phase:** MUST HAVE NOW.

### 17.5 `GET /api/v1/facets`
- **Purpose:** Controlled vocabularies + price bounds for the search filter UI.
- **Method:** GET · **Auth:** none · **Parameters:** none.
- **Response:** `200 { propertyTypes:[…], tenures:[…], amenities:[…], priceBounds:{min,max,currency} }`.
- **Errors:** `503`. **Caching:** long `s-maxage`; may be **build-time/ISR** generated.
- **Source of truth:** WordPress taxonomies. **Phase:** MUST HAVE NOW.

### 17.6 `POST /api/v1/leads`
- **Purpose:** Capture a client request, including the “**No results → ask a broker**”
  conversion (brief §19). **Decoupled from search and from the future assignment engine.**
- **Method:** POST · **Auth:** none, but **anti-abuse**: strict rate limit, honeypot field,
  optional CAPTCHA; `Idempotency-Key` supported.
- **Request body:**
  ```json
  { "name": "…", "phone": "+2567…", "contactPreference": "whatsapp|call",
    "message": "…", "context": { "searchQuery": { … }, "propertyId": "prop_…" },
    "consent": true }
  ```
- **Response:** `201 { data: { id: "lead_…", status: "received" } }`.
- **Errors:** `422 VALIDATION_ERROR`, `429 RATE_LIMITED`, `409` (idempotency replay with
  changed body).
- **Caching:** `no-store`.
- **Source of truth:** **Not WordPress** — delivered to the company inbox/WhatsApp
  (store-and-forward); optional minimal record outside WordPress. In Phase 2 the CTA
  migrates to `POST /api/v1/assignments` (PostgreSQL); **this endpoint’s existence keeps
  search decoupled from assignments**. **Phase:** MUST HAVE NOW.

### 17.7 `POST /api/v1/revalidate` (internal)
- **Purpose:** WordPress → Next.js cache purge on content change (§10).
- **Method:** POST · **Auth:** shared secret (HMAC-signed); **not public**.
- **Request body:** `{ "entity": "property|listing|location", "uid": "prop_…" }`.
- **Response:** `200 { revalidated: true }`. **Errors:** `401`, `422`.
- **Caching:** `no-store`. **Source of truth:** n/a (control plane). **Phase:** MUST HAVE NOW (internal).

### 17.8 `GET /api/v1/health`
- **Purpose:** Liveness + upstream status for monitoring.
- **Method:** GET · **Auth:** none (no sensitive detail).
- **Response:** `200 { status: "ok|degraded", upstream: { wordpress: "ok|down" } }`.
- **Caching:** `no-store`. **Source of truth:** n/a. **Phase:** SHOULD HAVE LATER (trivial; recommended from day one).

---

## 18. Endpoint classification

### MUST HAVE NOW (Phase 1)
- `GET /api/v1/search`
- `GET /api/v1/properties/:id`
- `GET /api/v1/locations`
- `GET /api/v1/locations/:id`
- `GET /api/v1/facets`
- `POST /api/v1/leads`
- `POST /api/v1/revalidate` (internal control plane)

### SHOULD HAVE LATER
- `GET /api/v1/health` (recommended immediately; not a blocker)
- `GET /api/v1/listings/:id` (shareable per-offer permalink, `rel=canonical` → property) —
  when a property carries multiple offers
- `GET /api/v1/brokers`, `GET /api/v1/brokers/:id` (public broker profiles) — when brokers
  are surfaced publicly
- Dedicated `GET /api/v1/properties/sitemap` projection — if the `fields=` approach proves
  insufficient at scale

### FUTURE (Phase 2+)
- `POST /api/v1/assignments`, `GET /api/v1/assignments/:id`,
  `POST /api/v1/assignments/:id/cancel`, `GET /api/v1/assignments/:id/transitions`
- Broker operational: `…/availability`, dispatch, acceptance, meetings, viewings
- `GET /api/v1/listings` (collection; admin/marketplace)
- Auth endpoints (client/broker sessions/tokens)

> No endpoint is created merely because it may someday be useful (brief §18, §4).

---

## 19. Cross-document consistency validation

Checked against `ARCHITECTURE.md`, `DEPLOYMENT.md`, `PHASE-1-DOMAIN-MODEL.md`:

| Dimension | Result |
|---|---|
| **Identifiers** | ✅ Prefixed-ULID UIDs (`prop_/lst_/brk_/frm_`); slug resolves; **no post IDs / meta keys / ACF** exposed — matches domain model §4. |
| **Property vs Listing** | ✅ Preserved: property-centric API **embeds** `listings[]`; no public listings collection in P1; distinction intact (domain model §2). |
| **WordPress ownership** | ✅ Property, listing, media, location, property/listing verification = WordPress, served via `mybroker/v1` on `cms.` (domain model §1, §3). |
| **PostgreSQL ownership** | ✅ Assignments, broker operational/availability/location, dispatch, events = PostgreSQL, Phase 2+, all under FUTURE endpoints (§12–§13; architecture §B). |
| **Domains** | ✅ API base `https://www.mybroker.ug/api/v1`; WordPress at `cms.mybroker.ug/wp-json/mybroker/v1` — matches deployment §2–§4. |
| **Environment boundaries** | ✅ No endpoint reaches across environments; WordPress reached server-side only (deployment §6, §7). |
| **API routing** | ✅ `/api/v1/*` served by Next.js app in P1; additive Phase 2 endpoints — matches architecture §D, §H. |
| **Authentication** | ✅ P1 public reads unauth; app→WP app-password server-side; P1 writes anti-abuse; P2 Bearer + roles — matches domain §13, deployment §6. |
| **Public/private data** | ✅ `plotReference`, verification evidence, PII, live broker location all excluded from public responses — matches domain §13, brief §33. |
| **Phase 1 vs Phase 2** | ✅ Leads (not assignments) in P1; search decoupled from the assignment engine; Postgres additive — matches architecture §E, domain §6. |

**No contradictions found.** One clarification was *introduced* (not a conflict): the
public discovery API is **property-centric with embedded offers**, and `/search` — not a
separate `/properties` collection — is the canonical collection. This is consistent with
“canonical URL per property” (domain model §12) and “`/api/v1/search` is the stable seam”
(architecture §7).

---

## Open/unresolved decisions (for review)

1. **`PropertyDetail.description`** composition — marketing copy (listing) vs. a factual
   asset summary (property): confirm whether detail shows the **primary listing's**
   description, the property summary, or both sections.
2. **`/api/v1/facets` vs build-time config** — confirm it is a live endpoint (recommended)
   or baked at build for Phase 1.
3. **Lead delivery channel** — email, WhatsApp click-to-chat, or both; and whether to keep
   a minimal lead record (and where), given it is explicitly **not** a WordPress entity.
4. **CAPTCHA provider** for `/leads` anti-abuse (vs honeypot + rate limit alone initially).
5. **`schema.org` type** for structured data — `Residence` vs `Product`+`Offer` vs
   `RealEstateListing`; confirm the primary type for best rich-result coverage.

*Awaiting collective approval of ARCHITECTURE.md + DEPLOYMENT.md + PHASE-1-DOMAIN-MODEL.md
+ API-CONTRACT.md before any implementation (CPTs, endpoints, Next.js pages, auth, DNS,
deployment) begins.*
