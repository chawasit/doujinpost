# Spec — Community Features (Nekopost-parity pack)

Status: draft · Owner: platform · Depends on: `PLAN.md`,
`specs/roles-and-content-model.md`, `specs/history-follows-recommendations.md`,
`docs/nekopost-comparison.md`

The recon (`docs/nekopost-comparison.md`) found our plan strong on infrastructure
but thin on the *community texture* that kept Nekopost sticky for 15 years. This
spec adds that texture:

- **A — Creator profiles** (Nekopost "editor" pages)
- **B — Shoutbox** (site-wide / per-series chat)
- **C — Title requests** ("Request Manga")
- **D — Gamification** (badges, points, levels)
- **E — Ranking & discovery surfaces** (Weekly Popular, Latest, Rising)
- **F — Public ratings & reactions**
- **G — Webtoon / vertical reader + content formats**

Everything here reuses the existing moderation core (`reports`,
`moderation_actions`), verification levels, and the follows/notifications
plumbing — it's texture on top of built systems, not new foundations.

---

## A. Creator profiles

A public page per creator (translator / writer / artist), the counterpart to
Nekopost's `/editor/:id`. Mostly a **view** over existing data.

- **URL**: `/@handle`. Tabs: **Works · Badges · About · Activity**.
- **Works ("projects")**: series the user has a `content_grant` on (owner /
  coauthor / translator), grouped by role, with per-series status and update
  recency. A "project" = a series; no new entity.
- **Header stats**: follower count (`follows` where `target_type=user`), total
  works, aggregate reads, join date, and top badges.
- **About**: bio, links (socials, group site), languages, open-to-collab flag.
- **Follow** from here (reuses Part B of the personalisation spec →
  notifications on new series/chapters).
- **Groups**: an optional `groups` construct (translation teams) — a creator can
  belong to groups; a group has its own profile and members with roles. Ties to
  gaps #12 (staff credits). Series can be credited to a group.

```sql
profiles (user_id pk, bio text, links jsonb, languages text[],
          open_to_collab bool, banner_blob bytea, updated_at)
groups (id pk, slug, name, about, created_by, created_at)
group_members (group_id, user_id, role)          -- lead|member
series_credits (series_id, user_id|group_id, credit)  -- translator|cleaner|typesetter|writer|artist
```

---

## B. Shoutbox

A lightweight real-time chat — Nekopost has a site-wide one (asset
`WrapperShoutBox`). High engagement, **high moderation load** — designed
conservatively.

- **Scopes**: a global site shoutbox; optionally per-series shoutboxes (a series'
  fan chat). Same component, different `room`.
- **Transport**: WebSocket via **Elysia/Bun native WS**. Last ~50 messages kept
  hot in **Redis** per room; a short DB retention window (e.g. 7–30 days) for
  moderation/audit, then trimmed. Chat is ephemeral, not archival.
- **Post gate**: verification **L1** (email-verified) to post; read is public.
- **Anti-abuse (mandatory)**: per-user rate limit + **slow mode**, a profanity/
  link filter, new-account throttle, and mod tools — delete message, **mute**
  (timed), ban from room, and a global **read-only kill switch**. Every mod
  action → `moderation_actions`.
- **Launch posture** (open question in `PLAN.md` §9): ship **read-mostly** with
  tight rate limits and slow-mode on; loosen once moderation coverage exists.
  A real-time firehose with no mods is how communities get sued and doxxed.

```sql
shout_messages (id, room text, user_id, body, created_at, deleted_by null)
shout_mutes (room, user_id, until, by)
```

---

## C. Title requests ("Request Manga")

Readers ask for titles; the crowd upvotes; translators claim and fulfil. A demand
signal that also **directs volunteer effort** — valuable for a nonprofit
scanlation community.

- **Submit**: title, optional source link/description, language wanted. Deduped
  against existing series and open requests (fuzzy match warns before creating).
- **Upvote**: one per user; ranks the request queue.
- **Lifecycle**: `open → claimed → fulfilled` (also `duplicate | rejected |
  unavailable`). A Writer **claims** a request (soft-lock, expires if idle) and
  on publish links the new `series_id` → requesters auto-notified (follows/
  notifications).
- **Anti-spam**: L1 to submit, per-user request cap, mod moderation of the queue.

```sql
title_requests (id, requester_id, title, source_url, language, description,
                state, claimed_by null, claimed_at null, fulfilled_series_id null,
                upvotes int, created_at)
request_votes (request_id, user_id)              -- unique
```

---

## D. Gamification (badges · points · levels)

Nekopost gamifies both creators and readers (badges, points) — a genuine
retention and volunteer-motivation driver. Built as an **event-driven rules
engine**, not scattered counters.

- **Events** (reads, comment, rating given, chapter published, request fulfilled,
  daily login streak, received-follow, received-reaction) are emitted to the
  worker queue; a rules engine awards points and evaluates badge criteria.
- **Badges**: milestone (first upload, 10/100 chapters, 100k reads), role
  (verified translator, group lead), quality (highly-rated series), and event
  badges. Auto-awarded; some admin-granted.
- **Levels**: derived from XP; cosmetic + unlocks minor perks (e.g. higher rate
  limits, custom profile flair) — never gates safety features.
- **Leaderboards**: top creators/translators/groups, weekly + all-time.

> **⚠️ The critical design guard (PLAN.md §9 open question).** Points must reward
> **quality and engagement, not raw upload volume** — otherwise you incentivise
> mass junk/pirated uploads, the exact failure a scanlation site must avoid.
> So: weight by *reads/ratings/completion*, apply **time-decay**, cap points per
> action, require the content to survive moderation before points settle, and
> support **mod clawback** on removed content. Reader points likewise resist
> comment/rating spam (diminishing returns, dedupe).

```sql
badges (id, slug, name, description, icon, criteria jsonb, is_admin_only)
user_badges (user_id, badge_id, awarded_at, context jsonb)
points_ledger (id, user_id, delta, reason, ref jsonb, settled bool, created_at)  -- append-only
levels (user_id, xp, level, updated_at)          -- derived cache
```

---

## E. Ranking & discovery surfaces

Nekopost surfaces content mainly through **ranking pages**, not search. We have
the `trending` data (`specs/history-follows-recommendations.md`); this adds the
first-class **surfaces**.

- **Weekly Popular** — time-decayed reads/views over a 7-day window.
- **Latest Updates** — most recently published chapters (the default "what's
  new" feed).
- **Rising / Trending** — read *velocity* (catches new hits before raw counts).
- **Top Rated** — from Part F public ratings (min-count guarded).
- **Recommended** — personalised (recs spec) when logged in.
- Each is available **globally and per-category/per-tag**.

Backed by a **ranking worker** that refreshes materialised rollups on a schedule
(reads/velocity are expensive to compute live). Guard against gaming: dedupe
reads per user/session, ignore self-reads, decay, and a min-sample floor for
rated lists.

```sql
rankings (surface text, scope text, series_id, rank int, score, window, computed_at)
```

---

## F. Public ratings & reactions

Distinct from the **private** `library.rating` (personalisation spec): this is
the **public aggregate** that feeds Top-Rated and recs, plus lightweight
reactions.

- **Rating**: a user's public score for a series (thumbs or 1–5/1–10, pick one
  and keep it). Aggregate (`avg`, `count`) cached on the series; shown on cards;
  min-count before a score displays (avoids 1-vote 5-stars).
- **Reactions**: emoji reactions on **chapters** and **comments** (❤️ 😂 😮 …) —
  cheap engagement, feeds gamification and "funny comments" surfacing (Nekopost
  highlights these).
- Anti-abuse: L1 to rate/react, one rating per user per series, rate-limit,
  exclude from a series' own creator.

```sql
ratings (series_id, user_id, score, created_at)          -- unique (series,user), public
reactions (target_type[chapter|comment], target_id, user_id, emoji, created_at)
series_rating_agg (series_id, avg, count, updated_at)    -- cache
```

---

## G. Webtoon / vertical reader + content formats

Nekopost serves **manga, comic, webtoon, webcomic, novel, original novel**. Our
model needs the **vertical (long-strip) reading mode** and format flags.

- Keep `series.type ∈ {manga, novel}`; add **`reading_mode ∈ {paged_ltr,
  paged_rtl, vertical}`** and a **`format` tag** (`manga`/`comic`/`webtoon`/
  `webcomic`) via the existing `meta:` tag namespace so browse/labels work
  without a schema explosion.
- **Reader modes**:
  - *paged* (manga/comic): LTR or **RTL**, single/**double-page spread**,
    fit-width/height, zoom, keyboard/tap nav, preload next N pages.
  - *vertical* (webtoon): continuous lazy-loaded scroll, gap control, progress by
    scroll position (feeds `read_progress.position = {scroll_pct}`).
  - *novel*: font/size/theme/line-height, paginated or scroll, resume by
    paragraph, optional TTS (later).
- `reading_mode` drives which reader loads; defaults inferred from `format` tag
  at upload (webtoon → vertical, manga → paged_rtl, comic → paged_ltr).

```sql
-- series gains: reading_mode text  (paged_ltr | paged_rtl | vertical)
```

---

## H. Moderation & abuse (cross-cutting)

Every surface here is a spam/abuse vector and routes through the existing core:

- All user content (shout, requests, ratings, reactions, profile text) is
  **reportable** → `reports` queue; all mod actions → `moderation_actions`.
- Verification-level gating (L1 for social actions) + rate limits everywhere.
- Gamification clawback + points-settle-after-moderation prevents reward farming.
- Shoutbox gets the heaviest tooling (mute/ban/slow-mode/kill-switch) — it's the
  biggest real-time risk.

---

## I. Roadmap fit (updated `PLAN.md` §8)

- **Phase 2**: ranking surfaces + public ratings (ride the catalog).
- **Phase 3**: creator profiles + shoutbox + title requests (+ history/library/
  follows from the personalisation spec).
- **Phase 4**: gamification (needs event history to be meaningful) + webboard.
- Webtoon/vertical reader lands in **Phase 1** with the reader (it's core, not
  texture).

**Test checklist**: profile aggregates match grants/follows; group credits render;
shoutbox WS rate-limit + mute + kill-switch; request dedupe + claim expiry +
fulfil-notify; **points reward engagement not upload count**; clawback on removed
content; ranking dedupes self/duplicate reads + min-sample floor; public vs
private rating kept separate; reactions gated + deduped; vertical reader resumes
by scroll %; reading_mode inferred from format on upload.
