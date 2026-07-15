# Spec — History, Follows & Recommendations

Status: draft · Owner: platform · Depends on: `PLAN.md` §3–6,
`specs/roles-and-content-model.md` (series/chapters/tags),
`specs/authentication.md` (sessions)

Three linked systems that make the site personal:

- **Part A — Reading history** — what a user has read + resume position
  ("continue reading").
- **Part B — Follows & library** — subscribe to series/creators/tags for updates;
  shelve series with a status and rating.
- **Part C — Recommendations** — turn A + B into "what to read next".

History and follows are private by default; they are also the **signals** the
recommender consumes. Privacy therefore isn't a bolt-on — it's load-bearing.

---

# Part A — Reading history

## A.1 What we record

- **Resume position** per (user, series): last chapter + intra-chapter position
  (manga = page index; novel = paragraph/scroll offset) + percent complete. This
  is the "Continue reading" state — one row per user+series, upserted.
- **Read events** (optional, retention-capped): an append log of
  `open | progress | complete` used for recs signal, trending, and analytics —
  not for display. Trimmed to a rolling window (e.g. 180 days) or last N per user.

```sql
read_progress (
  user_id uuid, series_id uuid,
  chapter_id uuid, position jsonb,   -- {page:N} | {para:N, offset:pct}
  percent numeric, updated_at timestamptz,
  primary key (user_id, series_id) )

read_events (                         -- append-only, TTL-trimmed
  id bigint pk, user_id uuid, series_id uuid, chapter_id uuid,
  event text, dwell_ms int, ts timestamptz )
```

## A.2 Write path (it's high-volume)

- The reader emits lightweight progress pings (throttled, e.g. every ~10s or on
  page turn). Pings hit **Redis** first (hot resume state), flushed to
  `read_progress` async by a worker; `read_events` are enqueued, not written
  inline. This keeps the read UX off the DB hot path.
- A `complete` event fires when the last page/paragraph is reached → feeds
  library auto-status (B.2) and recs.

## A.3 Privacy & controls (defaults matter)

- History is **private by default** — visible only to the user. A profile
  "reading activity" is opt-in per user.
- **Incognito read**: a session toggle that suppresses history/events entirely.
- **Manage history**: view, remove a single entry, or clear all; an
  "exclude from recommendations" flag lets a user keep resume state but stop a
  series from influencing recs.
- Deleting history is a real delete of `read_progress`/`read_events` (recs
  recompute on next batch), not a hide flag.

---

# Part B — Follows & library

## B.1 Follows (get updates)

Follow a target to receive updates; the target type decides what triggers a
notification.

```sql
follows (
  user_id uuid, target_type text,   -- 'series' | 'creator' | 'user' | 'tag'
  target_id uuid,                   -- series id, creator-tag id, user id, tag id
  notify text[],                    -- {chapter,notice,new_series}
  created_at timestamptz,
  primary key (user_id, target_type, target_id) )
```

| Target | Fires on |
|---|---|
| **series** | new published Chapter, new Notice, Extra |
| **creator** (circle/author, via `creator:` tag) | new series, new chapter from that creator |
| **user** (a Writer/translator account) | new series they publish |
| **tag** / category | new series matching — a *discovery* subscription (digest, low-noise) |

**Delivery** (reuses the PWA + worker stack):

- On a publish event, a worker **fans out** to followers → writes
  `notifications` rows (in-app bell) and, per user prefs, sends **Web Push**
  (PWA) and/or batches into an **email digest**. Fan-out-on-write for the
  notification feed; large-follower creators fall back to fan-out-on-read to
  avoid write storms.
- Per-user notification preferences (channel + per-follow `notify[]`) gate
  everything; respect quiet-hours and a daily cap; deep-link straight to the new
  chapter.

```sql
notifications (id, user_id, type, subject_ref jsonb, read_at, created_at)
```

## B.2 Library (save & shelve)

Following ≠ saving. The **library** is the user's shelf with an explicit status
and optional rating — the backbone signal for recs and for "my list".

```sql
library (
  user_id uuid, series_id uuid,
  shelf text,        -- 'reading'|'planned'|'completed'|'on_hold'|'dropped'|'favorite'
  rating int null,   -- 1..10 or thumbs, user's own
  is_private bool default true,
  updated_at timestamptz,
  primary key (user_id, series_id) )
```

- Auto-transitions from history: first read → offer `reading`; `complete` on the
  final chapter → offer `completed` (suggested, never forced).
- `favorite` and high `rating` are the strongest positive recs signals; `dropped`
  is a strong negative signal (down-rank the series *and* its dominant tags).

---

# Part C — Recommendations

Goal: a "what to read next" that is **relevant, explainable, cold-start-safe,
and privacy-respecting**. We blend three cheap-to-strong approaches rather than
betting on one.

## C.1 Signals

Implicit: reads, dwell/completion ratio, downloads, follows, resume-but-abandon.
Explicit: library shelf + rating, favorite, "not interested", blocked tags.
All are **private-history-derived** — recs never leak what another user read.

## C.2 Approaches (blended)

1. **Content-based (tag/category profile).** Build a per-user **taste vector**
   over tags/categories, weighted by engagement (completed+favorited ≫ opened;
   dropped negative). Recommend series whose tag vector is close (cosine) and
   unread. Cheap, works from a single read, and **explainable**: "because you
   read *X*" / "more `theme:isekai`".

   ```sql
   user_taste (user_id, tag_id, weight, updated_at)   -- rebuilt from signals
   ```

2. **Collaborative (item-item co-occurrence).** Offline batch computes
   "readers who finished A also finished B" similarity (implicit ALS or
   normalized co-occurrence) over the read/library matrix. Catches affinities
   tags miss. Stored precomputed:

   ```sql
   item_similarity (series_id, other_series_id, score, method, computed_at)
   ```

3. **Popularity / trending fallback.** Time-decayed reads+downloads per scope
   (global, per category, per tag) for cold-start users and to seed diversity.

   ```sql
   trending (scope text, series_id, score, window, computed_at)
   ```

## C.3 Serving pipeline

```
batch (worker, periodic):
  rebuild user_taste · item_similarity · trending  → cache per-user candidate sets
online (request time):
  merge candidates (content ⊕ collaborative ⊕ trending)
  FILTER: drop read/dropped/blocked-tag; enforce age-gate & rating prefs;
          respect "exclude from recs" and "not interested"
  RANK: weighted blend + recency + diversity (MMR so one tag/creator
        doesn't dominate a row)
  → surfaces (C.4) with an attached reason
```

- Precompute candidates in a batch worker; do the cheap filter/rank/dedup
  **online** so brand-new reads and just-published series show up promptly.
- **Filters are hard constraints**, not soft penalties: already-read, `dropped`,
  `blocked` tags, and age-gate (`warning:*` from the tags spec) are removed, not
  merely down-weighted.

## C.4 Where recs surface

- **Continue reading** (from `read_progress`) — always first when present.
- **New from your follows** (from Part B fan-out).
- **Because you read *X*** — content-based, carries the explaining series.
- **Readers also enjoyed** — item-item.
- **Popular / Trending in {category|tag}** — cold-start + discovery.
- Home feed = an ordered composition of these rows, deduped across rows.

## C.5 Controls, explainability, fairness

- Every rec carries a **reason** ("because you read…", "popular in Romance") —
  shown in the UI; recs with no defensible reason aren't shown.
- **Not interested / hide series / mute tag / mute creator** feed straight back
  into the filter set.
- **Reset personalization** clears the taste profile (history choice separate).
- **Cold start**: no history → trending + onboarding tag-picker to seed
  `user_taste`.
- Guardrails: cap over-personalization with a diversity/exploration slice so new
  and smaller creators still surface; never let `warning:*`-gated content leak
  past the viewer's age setting.

---

## D. Roadmap fit

- **Phase 1**: `read_progress` + "Continue reading" (ships with the reader);
  `library` shelves.
- **Phase 4**: `follows` + notifications (rides the comments/forum + notification
  plumbing) and Web Push.
- **Phase 6+**: batch recommender (`user_taste`, `item_similarity`, `trending`)
  and the composed home feed — after there's enough interaction data to be
  worth it; until then, home = follows + trending + continue-reading.

**Test checklist**: resume position round-trips per format; incognito writes
nothing; clear-history really deletes + recs recompute; follow fan-out hits
in-app + Web Push within prefs; huge-follower creator uses fan-out-on-read;
library auto-status suggestions are opt-in; recs exclude read/dropped/blocked/
age-gated as hard filters; every rec has a reason; cold-start falls back to
trending; "not interested" persists into future ranking; diversity slice present.
