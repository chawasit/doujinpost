# Nekopost.net — Reconnaissance & Gap Comparison

Target being cloned: **nekopost.net** — a Thai manga/comic/novel community
scanlation-and-original platform, online since 2009. Goal of *this* project:
same product, **better codebase and service health**. This doc records what
Nekopost actually is, how it's built, where it hurts, and how our current specs
line up (over- and under-shooting).

> Method: public pages + HTTP fingerprinting on 2026-07-15. Nekopost is a
> client-rendered app, so some feature detail is inferred from asset/route names
> and third-party writeups, not from reading its source.

---

## 1. What Nekopost is (observed)

- **Content types**: `manga`, `comic`, `novel`, `original novel`, and (per meta)
  **webtoon / webcomic**. Meta description: *"Read manga, light novel, webtoon,
  webcomic, and more."*
- **Discovery**: genre/category browse (Shonen, Shoujo, Isekai, School Life,
  Mystery, Fantasy, Romance…), stackable genre tags, **"Weekly Popular"** and
  **"Recommended"** rankings. Ranking-by-popularity, not user star ratings, is
  the primary surfacing mechanism.
- **Creators** ("**editor**" profiles, e.g. `/editor/77652`): profile with tabs
  **Home / Projects / Badges / About**, a works/"projects" list, and a
  **badges/achievements** system. Creators are translators, writers, and artists.
- **Community**: comments per work, a **shoutbox** (site-wide chat), ratings/
  reactions, following series, and a **"Request Manga"** flow (readers ask for
  titles — e.g. `/manga/4880` "Request Manga").
- **Accounts**: **Google sign-in** (prominent), plus a creator "backend" portal.
- **Funding**: a **"Support us"** donation mechanism (a support % is shown) — no
  storefront/paywall observed; content reads free. Community-run ("powered by
  CLAZ and contributors").
- **Scale**: historically >1M visits/month; still Top-5 in Thailand's
  Animation & Comics category. ~600K+ recent monthly visits.
- **Legal posture**: predominantly **unlicensed fan scanlation** (Thai forums
  openly discuss the lack of licensing) — i.e. it operates in the usual
  scanlation grey zone.

## 2. How it's built (fingerprint)

| Signal | Finding |
|---|---|
| Frontend | **SvelteKit** — assets served from `/_app/immutable/…` with hashed names (SvelteKit's signature). Not Next.js/Nuxt. |
| Asset/route names | `WrapperShoutBox`, `commentBox`, `postEditor`, `postBoxPreview`, `Toaster`, `pageNormal`, `ui-utility` — confirms shoutbox, comments, a post/blog editor, toasts. |
| Rendering | Thin HTML shell (~13 KB) → client-hydrated app. SEO/first-paint depend on how much SvelteKit SSRs. |
| Edge/CDN | **DDoS-Guard** (`server: ddos-guard`) — a Russian DDoS-mitigation/CDN, commonly used by sites mainstream CDNs (Cloudflare/Fastly) won't keep. **Not** a performance-tuned global image CDN. |
| Analytics | Google Analytics (gtag `G-CLBWZ0Z2RC`). |
| Auth | Google OAuth. |
| Response | Homepage ~0.6–1.0 s TTFB through DDoS-Guard in this probe. |

## 3. Service-health pain points (the thing we're meant to beat)

1. **Slow image loading** — the most common user complaint (Thai forums:
   *"nekopost โหลดรูปช้า"* = "Nekopost loads images slowly"). For a manga reader
   this is *the* core-product failure. Root causes are consistent with what we
   see: DDoS-Guard is not an image-optimising CDN, and there's no evidence of
   modern codecs (AVIF/WebP), responsive/tiled images, or edge image caching.
2. **DDoS-Guard dependency** signals a history of attacks/abuse **and** a hard
   constraint (deplatforming risk) — plus mediocre global image performance vs.
   a real image CDN.
3. **Client-rendered shell** → SEO and first-load are only as good as their SSR
   discipline; large hydration cost on low-end mobile (the majority audience).
4. **High bounce / short sessions** in one traffic sample — consistent with
   friction (load speed, ads/redirects on mirror domains).

## 4. Feature parity vs. our specs

Legend: ✅ covered · 🟡 partial · ❌ missing · ➕ we add (Nekopost lacks).

| Nekopost feature | Our plan | Notes |
|---|---|---|
| Manga / comic reader | ✅ | but our reader UX is one clause — see gaps #10 |
| Novel / original novel | ✅ | `series.type` + novel canonicalisation |
| **Webtoon / webcomic (vertical)** | 🟡 | must add a `meta:long-strip` type + **vertical reader mode** |
| Genre categories + sub-cats | ✅ | our category tree + descriptions |
| Stackable genre tags | ✅ | our namespaced tags (richer than theirs) |
| **Weekly-popular / recommended ranking** | 🟡 | we have `trending`, but need the *explicit surfaced ranking pages* |
| User ratings / reactions | 🟡 | `library.rating` exists; per-work public rating/reaction UI not specced |
| Comments per work | ✅ | |
| **Shoutbox (site-wide chat)** | ❌ | Nekopost has one; we don't |
| Webboard / forum | ➕✅ | we spec a forum; Nekopost leans on shoutbox+comments |
| Follow series + notifications | ✅ | our follows/notifications spec |
| **"Request Manga" flow** | ❌ | reader requests a title; not in our specs |
| **Creator profiles / "projects"** | 🟡 | we have `content_grants`; no public creator profile/portfolio page |
| **Badges / gamification / points** | ❌ | Nekopost gamifies creators & readers; we have none |
| Translation groups / staff credits | 🟡 | flagged in gaps #12; not yet specced |
| Google sign-in | ✅ | |
| Passkeys / SMS verify | ➕ | we add; Nekopost doesn't appear to |
| Donations / "support us" | 🟡 | we chose **off-platform** donations (nonprofit) |
| DMCA / takedown pipeline | ➕ | we add a real one; Nekopost's is minimal |
| **Forensic per-user watermark** | ➕❓ | we add; Nekopost does **not** — see §6 |
| Optimised image delivery (AVIF/CDN) | ➕✅ | **our biggest intended win** |
| Backups / DR / observability | ➕ | our gaps doc; unknown for Nekopost |

## 5. What we're missing that a faithful clone needs

Add these to be a real Nekopost clone (they're small next to what we've specced):

1. **Vertical webtoon reader + webtoon/webcomic content types.**
2. **Shoutbox** — site-wide (and/or per-series) lightweight chat.
3. **"Request a title" board** — reader demand signal, feeds translators.
4. **Public creator profiles** — portfolio, projects, follower count, About.
5. **Gamification** — badges/points/levels for creators *and* readers (this is a
   real retention driver on Nekopost; it also motivates unpaid translators).
6. **Explicit ranking pages** — "Weekly Popular", "Recommended", "Latest Update"
   as first-class surfaces (we have the `trending` data, not the pages).
7. **Public per-work rating/reaction** surface (distinct from private library
   rating).
8. **Translation-group / staff-credit** model (already in gaps #12).

## 6. What we *over*-built relative to the target

Nekopost is a **free community scanlation site**. Two big pieces of our plan are
heavier than the target — worth an explicit decision:

- **Forensic per-user watermarking (`PLAN.md` §3).** Nekopost doesn't do this,
  and conceptually it's a **poor fit for free scanlation**: the content is
  already freely shared, so "trace the leaker" has little meaning, while the
  cost is severe (per-user renders defeat CDN caching — gaps #5 — the *exact*
  image-delivery problem we're trying to *beat*). If the real goal is "better
  than Nekopost," the watermark actively works against the headline metric.
  **Recommendation**: drop watermarking for in-app reads entirely; keep it (if
  at all) only for opt-in official/licensed uploads, or cut it for v1.
- **Passkeys + SMS verification.** Nice, but Nekopost gets by on Google-only.
  Keep passwordless/Google as the fast path; treat passkeys/SMS as later hardening.

Meanwhile our **legal/T&S spine and image-delivery optimisation are genuine
upgrades** over Nekopost and should stay front-and-centre — they're most of the
"better service health" promise.

## 7. "Better codebase & service health" — concrete wins

1. **Fix image delivery first — it's Nekopost's #1 complaint.** AVIF/WebP,
   responsive/tiled images, blurhash placeholders, aggressive **edge caching on
   a real image CDN** (Cloudflare/Fastly/bunny), and preloading in the reader.
   This single area is where we most visibly beat them — *don't* undermine it
   with per-user watermarking.
2. **Move off DDoS-Guard-style hosting** to a mainstream CDN + object store;
   keep DDoS protection without sacrificing global image POPs.
3. **SSR/SEO discipline** in SvelteKit-or-our-stack: our plan uses a React PWA —
   ensure catalog/series/chapter pages server-render for SEO and fast first
   paint on low-end mobile (the Thai audience is mobile-majority).
4. **Observability + backups** (gaps #7) — Nekopost's uptime/health is opaque;
   ours should be measured (SLOs, PITR, DR).
5. **Keep the stack boring and self-hostable** (nonprofit/OSS) so the community
   can run mirrors without the DDoS-Guard-tier compromises.

## 8. Net finding

Our specs are **stronger than Nekopost on infrastructure** (storage, image
optimisation, auth, DMCA, observability) and **weaker on community/product
texture** (shoutbox, requests, gamification, creator profiles, webtoon reader,
ranking surfaces) — the very things that made Nekopost sticky for 15 years. And
one flagship feature — the **forensic watermark** — is arguably *counter* to the
"faster than Nekopost" goal and should be reconsidered. Close the community-
feature gap, nail image delivery, and drop/scope-down the watermark, and this is
a credible "better Nekopost."

Sources: Nekopost [/about](https://www.nekopost.net/about),
[homepage & fingerprint](https://www.nekopost.net) (observed 2026-07-15),
[/manga](https://www.nekopost.net/manga),
[Yakpood retrospective](https://yakpood.com/nekopost/),
[Pantip on scanlation licensing](https://pantip.com/topic/42169202).
