# Doujinpost — Technical Plan

A self-publishing platform for doujin **manga** (image sets) and **novels**
(text/EPUB), with:

- **Optimised storage** — deduplicated, content-addressed masters; modern image
  codecs; per-user copies never stored, only derived on the fly.
- **Embedded user tracking** — a forensic watermark ("traitor tracing") stamped
  into each download so a leaked copy can be traced back to the account that
  pulled it, built to survive re-compression and cropping via a *chained,
  compression-resilient* encoding.
- **Comments + a basic webboard** (forum).
- **A DMCA takedown pipeline** — notice intake, escrow/takedown, counter-notice,
  repeat-infringer accounting.

This document is the build plan: architecture, data model, the watermark scheme
in detail, and a phased roadmap. It is design-only — no code is committed yet.

> **Scope & intent.** The watermarking here is *defensive* anti-piracy /
> leak-attribution on content the platform is licensed to distribute. It stamps
> the downloading account's own identifier into their own copy. It is not a tool
> for tracking third parties or de-anonymising anyone but the account holder who
> agreed to the ToS. The DMCA system is the counterweight: uploaders can be
> wrong, and takedown/counter-notice must be first-class.

---

## 1. Architecture at a glance

```
                    ┌────────────────────────────────────────────┐
   Browser  ───────▶│  Next.js (React) web app  ·  CDN edge cache │
                    └───────────────┬────────────────────────────┘
                                    │  REST/JSON + signed URLs
                    ┌───────────────▼────────────────┐
                    │  API — FastAPI (Python)         │
                    │  auth · catalog · comments ·    │
                    │  forum · DMCA · download broker  │
                    └───┬───────────┬─────────┬───────┘
                        │           │         │
              ┌─────────▼──┐  ┌─────▼─────┐  ┌▼──────────────┐
              │ PostgreSQL │  │  Redis    │  │ Object store  │
              │ (metadata) │  │ queue+cache│  │ S3 / MinIO   │
              └────────────┘  └─────┬─────┘  │  CAS blobs    │
                                    │        └───────────────┘
                        ┌───────────▼─────────────┐
                        │  Worker pool (RQ/Celery) │
                        │  ingest · transcode ·    │
                        │  dedup · watermark-render │
                        └──────────────────────────┘
```

**Why this split.** Image/text processing (Pillow, NumPy, SciPy, `reedsolo`,
zstd) is far more mature in Python, and the watermark math lives there; FastAPI
keeps the API in the same language as the workers. Next.js gives SSR reader
pages and a good image-reader UX. Heavy work (transcode, watermark render) is
async on a queue so uploads and downloads never block on CPU.

**Stack**

| Concern            | Choice                                   |
|--------------------|------------------------------------------|
| Frontend           | Next.js + TypeScript + Tailwind          |
| API                | Python 3.12 + FastAPI + Pydantic         |
| Workers            | RQ (Redis) or Celery                     |
| DB                 | PostgreSQL 16                            |
| Cache / queue      | Redis                                    |
| Object storage     | S3-compatible (MinIO self-host, or S3)   |
| Search             | Postgres FTS first; OpenSearch if needed |
| Auth               | Session cookies + Argon2id; TOTP 2FA     |
| Media libs         | Pillow, pillow-avif, NumPy, SciPy, reedsolo, zstandard |
| Deploy             | Docker Compose → k8s; CDN in front       |

---

## 2. Storage design (the "optimised storage" requirement)

The core idea: **store one canonical master per unique blob, deduplicated
globally; never store a per-user watermarked copy.** Watermarking happens at
*download* time on a derived stream, so N downloads of a work cost O(1) storage,
not O(N).

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

### 2.3 Novels

- Normalise to a canonical internal form (sanitised XHTML chapters + a
  `manifest.json`), then store **zstd-compressed** (`--long` window; text
  compresses 3–5×). EPUB export is generated on demand from the canonical form.
- Chapter-level blobs so revisions and shared boilerplate dedup naturally.

### 2.4 Tiering & lifecycle

- **Hot** (recent/popular): CDN + fast object tier.
- **Cold**: works untouched for N months migrate to a cheaper/archive tier;
  first read after that pays a rehydrate latency (surfaced in the UI).
- Derived renditions (thumbnails, EPUBs, watermarked streams) are cache, not
  source of truth — evictable and reproducible from the master + manifest.

### 2.5 Dedup vs. watermarking — the key tension

If we stored a watermarked copy per user, dedup would be impossible (every copy
differs). We resolve this by keeping the **master clean and shared**, and
injecting the per-user payload in a **streaming render step at download time**
(Section 3). Storage stays deduplicated; only the short-lived output stream is
user-specific.

---

## 3. Embedded user tracking — the *chained, compression-resilient* watermark

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

```
users(id, handle, email, pw_hash, roles, totp_secret, state, created_at)
works(id, kind[manga|novel], title, uploader_id, tags, state, rating, created_at)
chapters(id, work_id, index, title)
blobs(b3_hash PK, size, mime, tier, refcount, created_at)
manifest_entries(work_id, chapter_id, page_index, blob_hash, rendition)
phash_index(blob_hash, phash)                      -- perceptual dedup
download_grants(id, user_id, work_id, token, issued_at, ip_trunc, ua_hash)
watermark_seeds(work_id, seed, algo_version)       -- secret, in KMS ref
comments(id, work_id, author_id, parent_id, body, state, created_at)
boards(id, slug, title, mod_roles)
threads(id, board_id, title, author_id, locked, pinned, created_at)
posts(id, thread_id, author_id, body, state, created_at)
reports(id, target_type, target_id, reporter_id, reason, state)
moderation_actions(id, actor_id, target_type, target_id, action, note, ts)
dmca_notices(id, claimant, target_work_id, statements, state, ts)
dmca_counter_notices(id, notice_id, uploader_id, statements, ts)
strikes(id, uploader_id, notice_id, ts)
dmca_events(id, notice_id, event, actor, ts)       -- append-only
audit_log(id, actor, action, subject, meta, ts)    -- append-only
```

---

## 7. Security, abuse, and legal posture

- **Auth**: Argon2id, session cookies (HttpOnly/SameSite), optional TOTP, login
  throttling, breach-password check on set.
- **Uploads**: strict MIME/magic sniffing, image bomb limits, metadata strip,
  virus scan hook, per-account quotas.
- **Access**: entitlement checks on every download; signed, short-TTL URLs; no
  hotlinking of masters (only rendered/watermarked streams leave the perimeter).
- **Rate limiting** everywhere user-generated content or downloads occur.
- **Age/rating gating** for adult doujin categories, per jurisdiction config.
- **Legal**: DMCA designated agent registered; ToS discloses watermarking and
  data retention; clear repeat-infringer policy for safe-harbour.

---

## 8. Phased roadmap

| Phase | Deliverable |
|------:|-------------|
| **0** | Repo scaffold, Docker Compose (Postgres/Redis/MinIO), CI, auth, migrations. |
| **1** | Upload → CAS ingest → AVIF/WebP transcode → reader for manga; novel canonicalise + reader. Exact + perceptual dedup. |
| **2** | Download broker + **watermark v1**: manga DCT spread-spectrum embed/detect, RS+fountain chain, admin detector tool. |
| **3** | Novel stego layers + canonical fingerprint fallback; unify decoder. |
| **4** | Comments + webboard on the shared content/moderation core. |
| **5** | DMCA intake → takedown → counter-notice → strikes → transparency log. |
| **6** | Storage tiering/lifecycle, cold-storage rehydrate, cost dashboards. |
| **7** | Hardening: red-team the watermark (recompress/crop/subset suites), load test, moderation tooling, launch. |

**Watermark validation harness** (built in Phase 2, run every phase after):
an automated suite that takes a marked file, applies a battery of attacks
(JPEG q=50/70/85, AVIF re-encode, 10–40% crop, page-subset, downscale, novel
format-convert + whitespace-normalise) and reports the recovered-token rate per
attack. Ship thresholds are defined against this suite, not vibes.

---

## 9. Open questions to settle before Phase 2

1. **Redundancy vs. quality**: acceptable file-size/PSNR overhead for the manga
   mark? Sets `redundancy_factor` and DCT strength.
2. **Retention**: how long do `download_grants` live? (Attribution power vs.
   data-minimisation.) Proposal: N months, then hash-only.
3. **Novel honesty**: confirm we advertise novel attribution as *degrading to
   work-level* under full stego-stripping.
4. **Jurisdiction**: primary legal home decides DMCA specifics vs. EU/JP
   equivalents and age-gating rules.
5. **Screenshot resistance**: is reader-resolution screenshot survival in scope
   for v1, or a later reader-side (visible/forensic) overlay?
