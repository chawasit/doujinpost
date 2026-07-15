# Doujinpost — Technical Plan

A self-publishing platform for doujin **manga** (image sets) and **novels**
(text/EPUB), with:

- **Optimised storage** — deduplicated, content-addressed masters; modern image
  codecs; per-user copies never stored, only derived on the fly.
- **Embedded user tracking** — a forensic watermark ("traitor tracing"). **⚠️
  DEFERRED (post-v1, optional).** Designed in §3 but *not* built for launch: for a
  free community scanlation site it has little meaning and it defeats CDN caching
  — the exact image-delivery win we're chasing. Kept on the shelf for possible
  licensed/official uploads later. See `docs/nekopost-comparison.md` §6.
- **Community features** — shoutbox, title requests, creator profiles,
  gamification/badges, ranking pages, public ratings — see
  [`specs/community-features.md`](specs/community-features.md).
- **Discovery** — cross-lingual **semantic search** (hybrid lexical + LLM
  embeddings, LEANN for deep text; [`specs/search.md`](specs/search.md)) over an
  **IMDB-style entity/credit graph** with community **wiki relations**
  ([`specs/entities-and-relations.md`](specs/entities-and-relations.md)).
- **Comments + a basic webboard** (forum).
- **A DMCA takedown pipeline** — notice intake, escrow/takedown, counter-notice,
  repeat-infringer accounting.
- **Personalisation** — private reading history ("continue reading"), follows/
  library, and blended recommendations — see
  [`specs/history-follows-recommendations.md`](specs/history-follows-recommendations.md).

This document is the build plan: architecture, data model, the watermark scheme
in detail, and a phased roadmap. It is design-only — no code is committed yet.

> **Project posture — open-source, nonprofit, self-hostable.** There is **no
> monetization**: no payments, subscriptions, tips, ads, or creator payouts. The
> codebase is open-source (license TBD — see `docs/gaps-and-risks.md`), the stack
> is chosen so a community can self-host, and sustainability comes from
> **donations, grants, and volunteer labour**, not in-app commerce. This deletes
> the payment-processor and VAT problems — but it does **not** reduce the legal
> and trust-&-safety obligations (illegal-content detection, age-gating, DMCA
> safe-harbour, GDPR). A volunteer nonprofit has *less* compliance budget, so
> those must be cheap-by-design, not skipped. Hosting/egress cost — especially
> the per-user watermark (§3) — is now a direct threat to a donation-funded
> budget, which pushes watermarking toward downloads/exports only, not in-app
> reads.

> **Scope & intent.** The watermarking here is *defensive* anti-piracy /
> leak-attribution on content the platform is licensed to distribute. It stamps
> the downloading account's own identifier into their own copy. It is not a tool
> for tracking third parties or de-anonymising anyone but the account holder who
> agreed to the ToS. The DMCA system is the counterweight: uploaders can be
> wrong, and takedown/counter-notice must be first-class.

---

## 1. Architecture at a glance

```
              ┌──────────────────────────────────────────────────┐
   Browser  ─▶│  PWA — React + Vite (installable, service worker) │
              │  offline reader cache · Web Push · add-to-home    │
              └───────────────┬──────────────────────────────────┘
                              │  REST/JSON + signed URLs · CDN edge
              ┌───────────────▼──────────────────────┐
              │  API — ElysiaJS on Bun (TypeScript)   │
              │  auth (password · Google · passkey) · │
              │  catalog · comments · forum · DMCA ·  │
              │  download broker                      │
              └───┬───────────┬─────────┬─────────────┘
                  │           │         │        │ enqueue jobs
        ┌─────────▼──┐  ┌─────▼─────┐  ┌▼─────────────┐  │
        │ PostgreSQL │  │  Redis    │  │ Object store │  │
        │ (metadata) │  │ queue+cache│  │  S3 / MinIO │  │
        └────────────┘  └─────┬─────┘  │  CAS blobs   │  │
                              │        └──────────────┘  │
              ┌───────────────▼────────────┐   ┌─────────────────────────────┐
              │ Bun workers (TS)           │   │ (deferred) Python watermark │
              │ transcode (sharp/libvips)· │   │ sidecar — DCT/RS/fountain,  │
              │ dedup · notify · EPUB      │   │ only if §3 is ever enabled  │
              └────────────────────────────┘   └─────────────────────────────┘
```

**One uniform TypeScript stack (v1).** The user-facing app and API are a single
TypeScript codebase on **Bun + ElysiaJS**, shipped as an installable **PWA** —
Elysia's end-to-end types (Eden client) give the React frontend a typed API with
no codegen, and Bun keeps dev/CI fast. **All** v1 workers are Bun: transcode via
`sharp`/libvips, dedup, EPUB build, notifications, ranking/trending rollups.

The Python **watermark sidecar** shown dashed above is **deferred with §3** — it
is the only component that would break the single-language stack, and since
watermarking is off the v1 roadmap, so is it. If §3 is ever enabled for licensed
content, the DSP (mid-band DCT, Reed–Solomon, fountain/LT decode — far stronger
in NumPy/SciPy/`reedsolo` than in JS) drops in behind the same Redis job contract
without touching the rest of the system.

**Stack**

| Concern            | Choice                                   |
|--------------------|------------------------------------------|
| App shell          | **PWA** — React + Vite, service worker, Web App Manifest, Web Push |
| UI base template    | **TailAdmin** (free React edition, MIT) — Tailwind design system, dashboard shells, dark mode |
| Runtime            | **Bun**                                  |
| API framework      | **ElysiaJS** (TypeScript) + Eden typed client |
| Bun workers        | BullMQ (Redis) — transcode/dedup/notify/EPUB/ranking |
| Media/watermark    | **Deferred** — Python sidecar (NumPy, SciPy, `reedsolo`), only if §3 is enabled |
| DB                 | PostgreSQL 16 (Drizzle ORM)              |
| Cache / queue      | Redis                                    |
| Object storage     | S3-compatible (MinIO self-host, or S3)   |
| Search             | **Hybrid**: Postgres FTS (lexical) ⊕ vector (pgvector HNSW; **LEANN** for deep text) with a self-hosted multilingual embedding model — see [`specs/search.md`](specs/search.md) |
| Image transcode    | `sharp` (libvips) → AVIF/WebP            |
| Auth               | Sessions + Argon2id · Google OIDC · WebAuthn passkeys · TOTP 2FA — see [`specs/authentication.md`](specs/authentication.md) |
| Deploy             | Docker Compose → k8s; CDN in front       |

**UI base — TailAdmin.** The frontend is scaffolded from the free **TailAdmin
React** template (React + Vite + Tailwind, MIT-licensed), which gives us the
design tokens, layout shells, dark mode, and a large stock of prebuilt
components (tables, forms, modals, charts, alerts) on day one. We use it in two
tiers:

- **Admin & account surfaces — TailAdmin as-is.** Moderation queue, DMCA
  console, uploader dashboard, storage/cost analytics (Phase 6), and all
  settings/auth screens map almost directly onto TailAdmin's dashboard shell,
  data tables, and form components. This is where the template earns its keep.
- **Public reader & catalog — custom, on TailAdmin's tokens.** The manga reader,
  novel reader, and browse grid are bespoke components, but built against the
  same Tailwind theme (colours, spacing, typography, dark mode) so the whole PWA
  reads as one product rather than a dashboard bolted to a reader.

Practical notes: keep TailAdmin's config as our Tailwind base and layer a thin
theme override so upstream updates stay mergeable; strip the demo pages/data on
first commit; preserve the MIT `LICENSE`/attribution in the frontend package.

---

## 2. Storage design (the "optimised storage" requirement)

The core idea: **store one canonical master per unique blob, deduplicated
globally, and serve a shared rendition to everyone.** Because v1 does no per-user
watermarking (§2.5), those renditions are identical across users and therefore
edge-cacheable — N downloads of a work cost O(1) storage *and* mostly O(1) origin
traffic.

### 2.1 Content-addressable store (CAS)

- Every ingested file (a manga page, a novel body, a cover) is hashed with
  **BLAKE3**. The digest is the object key: `blobs/<b3[:2]>/<b3[2:4]>/<b3>`.
- Identical bytes → identical key → stored once. Two uploaders posting the same
  page, or a re-upload of the same chapter, share the blob. A refcount row
  tracks how many works reference each blob so garbage collection is safe.
- Blobs are immutable. "Editing" a work creates new blobs and repoints the
  manifest; old blobs are GC'd when refcount hits zero.

### 2.2 Manga pages

1. On ingest, decode each page, strip metadata, normalise colour profile.
2. Transcode to a **tiered set**:
   - `avif` (primary, ~50% smaller than JPEG at equal quality),
   - `webp` (fallback for older clients),
   - a small `jpeg` thumbnail and a blurhash placeholder for grids.
3. **Perceptual dedup** on top of exact dedup: compute a pHash per page. Near-
   duplicate pages (re-scans, minor recompression) are flagged for the uploader
   and can be collapsed at the manifest level, but we never silently discard —
   perceptual collisions are advisory, exact BLAKE3 is authoritative.
4. Tiled/progressive encoding so the reader streams pages at the viewport
   resolution instead of pulling full-res masters.

Full pipeline (rendition ladder, responsive `srcset`, blurhash, and the reader
itself) is specified in
[`specs/reader-and-image-delivery.md`](specs/reader-and-image-delivery.md) —
this is the core service-health win over Nekopost.

### 2.3 Novels

- Normalise to a canonical internal form (sanitised XHTML chapters + a
  `manifest.json`), then store **zstd-compressed** (`--long` window; text
  compresses 3–5×). EPUB export is generated on demand from the canonical form.
- Chapter-level blobs so revisions and shared boilerplate dedup naturally.

### 2.4 Tiering & lifecycle

- **Hot** (recent/popular): CDN + fast object tier.
- **Cold**: works untouched for N months migrate to a cheaper/archive tier;
  first read after that pays a rehydrate latency (surfaced in the UI).
- Derived renditions (thumbnails, AVIF/WebP pages, EPUBs) are cache, not
  source of truth — evictable and reproducible from the master + manifest.

### 2.5 Delivery path (watermark deferred → CDN-cacheable)

**v1 default: no per-user watermarking.** Reads and downloads serve the **shared,
deduplicated rendition** straight from cache. Images use **content-addressed,
immutable URLs** cached forever at the edge; access to *restricted* content is
gated by **signed cookies** (not per-request signed URLs, which would bust the
cache), so the cache key stays the content hash and one edge object serves all
entitled users. Identical bytes per user → **~100% cache-hit ratio once warm** —
the core service-health win over Nekopost (whose slow image loading is its #1
complaint). No Python DSP sidecar on the read path. Details:
[`specs/reader-and-image-delivery.md`](specs/reader-and-image-delivery.md) §A.2.

> If watermarking is later enabled for licensed/official uploads (§3), it applies
> only to that opt-in subset and only on *downloads/exports*, never in-app reads —
> so the cache-defeating cost stays off the hot path. The old "dedup vs.
> per-user copy" tension only returns for that subset.

---

## 3. Embedded user tracking — the *chained, compression-resilient* watermark

> **⚠️ STATUS: DEFERRED — post-v1, optional.** This section is a preserved design,
> **not** part of the launch build. It is not on the v1 roadmap (§8). Rationale in
> `docs/nekopost-comparison.md` §6: for free scanlation it adds little and its
> per-user renders defeat CDN caching. Revisit only if the platform hosts
> licensed/official content that warrants leak attribution, and even then apply it
> to downloads/exports of that subset only. Everything below stands as the design
> for that future case.

**Goal.** Given a leaked file (or a fragment of one), recover the account id that
downloaded it, even after the pirate has re-saved/re-compressed the images,
cropped pages, dropped pages, or reflowed the novel text.

**Threat model we design against**

- JPEG/AVIF/WebP **re-compression** at moderate quality.
- **Cropping** and mild rescaling of images.
- **Page subsetting** — leaker posts only chapters 2–4.
- **Screenshotting** at reader resolution (degrades but shouldn't fully defeat).
- For novels: format conversion, whitespace normalisation, partial copy-paste.

**Non-goals.** Surviving a determined adversary who OCRs every page and retypes
the novel, or who runs aggressive denoise + heavy downscale on every image. No
watermark survives arbitrary re-authoring; the design targets *casual-to-
moderate* leakage, which is the overwhelming majority.

### 3.1 The payload and why it is "chained"

The raw secret is small: a per-download **tracking token** = `HMAC_k(user_id ‖
work_id ‖ issued_at)` truncated to, say, 64 bits, stored server-side in a
`download_grant` row so it maps back to the exact account and session. We embed
the *token*, never the raw user id, so the mark carries no PII and is revocable.

We then make the embedded bits robust with a **chain of erasure-coded shards**:

1. **Error-correct**: encode the 64-bit token with a **Reed–Solomon** code into
   an ~N-symbol codeword that tolerates a fixed fraction of symbol
   loss/corruption (RS handles the "flipped bits" from recompression noise).
2. **Fountain / chain across carriers**: split the RS codeword into shards and
   spread them with a **rateless (LT/fountain) code** so the payload is
   recoverable from *any sufficiently large subset* of carriers — this is the
   "chained" property. Each page (manga) or each text block (novel) carries one
   or more shards plus a small header `(work_id_tag, shard_index, chain_seq)`.
   Because it's rateless, recovering the token needs only *K of the produced
   shards*, so:
   - crop one page → its shards are lost, others still decode;
   - keep only chapters 2–4 → the surviving pages still carry ≥ K shards;
   - recompress everything → RS mops up the per-shard bit errors.
3. **Repeat/interleave**: the whole chain repeats every M carriers, so even a
   short contiguous fragment tends to contain a full recoverable generation.

Result: attribution degrades gracefully instead of failing at the first edit.
A tuning knob (`redundancy_factor`) trades file-size/quality overhead against
how much mutilation the mark survives; picked per content tier.

### 3.2 Carrier — manga images (survives recompression)

Embed in the **frequency domain**, not raw pixels, because JPEG/AVIF quantise
DCT coefficients — a spatial (pixel) watermark is destroyed by the first
re-save, a mid-band DCT watermark rides along inside it.

- Work in luminance (Y) of YCbCr; split into 8×8 blocks aligned to the JPEG
  grid.
- For each block carrying a shard bit, apply a **spread-spectrum** nudge to a
  set of **mid-frequency DCT coefficients** (low freq = visible, high freq =
  killed by compression; mid = the sweet spot). The bit is encoded as a sign/
  magnitude pattern correlated against a secret pseudo-random sequence keyed per
  work — a **blind detector** recovers it by correlation without the original.
- Strength is **perceptually masked** (JND model): more energy where texture
  hides it, less in flat gradients, so the mark stays invisible at normal
  viewing.
- Detection: run the correlator over the suspect image's DCT blocks, majority-
  vote per shard bit, feed soft-decision values into the RS+fountain decoder.
- Cropping resilience comes from redundancy (many blocks per shard) plus a
  low-frequency **synchronisation template** to re-register a cropped/rescaled
  image before correlating.

### 3.3 Carrier — novels (survives format conversion & copy-paste)

Text has no DCT, so we use **linguistic/format steganography** layered for
redundancy:

- **Zero-width & homoglyph coding** (primary, invisible): encode shard bits in
  choices of zero-width joiners and confusable Unicode codepoints at controlled
  positions. Survives HTML→EPUB→TXT if the pipeline preserves codepoints;
  fragile to aggressive normalisation, so it is *not* the only layer.
- **Whitespace / typographic coding**: spacing, hyphenation, and punctuation-
  variant choices carry additional shards.
- **Canonical-text fingerprint** (fallback, survives even full de-formatting):
  we also store, server-side, a **SimHash / winnowing fingerprint** of the
  canonical text per work. A recovered plain-text leak with *all* stego stripped
  can still be matched to the work (not the user) by fingerprint — which at
  least scopes the investigation. User-level attribution for novels therefore
  degrades to work-level under maximal stripping, and we say so honestly.

Each text block gets header `(work_id_tag, shard_index, chain_seq)` mirroring
the manga chain, so the same RS+fountain decoder is reused.

### 3.4 The download broker (where embedding happens)

```
GET /d/<work>/<rendition>   (authenticated, rate-limited)
  1. authz: entitlement + not-DMCA'd + not repeat-abuser
  2. mint download_grant → token = HMAC(user, work, ts); persist row
  3. build fountain chain of shards from token
  4. stream master blob(s) through the watermark encoder:
       manga → per-page DCT embed
       novel → per-block stego embed
  5. deliver via short-lived signed URL; log grant id
```

- Embedding is **streaming and cache-friendly at the shard level**: the shard
  layout for a given token is deterministic, and the expensive per-page DCT
  transform of the *master* is cached; only the light per-user modulation is
  applied at request time. Popular works stay cheap.
- Every download writes an audit row (`grant_id, user_id, work_id, ip_trunc,
  ua_hash, ts`). On a leak, `detect(sample) → token → grant row → account`.
- **Detector tool** (internal, admin-only): upload a suspect image/text, it
  returns candidate `grant_id`s ranked by decoder confidence, with a clear
  "insufficient signal" verdict when the sample is too degraded — we never
  assert attribution the math doesn't support.

### 3.5 Keys, privacy, revocation

- HMAC keys and per-work watermark seeds live in a KMS/secrets store, rotated on
  a schedule; the embedded token is opaque and reversible only server-side.
- Because the mark is a token (not user id), a user's marks can be invalidated by
  dropping the grant rows; and the scheme carries no readable PII if a copy
  leaks to a third party.
- Documented in the privacy policy and ToS: what is embedded, why, retention.

---

## 4. Comments & webboard

### 4.1 Comments (attached to a work)

- Threaded (1–2 levels), markdown-lite (sanitised), reactions, spoiler tags.
- Per-work sort: newest / top. Rate-limited; new accounts throttled.
- Moderation: report → queue; soft-delete keeps thread shape; shadow-hide option.

### 4.2 Webboard (standalone forum)

- **Boards → threads → posts**. Boards for discussion, requests, translation
  groups, site meta. Pinned/locked threads; per-board mod roles.
- Post editor shares the sanitiser and rate-limiter with comments.
- Search over posts via Postgres FTS.
- Anti-abuse: Argon2 login, email verify, per-IP + per-account rate limits,
  optional captcha on signup and on burst posting, a spam heuristic queue.

Both systems share one `content_item` + `moderation_action` core so reporting,
hiding, and audit logging are written once.

---

## 5. DMCA system

A real notice-and-takedown workflow, not a mailto link.

**Data**: `dmca_notice` (claimant, target work/blob, sworn statements,
timestamps, state), `dmca_counter_notice`, `strike` (per uploader), and an
immutable `dmca_event` audit log.

**Flow**

1. **Intake** — public form + a designated-agent email inbox. Captures the
   statutory elements (identification of work, good-faith + accuracy +
   authority-under-penalty-of-perjury statements, signature, contact).
   Incomplete notices are rejected with a reason, not silently dropped.
2. **Action** — on a valid notice the target work is **disabled** (hidden,
   downloads blocked) fast; blob stays in escrow (not hard-deleted) pending
   resolution. Uploader and claimant both notified.
3. **Counter-notice** — uploader can file a counter-notice with the statutory
   statements; after the statutory waiting window with no claimant lawsuit
   notice, content may be **restored**.
4. **Repeat-infringer policy** — strikes accrue per uploader; a threshold
   triggers account termination. This policy is required to keep safe-harbour
   and is enforced by the `strike` accounting, surfaced to admins.
5. **Transparency** — redacted notices logged; every state change is in the
   append-only `dmca_event` log for auditability.

**Interaction with watermarking**: a leak traced by the detector can seed a
takedown against the *reposter's* host, while the DMCA system handles inbound
claims against *our* users — the two are separate lanes and must not be
conflated in the UI.

---

## 6. Data model (core tables)

Roles, grants, and the `Category → Series → [Chapter · Notice · Extra]`
hierarchy are specified in full in
[`specs/roles-and-content-model.md`](specs/roles-and-content-model.md); the core
tables:

```
users(id, handle, email, pw_hash, totp_secret, state, created_at)
roles(user_id, role[reader|writer|moderator|admin|publisher_rep], scope_publisher_id, granted_by, granted_at)
content_grants(series_id, user_id, grant[owner|coauthor|translator])

categories(id, kind[type|genre], slug, title, description, parent_id,   -- shallow tree: category > sub-category
           cover_blob, backing_tag_id, sort_order, is_listed, created_at)
series_categories(series_id, category_id, primary_flag)                 -- many-to-many, one primary
tags(id, namespace, name, description, state, is_restricted, usage_count, created_by, created_at)
tag_aliases(alias, tag_id)                          -- synonyms/typos -> canonical
tag_implications(tag_id, implies_tag_id)            -- auto-add entailed tags
series_tags(series_id, tag_id, added_by, created_at)
series(id, type[manga|novel], title, origin[original|translation], source_ref,
       language, status[ongoing|complete|hiatus|licensed], state, rating,
       reading_mode[paged_ltr|paged_rtl|vertical],  -- drives which reader loads (webtoon = vertical)
       tag_ids[],                                   -- denormalised, GIN-indexed for filtering
       publisher_id, created_by, created_at)
chapters(id, series_id, number, title, kind, is_extra, extra_type,
         state[draft|pending|published|hidden|dmca_disabled|licensed_removed|deleted], release_at)
notices(id, series_id, author_id, type, body, pinned, state, created_at)

blobs(b3_hash PK, size, mime, tier, refcount, created_at)
manifest_entries(series_id, chapter_id, page_index, blob_hash, rendition)
phash_index(blob_hash, phash)                      -- perceptual dedup
download_grants(id, user_id, series_id, token, issued_at, ip_trunc, ua_hash)
watermark_seeds(series_id, seed, algo_version)     -- secret, in KMS ref

-- personalisation — see specs/history-follows-recommendations.md
read_progress(user_id, series_id, chapter_id, position, percent, updated_at)  -- "continue reading"
read_events(user_id, series_id, chapter_id, event, dwell_ms, ts)              -- append-only, TTL-trimmed
library(user_id, series_id, shelf, rating, is_private, updated_at)            -- reading|planned|completed|on_hold|dropped|favorite
follows(user_id, target_type[series|creator|user|tag], target_id, notify[], created_at)
notifications(id, user_id, type, subject_ref, read_at, created_at)
user_taste(user_id, tag_id, weight, updated_at)                              -- content-based profile
item_similarity(series_id, other_series_id, score, method, computed_at)      -- batch collaborative
trending(scope, series_id, score, window, computed_at)

-- community — see specs/community-features.md
profiles(user_id, bio, links, languages[], open_to_collab, banner_blob, updated_at)
groups(id, slug, name, about, created_by, created_at)
group_members(group_id, user_id, role)
series_credits(series_id, creditee_id, credit)      -- translator|cleaner|typesetter|writer|artist
shout_messages(id, room, user_id, body, created_at, deleted_by)
shout_mutes(room, user_id, until, by)
title_requests(id, requester_id, title, source_url, language, description, state, claimed_by, fulfilled_series_id, upvotes, created_at)
request_votes(request_id, user_id)                  -- unique
badges(id, slug, name, description, icon, criteria, is_admin_only)
user_badges(user_id, badge_id, awarded_at, context)
points_ledger(id, user_id, delta, reason, ref, settled, created_at)   -- append-only
levels(user_id, xp, level, updated_at)
rankings(surface, scope, series_id, rank, score, window, computed_at)
ratings(series_id, user_id, score, created_at)      -- unique, public
series_rating_agg(series_id, avg, count, updated_at)
reactions(target_type[chapter|comment], target_id, user_id, emoji, created_at)

comments(id, series_id, author_id, parent_id, body, state, created_at)
boards(id, slug, title, mod_roles)
threads(id, board_id, title, author_id, locked, pinned, created_at)
posts(id, thread_id, author_id, body, state, created_at)
reports(id, target_type, target_id, reporter_id, reason, state)
moderation_actions(id, actor_id, target_type, target_id, action, note, ts)

-- entities, relations & wiki — see specs/entities-and-relations.md
entities(id, type[person|circle|company|magazine|brand|character], name, alt_names[], bio, links, image_blob, linked_user_id, state, merged_into, created_at)
entity_credits(series_id, entity_id, role, note)     -- story_writer|illustrator|author|original_creator|...
series_relations(from_series, to_series, relation)   -- sequel|spin_off|adaptation_of|shares_universe|... (auto-inverse)
entity_relations(from_entity, to_entity, relation)   -- alias_of|member_of|imprint_of|collab_with
wiki_revisions(id, target_type, target_id, editor_id, diff, state, note, created_at)  -- append-only history
-- search index — see specs/search.md
embeddings(doc_type, doc_id, vector, model_version)  -- pgvector (series/entity); LEANN for deep text

publishers(id, name, verified_at, verified_by)
rights_claims(id, publisher_id, series_id, state, evidence, decided_by, decided_at)
dmca_notices(id, claimant, target_series_id, statements, state, ts)
dmca_counter_notices(id, notice_id, uploader_id, statements, ts)
strikes(id, uploader_id, notice_id, ts)
dmca_events(id, notice_id, event, actor, ts)       -- append-only
audit_log(id, actor, action, subject, meta, ts)    -- append-only
```

Extras are chapters with `is_extra = true` (reusing the whole upload/storage/
reader pipeline); Notices are lightweight text.

---

## 7. Security, abuse, and legal posture

- **Auth**: three sign-in methods — password (Argon2id), **Google Sign-In**
  (OIDC), and **passkeys** (WebAuthn/FIDO2) — over server-side sessions
  (HttpOnly/SameSite cookies), with optional TOTP 2FA, login throttling, and
  breach-password checks. Full design in [`specs/authentication.md`](specs/authentication.md).
- **Contact verification**: email + SMS control-of-channel checks gate uploading,
  posting, and downloading via a `verification_level` — see
  [`specs/user-verification.md`](specs/user-verification.md).
- **Uploads**: strict MIME/magic sniffing, image bomb limits, metadata strip,
  virus scan hook, per-account quotas.
- **Access**: entitlement + rating/age checks mint short-TTL **signed URLs**; no
  hotlinking of masters (only cached renditions leave the perimeter). Identical
  bytes per user → edge-cacheable (§2.5).
- **Rate limiting** everywhere user-generated content or downloads occur.
- **Age/rating gating** for adult doujin categories, per jurisdiction config.
- **Legal**: DMCA designated agent registered; ToS discloses data retention;
  clear repeat-infringer policy for safe-harbour. (Watermark disclosure only
  becomes relevant if §3 is ever enabled.)

---

## 8. Phased roadmap

Watermarking (old Phases 2–3) is **removed from the v1 critical path** and
delivery is plain signed-URL + CDN cache (§2.5). The reorder puts the
Nekopost-parity community texture and the image-delivery win first.

| Phase | Deliverable |
|------:|-------------|
| **0** | Repo scaffold, Docker Compose (Postgres/Redis/MinIO), CI, auth (password/Google), migrations. |
| **1** | Upload → CAS ingest → AVIF/WebP transcode → **manga + webtoon (vertical) reader**; novel canonicalise + reader. Signed-URL delivery, **CDN edge caching**. Exact + perceptual dedup. |
| **2** | Catalog: categories/sub-cats + tags; **hybrid semantic search** (lexical ⊕ multilingual vectors, cross-lingual Thai/JP/EN + "more like this"); **entities + credits + relations** (IMDB-style); **ranking pages** (Weekly Popular / Latest / Recommended); public ratings. |
| **3** | **Community pack** — creator profiles, shoutbox, title requests, comments; reading history + library + follows/notifications. See `specs/community-features.md`. |
| **4** | **Gamification** — badges/points/levels; webboard/forum on the shared content/moderation core. |
| **5** | DMCA intake → takedown → counter-notice → strikes → transparency log; T&S/legal spine (`docs/gaps-and-risks.md`). |
| **6** | Storage tiering/lifecycle, cold-storage rehydrate, observability/backups/DR, cost dashboards. |
| **7** | Hardening, load test, moderation tooling, i18n polish, launch. |
| **later / optional** | **Watermark (§3)** — only if licensed/official uploads warrant it: DCT+RS+fountain, admin detector, and the attack-validation harness (JPEG/AVIF recompress, 10–40% crop, page-subset, stego-strip → recovered-token rate). Add **collusion-secure (Tardos) codes** at this point (`docs/gaps-and-risks.md` #6). |

---

## 9. Open questions

**Live (v1):**
1. **Jurisdiction & operator entity**: the nonprofit legal home decides DMCA
   specifics, age-gating rules, and who holds safe-harbour.
2. ~~**Search engine**: Thai/CJK tokenisation~~ — **resolved**: hybrid lexical ⊕
   multilingual-embedding vector search sidesteps word-segmentation and is
   cross-lingual (`specs/search.md`). Remaining sub-choice: which embedding model
   and CPU-vs-GPU hosting budget.
3. **Gamification economy**: what do points/badges reward, and how do we avoid
   incentivising low-quality spam uploads? (`specs/community-features.md`)
4. **Shoutbox moderation load**: real-time chat is a moderation sink — global,
   per-series, or launch read-mostly with tight rate limits?

**Deferred with the watermark (only if §3 is revived):** redundancy-vs-quality
tuning, `download_grants` retention window, novel attribution honesty, screenshot
resistance, and collusion resistance.
