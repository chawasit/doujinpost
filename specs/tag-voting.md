# Spec — Community Tag Voting (E-Hentai / nhentai style)

Status: draft · Owner: platform · Depends on:
`specs/roles-and-content-model.md` §B.1.2 (tag vocabulary), `specs/search.md`,
`specs/community-features.md` (gamification/reputation)

Crowdsourced, self-correcting tagging in the style of **E-Hentai / ExHentai**:
any trusted user can apply a namespaced tag to a series, and the community
**upvotes/downvotes** it until consensus decides whether it sticks. This scales
tagging far past what a handful of mods or the uploader alone can do — the thing
that makes those catalogs so searchable.

**The clean split** (this is the whole design):

| Layer | Who controls it | Where |
|---|---|---|
| **Tag vocabulary** — which tags *exist*, aliases, implications, blocked/restricted | **Mods** (curated) | `roles-and-content-model.md` §B.1.2 |
| **Tag assignment** — which tags apply to *this series* | **Community vote** (this spec) | here |

Vocabulary stays curated so the namespace doesn't rot; *assignment* is
democratic so coverage explodes. E-Hentai works exactly this way.

---

## 1. Data model

```sql
-- extends series_tags from roles-and-content-model.md
series_tags (
  series_id uuid, tag_id uuid,
  proposed_by uuid,
  score numeric,            -- weighted vote sum (denormalised, recomputed on vote)
  state text,               -- 'proposed' | 'confirmed' | 'rejected' | 'locked'
  locked_by uuid null,      -- mod pin/removal overrides votes
  created_at timestamptz,
  primary key (series_id, tag_id) )

tag_votes (
  series_id uuid, tag_id uuid, user_id uuid,
  value smallint,           -- +1 | -1
  weight numeric,           -- voter's tagging weight at cast time (snapshotted)
  created_at timestamptz,
  primary key (series_id, tag_id, user_id) )

tagger_rep (
  user_id uuid pk,
  score numeric,            -- reputation → tagging weight
  confirmed int, rejected int,   -- history counters
  updated_at timestamptz )
```

---

## 2. Consensus mechanics

- **Proposing** a tag creates a `series_tags` row in `state='proposed'` with an
  implicit **+1 from the proposer** (weighted by their rep).
- **Voting**: each user casts at most one `+1`/`-1` per (series, tag); changing or
  clearing a vote recomputes the score. `score = Σ(value × weight)`.
- **Weighted by tagger reputation** (`tagger_rep.score`), bucketed into a bounded
  weight (e.g. 1×–5×). New/low-rep users count ~1×; proven taggers more. **Weight
  is capped** so no single account dominates.
- **State transitions** (thresholds are config, not hardcoded):
  - `score ≥ +C_confirm` → **`confirmed`** (searchable, shown, counted).
  - `score ≤ −C_reject` → **`rejected`** (hidden, unsearchable; kept as a
    tombstone so it can't be trivially re-spammed).
  - between → **`proposed`** (visible as low-confidence, de-emphasised, *not* in
    the primary search index).
- **`warning:*` exception**: age-gate/legal tags **cannot be confirmed by votes
  alone** — a Mod must confirm (`roles-and-content-model.md` §B.1.2). Votes may
  *surface* a suggestion, but the legal gate is never crowd-decided.
- **Mod override**: a Mod can `lock` a tag as present or removed regardless of
  score (vandalism, legal, edit-wars); `locked` ignores further votes until
  unlocked.

---

## 3. Reputation & incentives

- When a tag reaches **`confirmed`**, its proposer and majority-side voters gain
  `tagger_rep`; when **`rejected`**, the proposer loses rep (and wrong-side voters
  a little). This is the E-Hentai flywheel: good taggers accrue weight, bad ones
  decay toward irrelevance.
- Reputation feeds **gamification** (`community-features.md` §D) — tagging points
  are **quality-gated and settle after confirmation**, clawed back on later
  reversal, same anti-farming guard used for uploads and wiki edits.
- Gate to vote/propose: verification **L1**; a minimum rep unlocks higher weight
  and the ability to propose `is_restricted`-adjacent tags.

---

## 4. Anti-abuse (brigading is the real threat)

Crowd tagging invites vote manipulation; defences:

- **Reputation-weighted, capped** votes blunt sockpuppet swarms (many fresh
  accounts ≈ many 1× votes, still bounded).
- **Rate limits** per user on proposing/voting; new-account throttle.
- **Brigade detection**: a burst of correlated votes on one (series, tag) from
  low-rep/new/similar-IP accounts in a short window is **dampened** and flagged to
  the mod queue rather than counted at face value.
- **Reversible & audited**: every vote and state change is recorded; mod `lock`
  is the backstop; `rejected` tombstones stop re-spam churn.
- All of this routes through the existing `reports` / `moderation_actions` core.

---

## 5. Effect on search & browse

- Only **`confirmed`** tags populate the primary search/filter index and the
  denormalised `series.tag_ids[]` GIN array (`roles-and-content-model.md`).
  `proposed` tags show as low-confidence on the page but don't pollute results;
  `rejected`/`locked-off` never appear.
- Tag **counts** (`usage_count`) count confirmed assignments only — so
  browse-by-tag and autocomplete reflect consensus, not noise.
- This is what makes **tag-first search** (`specs/search.md` §1) trustworthy: the
  first-class axis is *community-validated*, not one uploader's guess.

---

## 6. Roadmap fit

- **Phase 2** (catalog/search): proposing + voting + confirm/reject thresholds +
  confirmed-only indexing — tagging has to be trustworthy before tag-first search
  ships on top of it.
- **Phase 4** (gamification): tagger reputation → weight and point rewards, once
  the reputation system exists. Until then votes are unweighted (1× each) with
  mod backstop.

**Test checklist**: propose seeds +1; confirm at threshold makes it searchable;
reject hides + tombstones against re-spam; `warning:*` never confirms on votes
alone; mod `lock` overrides score both ways; reputation raises weight within the
cap; brigade burst is dampened + flagged; points settle post-confirm and claw
back on reversal; only confirmed tags hit the GIN index and counts.
