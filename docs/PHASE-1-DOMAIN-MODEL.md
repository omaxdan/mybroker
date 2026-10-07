# mybroker — Phase 1 Domain Model (Architecture Contract)

> **Status:** Proposal for approval. **No implementation has started.**
> **Depends on:** `docs/ARCHITECTURE.md` (Option B approved).
> **Companion:** `docs/API-CONTRACT.md` (the stable `/api/v1/*` surface).
> **Date:** 2026-10-07
>
> This document locks the Phase 1 data architecture **before** any CPT, field,
> taxonomy, endpoint, or component is built. It is a contract: implementation must
> conform to it, and changes to it are reviewed, not made ad hoc.

---

## 0. Guiding rules for this model

1. **Property ≠ Listing.** The asset and the offer are separate entities, always (§2).
2. **WordPress owns content; it never owns operational state.** Fields destined for
   the Phase 2 operational database are marked and are *not* created in WordPress (§3).
3. **WordPress is not a dumping ground.** An entity becomes a CPT/taxonomy/meta/table
   only with a stated reason (§1).
4. **Identity is a domain UID, not a WordPress post ID** (§4).
5. **The frontend consumes domain objects, never raw WordPress** (see API contract).
6. **Small architecture that grows** — model for the future shape, build only Phase 1.

---

## 1. WordPress content model

### 1.1 Entity classification (with justification)

For every entity, *why* it is what it is:

| Entity | Representation | Why this and not something else |
|---|---|---|
| **Property** | **CPT `property`** | A rich content object with its own fields, media, editorial lifecycle, and permalink identity. Needs admin CRUD. Not a taxonomy (it is the thing, not a label); not meta (it is top-level). |
| **Listing** | **CPT `listing`** | A distinct content object (an *offer*) with its own price, media curation, source, and state, related to a property. Must exist independently so one property can carry many. Not meta on property (would collapse the two). |
| **Broker** | **CPT `broker`** | A public *profile* (name, photo, bio, areas). It is public content, not an account. Deliberately **not a WordPress user** (see 1.2). Not a taxonomy (brokers have structured fields + media). |
| **Firm** | **CPT `firm`** | Public profile content with its own page/SEO. Brokers belong to a firm. Not a taxonomy because it carries structured fields and media. |
| **Property type** | **Taxonomy `property_type`** | A controlled classification used for filtering/archives (residential plot, house, apartment, commercial, land…). Classification, not content. |
| **Tenure** | **Taxonomy `tenure`** | Uganda-specific land tenure (mailo, freehold, leasehold, customary/kibanja). A controlled vocabulary used for filtering. |
| **Amenity** | **Taxonomy `amenity`** | Many-to-many labels on a property (tarmac access, borewell, fenced, solar…). Classic taxonomy. |
| **Location / area** | **Taxonomy `location_area`** (hierarchical) | Region → District → Area/suburb. Hierarchical classification driving archives and filters. **Not a CPT** — it is a label, not an editorial object. |
| **Price / availability history** | **Custom table (deferred)** `{prefix}_listing_change_log` | Append-only temporal rows; querying/ordering this via post meta is wrong. Schema defined in 2.4; **not built in Phase 1** (WP revisions cover audit until queryable history is a product need). |
| **Searchable listing index** | **Custom table (deferred)** `{prefix}_listing_index` | A flat denormalized row per listing for fast filtering, built only if/when `meta_query` gets slow (see §7). **Not built in Phase 1.** |
| **Media (image/doc)** | **WordPress attachment** | Native media library + offload/CDN. Not a CPT. |
| **Walkthrough video** | **Reference (meta) to YouTube** | We never host video. Store provider + id only (§9). |
| **Client** | **External operational entity (Phase 2)** | End users; kept out of WordPress entirely to shrink attack surface and because they are operational, not content. |
| **Assignment / dispatch / viewing / meeting / availability / live location / events** | **External operational entities (Phase 2+)** | High-frequency, state-machine, integrity-critical, realtime. Explicitly **never** in WordPress (§3). |

### 1.2 Why Broker is a CPT and not a WordPress user

A broker profile is **public content**. A WordPress *user* is an **account with login
and capabilities**. Conflating them would (a) put broker records in `wp_users`,
enlarging the authentication attack surface, and (b) force every broker to be a WP login
even though in Phase 1 brokers do not log into WordPress at all. In Phase 2 a broker
authenticates against the **operational/auth service** (not WordPress), and that
operational broker record links to this public `broker` CPT by `broker_uid`. The only
WordPress *users* are company staff/admins who manage content.

### 1.3 Custom fields (post meta) — field dictionary

All meta keys are prefixed `_mybroker_` and registered via `register_post_meta` with
explicit types and `show_in_rest: false` (exposed only through our custom endpoints,
never raw). ACF (or native meta boxes) provides the admin UI; **ACF field keys and meta
keys never appear in the public API** — the API maps them to clean domain fields.

#### Property meta (the asset — intrinsic, stable facts)

| Field | Type | Notes |
|---|---|---|
| `property_uid` | string | Opaque domain id `prop_<ULID>`. Immutable. Canonical identity (§4). |
| `plot_reference` | string | Legal/plot reference (e.g. "Block 123 Plot 45"). May be private (see §13). |
| `address_text` | string | Human address/landmark description. |
| `latitude`, `longitude` | float | Static asset coordinates. Map is intent-triggered (§8). |
| `land_size`, `land_size_unit` | float, enum | acres / decimals / hectares / sqm. |
| `bedrooms`, `bathrooms`, `parking` | int | For built structures; null for bare land. |
| `road_access` | enum | tarmac / murram / none. |
| `utilities` | set | water, nwsc, grid_power, solar, sewer… (multi). |
| `property_verification_status` | enum | `unverified\|pending\|verified\|disputed` (§10). |
| `property_verification_note` | string | **Internal**, never public. |
| `property_status` | enum | `active\|archived` (asset lifecycle; distinct from publish). |
| taxonomies | — | `property_type`, `tenure`, `location_area`, `amenity`. |

Canonical media pool (cover + gallery) attaches at property level (the true photos of
the asset); listings default to it and may curate (see §9).

#### Listing meta (the offer — market-facing)

| Field | Type | Notes |
|---|---|---|
| `listing_uid` | string | Opaque domain id `lst_<ULID>`. Immutable. |
| `property_ref` | int (WP post id) | Internal link to the backing property. |
| `property_uid` | string | **Denormalized** `prop_<ULID>` — the stable cross-system key. |
| `price_amount` | int | Minor-unit-safe integer; UGX has no minor units so whole shillings. |
| `price_currency` | enum | `UGX` (default). Multi-currency ready. |
| `price_qualifier` | enum | `fixed\|negotiable\|from\|poa` (price on application). |
| `availability_status` | enum | `available\|under_offer\|reserved\|taken\|withdrawn` (§ below). |
| `listing_source_type` | enum | `company\|broker\|firm` (Phase 1 always `company`). |
| `listing_source_ref` | string | `broker_uid`/`firm_uid` when not company. Null in Phase 1. |
| `cover_media_id` | int | Attachment id (defaults to property cover). |
| `gallery_media_ids` | int[] | Ordered attachment ids (defaults to property gallery). |
| `video_provider` | enum | `youtube` (only, Phase 1) / null. |
| `video_id` | string | YouTube video id (parsed from pasted URL). |
| `video_poster_media_id` | int | Optional custom poster; else YouTube thumbnail (§9). |
| `document_media_ids` | int[] | Floor plans / public docs. Private docs excluded (§13). |
| `listing_verification_status` | enum | `unverified\|pending\|verified` (§10). |
| `is_premium` | bool | **Future** (Phase 5). Present in model, always `false` in Phase 1. |
| `premium_rank` | int | **Future**. Null in Phase 1. |

#### Broker meta (public profile)

| Field | Type | Notes |
|---|---|---|
| `broker_uid` | string | `brk_<ULID>`. The link key to the Phase 2 operational broker. |
| `firm_ref` / `firm_uid` | int / string | Belongs-to firm. |
| `public_phone` | string | Optional, display-consented contact. Not client PII. |
| `service_area_terms` | term ids | `location_area` terms the broker covers (Phase 1 static). |
| `broker_verification_status` | enum | `unverified\|pending_review\|verified\|suspended\|rejected` (§10, brief §14). |

> **Boundary note:** broker *availability*, *live location*, *workload*, *performance*,
> and *dispatch* are **operational (Phase 2+) and are not fields here** (§3).

#### Firm meta

`firm_uid` (`frm_<ULID>`), `public_phone`, `registration_ref` (optional, may be private),
`firm_verification_status` (`unverified|pending|verified`).

### 1.4 Relationships

```
firm (1) ───< broker (N)
property (1) ───< listing (N)          # one asset, many offers
listing (N) >─── source: company | broker | firm
property ──< location_area, property_type, tenure, amenity (taxonomies)
```

Stored as post-id references **plus a denormalized UID** on the owning side, so the
stable cross-system key (`property_uid`, `broker_uid`, `firm_uid`) is always available
without a second lookup and survives re-platforming.

### 1.5 Publication states vs. domain states (kept separate)

Two independent axes — conflating them is a common bug we avoid:

- **Publication (WordPress post_status):** `draft → pending → publish`; `publish →
  draft/private` to unpublish; `trash` to remove. Governs *visibility*.
- **Availability (`availability_status`, listing meta):** `available | under_offer |
  reserved | taken | withdrawn`. Governs *market state*. A listing can be **published**
  yet **under_offer**.
- **Verification (three separate fields, §10).** Governs *trust*, independent of both.
- **Property lifecycle (`property_status`):** `active | archived`. The asset exists
  regardless of whether any listing is currently published.

### 1.6 Slugs

- Each **property** has a slug (SEO-facing, mutable). The canonical public detail URL is
  the **property's** (§12). Changing a slug 301-redirects the old one.
- Listings have WP slugs for admin only; they are **not** the public identity in Phase 1
  (listings render inside the property page). A shareable per-offer permalink can be
  added later, `rel=canonical` to the property.
- **Slugs are never identity** — identity is the UID (§4).

---

## 2. Property and Listing kept separate (mandatory)

### 2.1 Definitions

- **Property** = the underlying **real-world asset** (the land/building): physical
  facts, location, plot reference, canonical media, property verification. Stable.
- **Listing** = a **market-facing offer** for that property: price, availability,
  marketing copy, curated media, source (who offers it), listing verification, premium.

### 2.2 Why we do not collapse them even though Phase 1 is 1:1

Phase 1 has one company listing per property, so it is *tempting* to merge them. We do
not, because the future shapes below must arrive **without a data rewrite or URL/SEO
change**:

| Future need | How the split already supports it |
|---|---|
| Multiple listings per property | `property (1) ─< listing (N)` already holds. |
| Broker listings / firm listings | `listing_source_type/ref` already models source. |
| Premium listings | `is_premium`/`premium_rank` already on the listing. |
| Listing history | Listing is a first-class row; change log keyed by `listing_uid` (2.4). |
| Price changes | Price lives on the listing; history table keyed by `listing_uid`. |
| Availability changes | `availability_status` on the listing; logged like price. |
| Ownership/source distinctions | `listing_source_type/ref` + firm/broker links. |

Because the **canonical public URL is the property's** and offers render inside it,
going from one offer to many is additive (more listing rows under the same property),
not a restructuring.

### 2.3 Which fields live where (the dividing line)

- **Property:** type, tenure, location/coords, plot reference, land size, beds/baths/
  parking, road access, utilities, amenities, canonical media, property verification.
- **Listing:** price (+ currency/qualifier), availability, marketing title/description,
  curated cover/gallery/video/documents, source, listing verification, premium.

Rule of thumb: *if it is true of the asset no matter who sells it → Property; if it is
part of a specific offer → Listing.*

### 2.4 Deferred: `{prefix}_listing_change_log` (designated growth path, not built now)

```
listing_change_log(
  id            BIGINT PK,
  listing_uid   VARCHAR(40)  INDEX,   -- stable key, survives move to Postgres
  field         ENUM('price','availability','verification','source','premium'),
  old_value     TEXT,
  new_value     TEXT,
  changed_by    VARCHAR(64),          -- staff id
  changed_at    DATETIME INDEX
)
```

Phase 1 relies on current-value meta + native WP revisions. This table is built only
when queryable history (e.g. price trends) becomes a product need. Keyed by
`listing_uid`, so it migrates to Postgres cleanly.

---

## 3. WordPress does NOT own future operational state (explicit boundary)

| Owned by WordPress (Phase 1) | Owned by operational PostgreSQL (Phase 2+) |
|---|---|
| Property identity & physical facts | Assignment (+ all fields) |
| Listing content, price, availability, source | Assignment **state transitions** |
| Photographs, floor plans, public documents | Broker availability / online status |
| YouTube video reference | **Live broker location** (ephemeral) |
| Descriptions, marketing copy, SEO metadata | Dispatch / matching / ranking |
| Property characteristics & amenities | Active viewing / meeting |
| **Static** location (coords, area) | Realtime events |
| **Public** verification badges (property/listing/broker) | Operational audit log |
| | Broker workload / performance metrics |
| | Client identity & PII |
| | User location captures |

**None of the right-column fields are created in WordPress — not as meta, not as a CPT,
not as a custom table.** The handoff of **broker verification** from WordPress (Phase 1
author) to Postgres (Phase 2 system of record) is the single owner change, and it is
one-directional and explicit (see `ARCHITECTURE.md` §H).

---

## 4. Identifier strategy

### 4.1 Decision

- **Canonical identity = an opaque, type-prefixed domain UID**, generated at creation
  and stored as meta: `prop_<ULID>`, `lst_<ULID>`, `brk_<ULID>`, `frm_<ULID>`.
  - **ULID** (not UUIDv4) because it is lexicographically sortable (creation-ordered),
    URL-safe, and compact; UUIDv4 is an acceptable substitute if a library is preferred.
  - The **type prefix** makes IDs self-describing and prevents cross-entity mixups.
- **WordPress post ID** is **internal only**. It is used server-side to resolve a UID to
  a post (via an indexed meta lookup on `*_uid`), and is **never emitted in the public
  API or URLs**.
- **Slug** is SEO-facing and **mutable** → it is *not* identity; it resolves to a UID,
  and old slugs 301 to the current one.

### 4.2 Why this survives the introduction of PostgreSQL

The domain UID is generated and owned at the **application layer’s conceptual model**,
not by WordPress's auto-increment. When Postgres arrives:

- Operational rows reference content by its UID: `assignment.matched_listing_uid =
  'lst_01J...'`, `broker_operational.broker_uid = 'brk_01J...'`.
- If content is ever migrated off WordPress, the UID travels with the record, so every
  operational and external reference stays valid. Had we exposed `post_id`, every
  reference would break on migration.
- The public API (`/api/v1/*`) only ever exposes the UID + slug, so the frontend, saved
  links, and external integrations are **never coupled to WordPress internals**.

### 4.3 Resolution rules

- API/URL in → resolve **UID or slug** → internal post. Slug resolution 301s stale slugs.
- Never accept or return a raw `post_id` on the public surface.

---

## 5. (API contract — see `docs/API-CONTRACT.md`)

The stable `/api/v1/*` surface, request/response schemas, pagination, errors, versioning,
and the WordPress-decoupling guarantees are specified in the companion document so the
contract can be reviewed and versioned on its own.

---

## 6. Phase 2 compatibility (per entity)

Every Phase 1 entity exposes a **stable UID** that Phase 2 operational records reference.
Nothing in the public property platform is rebuilt.

```
WordPress (content)                 PostgreSQL (operational, Phase 2+)
──────────────────                  ─────────────────────────────────
property  (property_uid) ───────▶   viewing.property_uid
listing   (listing_uid)  ───────▶   assignment.matched_listing_uid
broker    (broker_uid)   ───────▶   broker_operational.broker_uid
                                     dispatch.broker_uid
                                     (availability, location, workload)
firm      (firm_uid)     ───────▶   (firm operational metrics, later)
```

```
Client ─creates▶ Assignment ─matching▶ Dispatch ─offer▶ Broker
                     │                                      │
                     │ references stable UIDs               │ links to WP broker_uid
                     ▼                                      ▼
            matched_listing_uid ───────────────▶ WordPress Listing (unchanged)
```

- **Added, not rewritten:** Phase 2 introduces `/api/v1/assignments`,
  `/api/v1/brokers/{uid}/availability`, etc. The property/listing/search endpoints are
  untouched, so property pages, search UX, listing URLs, SEO, media, and the frontend
  architecture **do not change**.
- **Broker verification handoff:** Phase 2 Postgres becomes system of record for broker
  verification; WordPress then displays a value read via the app, not authored locally.

---

## 7. Search architecture (and its migration path)

### Phase 1 (build this)

WordPress-backed: a custom endpoint (`/api/v1/search`, see API contract) translates
domain filters → `WP_Query` with `tax_query` (location_area, property_type, tenure,
amenity) + `meta_query` (price range, beds, baths, land size, availability, verified).
Adequate and cheap for controlled, low-volume inventory.

Filters: location/area, price range, property type, land-size range, bedrooms,
bathrooms, tenure, amenities, availability, verified status.

### Documented migration path (do NOT build now)

1. **Denormalized index table in MySQL** — `{prefix}_listing_index`, one flat row per
   published listing (price, beds, baths, size, lat/lng, type_id, area_id, status,
   verified, premium), maintained on save. Queries hit flat columns, bypassing the
   `wp_postmeta` self-joins that make `meta_query` slow. First optimization; stays inside
   WordPress/MySQL; cheap.
2. **PostgreSQL read model** — once Postgres exists (Phase 2), project listings into a
   Postgres table and serve `/api/v1/search` from it, with **PostGIS** for radius/nearby.
   The endpoint is unchanged, so the frontend never notices.
3. **External search engine** — only if full-text relevance, typo tolerance, or facet
   scale demands it: **Meilisearch** (cheap self-host) preferred over Algolia/Elastic on
   cost. **Explicitly deferred**; not introduced at inception (final principle).

`/api/v1/search` is the stable seam — its backing store can change invisibly.

---

## 8. Location architecture

| Location kind | Where | Rules |
|---|---|---|
| **Static property location** | Property (`latitude`, `longitude`, `address_text`, `location_area` terms) in WordPress | Authored once; read-only to the app; returned in the payload. |
| **Broker operational location** | Operational system (Phase 2+) | Ephemeral, short-TTL, **never WordPress**, **never public** (§13, brief §33). |
| **User location** | Collected only on explicit action (e.g. "near me" / assignment), with consent | Not collected on browse/search by default. |

**Maps remain intent-triggered.** Coordinates are present in the payload
(`location.has_map: true`), but no map provider is called on list/search/detail render —
the interactive map loads only on an explicit "View Map" click; "Explore Nearby" is a
further explicit action. No automatic map loading anywhere (brief §10–§11).

---

## 9. Media architecture

| Concern | Decision |
|---|---|
| Image storage | WordPress media library (attachments), **offloaded to object storage (Cloudflare R2, zero egress) + CDN**. API returns absolute CDN URLs. |
| Property ↔ media | Canonical cover + gallery attach at **property** level (true photos of the asset). |
| Listing ↔ media | Listing **curates** its own `cover_media_id` / `gallery_media_ids`, defaulting to the property's. Supports distinct broker/firm listing media later. |
| Responsive sizes | Register sizes: `card` (~640w), `gallery` (~1024w), `full` (~1600w), plus a tiny blurred **LQIP** placeholder. API returns a `srcset`-style variant map. |
| Cover image | Single designated attachment; falls back to first gallery image. |
| Gallery | Ordered attachment list. |
| Floor plans / public documents | Attachments flagged by role; **private documents (e.g. title deeds) are not placed in the public gallery** (§13). |
| Walkthrough video | **YouTube only.** Store `video_provider` + `video_id` (parsed from pasted URL). **Never store video locally.** |
| Video thumbnail/poster | Custom `video_poster_media_id` if set, else YouTube thumbnail. |
| Playback | **Lazy facade** (click-to-load): poster image first; YouTube iframe/JS loads only on click — no third-party JS on initial render. |
| Future CDN | CDN domain in front of object storage; API already emits CDN-absolute URLs so the origin can change without frontend changes. |

---

## 10. Verification architecture (three separate models)

**No single generic `verified=true`.** Three independent fields with distinct meaning:

| Verification | Field (owner) | States | Means |
|---|---|---|---|
| **Property** | `property_verification_status` (Property, WP) | `unverified\|pending\|verified\|disputed` | Asset existence, plot reference, physical facts checked. |
| **Listing** | `listing_verification_status` (Listing, WP) | `unverified\|pending\|verified` | The offer is legitimate — price/availability/right-to-list checked. |
| **Broker** | `broker_verification_status` (Broker, WP → Postgres from P2) | `unverified\|pending_review\|verified\|suspended\|rejected` | Broker identity/trust (brief §14). |

- A listing can be verified while its broker is only `pending_review`, etc. — they are
  orthogonal, surfaced as distinct badges.
- **Internal verification notes and documents are never public** (§13).
- Phase 1: company-controlled, so most records are `verified`, but all states exist in
  the model from the start. Broker verification's system-of-record moves to Postgres in
  Phase 2 (§3, §6).

---

## 11. Admin workflow (must be usable by a non-developer)

Delivered via CPT admin screens + ACF field groups + customized list columns. A staff
member never edits code or touches raw meta keys.

1. **Create a property** — *Properties → Add New*: internal title; pick `property_type`,
   `tenure`, `location_area`; drop a pin on a **map-picker field** (sets lat/lng); enter
   plot reference, address, land size + unit, beds/baths/parking, road access, utilities,
   amenities; upload canonical photos to the gallery field; Save (status `active`).
2. **Create a listing** — *Listings → Add New*: select the **Property** (relationship
   field) → facts inherited; enter market title + marketing description; set price +
   currency + qualifier; set `availability_status`; source defaults to **Company**.
3. **Upload media** — drag-drop into the gallery field; set the cover; listing gallery
   defaults to the property's and can be curated.
4. **Add video** — paste a YouTube URL into the video field; the id is parsed
   automatically; optional custom poster.
5. **Enter location** — on the property, via the map-picker + area taxonomy (step 1).
6. **Verify information** — set property/listing verification dropdowns; **capability-
   gated** to a `verifier` role (see §13).
7. **Publish** — set the listing to **Publish**.
8. **Unpublish** — set the listing to **Draft** or **Private**.
9. **Mark unavailable** — change `availability_status` (e.g. `under_offer`, `taken`,
   `withdrawn`) — **independent of publish state**.
10. **Update price** — edit the price field; the change is captured by WP revisions (and,
    once built, the change-log table).

Admin list columns surface price, availability, verification, and area at a glance.
Capability roles: `administrator`, `content_editor` (create/edit/publish listings &
properties), `verifier` (set verification), `read-only` staff.

---

## 12. SEO & canonical URL structure

### Decision

- **Area browse (indexable archives):**
  `/properties/<region>/<district>/<area>/`
  e.g. `/properties/central/wakiso/kira/`
- **Canonical detail page (the PROPERTY):**
  `/property/<area>/<property-slug>/`
  e.g. `/property/kira/3-bedroom-home-munyonyo-rd/`
- **Search (filtered, not primary SEO target):** `/properties/<area>/?type=…&min=…`
  with canonical pointing at the clean area archive.

### Why the detail URL is the property's, not the listing's

This is the decision that keeps URLs/SEO stable across the whole roadmap. In Phase 1 the
single company offer renders on the property page. When multiple/broker/premium listings
arrive (Phase 5), they appear as **offers within the same property URL** — so:

- listing URLs never become the canonical identity,
- the property page keeps its accumulated SEO authority,
- no redirects or re-indexing are needed when offers multiply.

A specific offer may later get a shareable permalink (`…/offers/<listing-slug>`) that is
`rel=canonical` to the property page, so it never competes for ranking.

- Slugs are mutable; identity is the UID; stale slugs **301** to the current URL.
- `<area>` in the path is the *primary* area term; re-tagging area 301-redirects.

---

## 13. Security — public vs. private

| Surface | Public | Private / protected |
|---|---|---|
| **WordPress admin** | — | Entire `/wp-admin`; behind 2FA + login throttling; ideally not on the public listing domain. |
| **WordPress REST** | Only our `mybroker/v1` **read** endpoints (published content) | Default `wp/v2/*` routes locked to authenticated requests; user enumeration disabled; XML-RPC disabled. |
| **Property/Listing data** | Published listings' public fields (title, description, price, media, location summary + coords, public verification badges) | `*_verification_note`, internal admin notes, exact legal/plot documents. |
| **Media** | Public images/floor plans via CDN | Private documents (e.g. title deeds) not placed in public gallery; access-controlled if stored. |
| **Broker info** | Public profile (name, photo, bio, areas, public verification badge, display phone if consented) | Operational data (availability, location, workload), internal verification evidence. |
| **Client info** | — | All client identity/PII (operational, Phase 2) — never in WordPress, never public. |
| **Write access** | None from the browser in Phase 1, except the rate-limited, honeypot-protected "ask a broker" lead form (delivered to the company inbox/WhatsApp — **not** a public WP write) | All content writes are server-side with an application credential; browsers never call WordPress. |

**Rule:** WordPress administrative structures (post IDs, meta keys, ACF keys, `wp/v2`
routes, `wp_users`) are **never** exposed through the public API (enforced by the
contract in `docs/API-CONTRACT.md`).

---

## Open questions for review

1. **ULID vs UUIDv4** for the domain UID — recommend ULID; confirm no library constraint.
2. **ACF vs native meta boxes** — recommend ACF (incl. map-picker, relationship, repeater
   fields) for admin usability; confirm licensing is acceptable (ACF free covers most;
   map-picker/repeater may want ACF Pro).
3. **Object storage/CDN provider** — recommend Cloudflare R2 (zero egress) + CDN; confirm.
4. **Tenure vocabulary** — confirm the Uganda-specific set (mailo, freehold, leasehold,
   customary/kibanja) before we seed the taxonomy.

*Awaiting approval of this model and the companion API contract before any CPT, field,
taxonomy, endpoint, or component is implemented.*
