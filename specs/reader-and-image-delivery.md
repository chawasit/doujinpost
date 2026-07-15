# Spec — Reader & Image Delivery

Status: draft · Owner: platform · Depends on: `PLAN.md` §2,
`specs/history-follows-recommendations.md` (progress),
`specs/community-features.md` (reading modes), `docs/nekopost-comparison.md`

This is the **core product and the headline win over Nekopost.** Nekopost's #1
complaint is slow image loading; its stack (DDoS-Guard, no evidence of modern
codecs or an image CDN) explains why. This spec is how we beat it: an
edge-cached, content-addressed, modern-codec image pipeline, and a reader that
paints instantly and preloads ahead.

Two parts: **A — Image delivery** (the pipeline), **B — Reader UX** (the client).

---

# Part A — Image delivery

## A.1 The delivery model: content-addressed + immutable + edge-cached

Every rendition is addressed by the **content hash of the rendition itself**:

```
https://img.<cdn>/r/<rendition_hash>.avif
Cache-Control: public, max-age=31536000, immutable
```

- The URL changes only when the bytes change, so it can be cached **forever** at
  every layer (browser, CDN edge, service worker). Re-encoding a page yields a
  new hash → automatic cache-bust, no purge needed.
- Because v1 does **no per-user watermarking** (`PLAN.md` §2.5/§3-deferred), the
  bytes are identical for every viewer → one edge object serves everyone →
  **cache-hit ratio approaches 100% once warm.** For a donation-funded nonprofit,
  **cache-hit ratio is the budget** — egress on a miss is the dominant cost.

## A.2 Access control without breaking the cache

The earlier plan said "short-TTL signed URLs." Taken literally that **defeats
caching** — a per-request signature in the query string makes every URL unique,
so the CDN never reuses an object. Reconciled design:

- **Public content (the majority)**: no per-request auth on the image bytes. The
  `rendition_hash` is a BLAKE3 digest — unguessable, so the URL *is* an
  unforgeable capability. Add **hotlink protection** (Referer/Origin allowlist +
  a signed **cookie** scoped to the image host) so other sites can't embed our
  bandwidth. The cache key ignores the cookie → still one shared object.
- **Restricted content (age-gated `warning:*`, unpublished, DMCA-holds)**: gate
  with **signed cookies** (CloudFront signed cookies / Cloudflare signed tokens),
  **not** signed URLs. The cookie authorises a *path prefix* for the session; the
  **cache key stays the content hash** (auth is validated at the edge, not baked
  into the URL), so restricted content is still edge-cacheable across entitled
  users. Entitlement is decided once when the reader page loads, not per image.
- Never expose masters; only derived renditions are reachable. Age/rating checks
  happen server-side before the reader page hands out the cookie.

## A.3 Rendition ladder (transcode on ingest)

On upload, the Bun transcode worker (`sharp`/libvips) produces, per page:

| Rendition | Purpose |
|---|---|
| `avif` @ widths {480, 720, 1080, 1600, orig} | primary; ~50% smaller than JPEG |
| `webp` @ same widths | fallback for clients without AVIF |
| `jpeg` @ 1080 | last-resort fallback |
| `thumb` (webp, ~320w) | grids/cards |
| **blurhash / LQIP** string | instant placeholder (stored in DB, not a blob) |
| stored **intrinsic width/height** | reserve layout box → zero CLS |

- Strip all metadata (privacy + size); normalise colour profile.
- Renditions are **derived cache** over the CAS master (`PLAN.md` §2.1) —
  reproducible and evictable; the master is the only source of truth.
- Novels: pages don't apply; the canonical XHTML + on-demand EPUB path (§2.3)
  is the analogue, and text is tiny so delivery is trivial.

## A.4 Serving the right bytes

- **Responsive**: `<img srcset>`/`<picture>` with `sizes` so a phone pulls the
  480/720 rendition, never the full-res master — the single biggest bandwidth
  saving and directly fixes Nekopost's "huge images on mobile" problem.
- **Format negotiation**: `<picture>` with AVIF → WebP → JPEG sources; the
  browser picks. (Avoid `Accept`-header branching at origin — it fragments the
  cache; let the markup negotiate so each format keeps a stable URL.)
- **Save-Data / slow links**: honour the `Save-Data` header and Network
  Information API to drop a tier (serve 720 where 1080 would go) and disable
  aggressive preloading — the Thai mobile-majority audience on 4G/3G matters.
- **HTTP/3 + priority hints** on the image host for fast multiplexed page fetch.

## A.5 CDN & cost

- Use a **real image CDN** with cheap egress and global POPs (Cloudflare / Fastly
  / BunnyCDN) — explicitly **not** a DDoS-mitigation host like Nekopost's, which
  isn't POP-optimised for images. Keep DDoS protection *and* image performance.
- Origin is the object store (S3/MinIO); the CDN pulls once per rendition, then
  serves from edge. Monitor **cache-hit ratio, egress GB, p95 image TTFB** — the
  three numbers that decide whether the nonprofit can afford to run.
- Cold-tier rehydrate latency (`PLAN.md` §2.4) is surfaced in the reader as a
  brief "loading from archive" state.

---

# Part B — Reader UX

The client that turns fast bytes into a good read. `reading_mode` on the series
(`specs/community-features.md` §G) selects the reader.

## B.1 Paged reader (manga / comic)

- **Direction**: LTR or **RTL** (manga default), from `reading_mode`.
- **Layout**: single page, or **double-page spread** on wide viewports with
  correct cover/offset handling; fit-width / fit-height / original / smart-fit.
- **Navigation**: keyboard (←/→/space), **tap zones** (prev/next/menu), swipe,
  a page slider with thumbnails, and jump-to-page.
- **Zoom/pan**: pinch + double-tap zoom, drag to pan, reset on page change.
- **Preload**: fetch the next **N pages** (N adapts to connection; paused under
  Save-Data) and keep a few previous decoded — so page turns are instant.
- **Decode off the main thread**: `img.decode()` / `createImageBitmap` before
  showing, so a turn never janks.

## B.2 Vertical reader (webtoon / long-strip)

- **Continuous lazy scroll**: virtualised list — only pages near the viewport are
  in the DOM/decoded; far ones are released to cap memory on long chapters.
- **Placeholder boxes** sized from stored intrinsic dimensions → **zero layout
  shift** as images stream in (Nekopost jumps; we won't).
- Configurable inter-page **gap** (0 for seamless webtoons), brightness/bg.
- Progress tracked by **scroll %** → `read_progress.position = {scroll_pct}`.

## B.3 Novel reader

- Typography controls: font family/size, line-height, margins, theme
  (light/sepia/dark), justification; paginated **or** scroll mode.
- Resume by **paragraph/offset**; per-chapter nav; in-text bookmarks.
- **TTS** (later): Web Speech API read-aloud.

## B.4 Cross-cutting reader concerns

- **Instant paint**: blurhash/LQIP shows immediately; the sharp image fades in on
  decode. No blank grey boxes.
- **Resume**: on open, jump to `read_progress` (page or scroll %); "Continue
  reading" entry point from home (personalisation spec).
- **Progress pings**: throttled, Redis-buffered writes (`specs/history-...` §A.2)
  — never block paging on a network write.
- **Offline (PWA)**: a **service worker** caches the current chapter's renditions
  (Cache API) so an opened chapter re-reads offline; an explicit "save for
  offline" pins a chapter. **Entitlement respected** — restricted content is only
  cached after the session is authorised, and cleared on sign-out. The SW must
  **not** cache authenticated API/JSON (only immutable image renditions).
- **Accessibility**: full keyboard nav, focus management, `alt`/aria on pages,
  screen-reader-first novel mode, `prefers-reduced-motion` disables fades,
  respects OS dark mode.
- **Anti-scrape reality check**: the reader necessarily exposes rendition URLs;
  hotlink/referer + signed cookies (A.2) and rate limits raise the cost of bulk
  scraping, but with watermarking deferred there is no per-user leak trace — an
  accepted trade for a free scanlation site (`docs/nekopost-comparison.md` §6).

---

## C. Performance budget & metrics

Targets (mobile, mid-tier Android, 4G) — measured, not assumed:

- **LCP** (first page image) < 2.5 s; time-to-first-page-interactive < 3 s.
- **Page-turn latency** (paged) < 100 ms when preloaded.
- **CLS** ≈ 0 (dimensions reserved from stored width/height).
- **Cache-hit ratio** > 95% steady-state; **p95 image TTFB** < 150 ms at edge.

Instrument with Core Web Vitals (RUM) + CDN analytics; these numbers *are* the
"better service health" claim, so they're tracked from Phase 1, not retrofitted.

---

## D. Roadmap fit

- **Phase 1**: transcode ladder + blurhash + content-addressed immutable URLs +
  CDN + responsive `<picture>`; paged **and** vertical readers; resume; RUM.
- **Phase 1–2**: Save-Data tiering, double-spread, offline SW pinning.
- **Later**: novel TTS, per-tile progressive for oversized pages.

**Test checklist**: immutable URL cache-busts on re-encode; public image needs no
per-request auth yet resists hotlinking; restricted image gated by **cookie** with
an unchanged cache key (verify edge hit across two entitled users); phone pulls a
small rendition via `srcset`; AVIF→WebP→JPEG fallback; blurhash paints before
decode; zero CLS on vertical stream; preload makes turns instant; Save-Data drops
a tier; resume lands on the right page/scroll; offline SW replays an opened
chapter but not on sign-out; RUM reports LCP/CLS.
