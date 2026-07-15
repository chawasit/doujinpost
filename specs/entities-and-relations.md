# Spec — Entities & Relations (IMDB-style credits + wiki graph)

Status: draft · Owner: platform · Depends on: `PLAN.md` §6,
`specs/roles-and-content-model.md`, `specs/community-features.md`, `specs/search.md`

An IMDB/AniList/MangaUpdates-style knowledge layer over the catalog:

- **Part A — Entities**: real-world people & organisations (authors, illustrators,
  story writers, circles, publishers, magazines) credited on series — first-class,
  with their own pages.
- **Part B — Relations**: a **wiki-style graph** connecting series (sequel,
  spin-off, adaptation, shared universe) and entities (aliases, group members).
- **Part C — The wiki**: community editing with revision history, moderation, and
  trust-gating that maintains A and B.

This is distinct from platform accounts: a **credited author** is a real-world
person who may have no account; a **creator profile**
(`specs/community-features.md` §A) is a *platform user* (usually a translator).
The two can be **linked** when a creator claims their entity.

---

## Part A — Entities (IMDB-style)

```sql
entities (
  id uuid pk,
  type text,              -- 'person' | 'circle' | 'company' | 'magazine' | 'brand' | 'character'
  name text,              -- primary/display name
  alt_names text[],       -- romanisations, native script, aliases, maiden/pen names
  bio text, links jsonb, image_blob bytea,
  linked_user_id uuid null,  -- set when a platform user claims this entity (verified)
  state text,             -- 'active' | 'pending' | 'merged'
  merged_into uuid null,  -- dedupe target
  created_at )

entity_credits (
  series_id uuid, entity_id uuid,
  role text,              -- see role list below
  note text,              -- e.g. "chapters 1–40", "cover only"
  primary key (series_id, entity_id, role) )
```

**Roles** — manga/novel credit reality, *not* film. The key split the request
named explicitly:

| Role | Meaning |
|---|---|
| `story_writer` (原作 *gensaku*) | writes the story |
| `illustrator` / `artist` (作画 *sakuga*) | draws it — **often a different person** |
| `author` | single creator who both writes and draws (use when undivided) |
| `original_creator` | source-work creator (for adaptations) |
| `character_design`, `cover_artist`, `editor` | supporting credits |
| `circle` (doujin group), `publisher`, `magazine`, `label` | org credits |

- **Entity pages** (IMDB-style): `/entity/:id` → bio + **"bibliography"**: every
  series they're credited on, grouped by role, sortable by year/popularity. This
  is the "browse everything by illustrator X" experience Nekopost lacks.
- **Claim & link**: a platform user can claim an entity (e.g. an original author
  who joins); Admin verifies → `linked_user_id` set, and the creator profile and
  entity page cross-link. Unclaimed entities are just catalog data.
- **Dedup/merge**: entities arrive messy (romanisation variants). Mods **merge**
  duplicates (`merged_into`); credits repoint; `alt_names` absorb the variants.
  Search embeds `name + alt_names` so variants still resolve (`specs/search.md`).

---

## Part B — Relations (the wiki graph)

Directed, typed edges — a knowledge graph over series and entities.

```sql
series_relations (
  from_series uuid, to_series uuid,
  relation text,          -- see below (directed; inverse auto-implied)
  primary key (from_series, to_series, relation) )

entity_relations (
  from_entity uuid, to_entity uuid,
  relation text )         -- 'alias_of' | 'member_of' | 'imprint_of' | 'collab_with'
```

**Series relation types** (with auto-inverse):

| Relation | Inverse |
|---|---|
| `sequel` | `prequel` |
| `side_story` / `spin_off` | `parent_story` |
| `adaptation_of` (e.g. manga of a novel) | `adapted_into` |
| `alternate_version` / `remake` | (symmetric) |
| `shares_universe` | (symmetric) |
| `same_work_other_language` | (symmetric) |

- Writing one edge **auto-creates its inverse** so the graph stays consistent
  (add `A sequel-of B` ⇒ `B prequel-of A`).
- Powers a **relations panel** on the series page ("Sequels, Spin-offs, Adapted
  from…") and feeds "more like this" — **explicit relations outrank embedding
  guesses** in `specs/search.md` §5.
- `same_work_other_language` links our own translations of one source together —
  central to a translation site (jump between the Thai and English editions).

---

## Part C — The wiki (community editing)

Entities, relations, alt-titles, descriptions, and credits are **community-
maintained** (the AniList/MangaUpdates model), not admin-only — otherwise the
graph never gets populated. That demands versioning and moderation.

```sql
wiki_revisions (
  id uuid pk,
  target_type text,       -- 'series' | 'entity' | 'series_relation' | 'entity_credit'
  target_id uuid,
  editor_id uuid,
  diff jsonb,             -- field-level before/after
  state text,             -- 'applied' | 'pending' | 'reverted'
  note text, created_at )
```

- **Every edit is a revision** — full history, attribution, one-click **revert**;
  vandalism is cheap to undo. Append-only; the live row is the latest applied
  revision.
- **Trust-gating** (reuses verification level + gamification reputation):
  - low-trust/new users → edits enter `pending`, a mod/trusted user approves;
  - trusted users → edits apply immediately, patrolled after the fact;
  - `is_restricted` fields (age-gate-relevant flags, `state`) stay mod-only.
- **Moderation & audit**: all wiki actions route through `moderation_actions`;
  edit wars → lock the field/page; repeat vandals lose edit rights.
- **Incentive tie-in**: wiki contributions earn **gamification** points/badges
  (`specs/community-features.md` §D) — but **quality-gated** (points settle after
  the revision survives patrol, clawed back on revert) so we don't farm-reward
  vandalism. Same guard as upload points.
- **Attribution page**: each entity/series shows contributors, like a wiki
  history tab.

---

## D. How it all connects

- **Search** embeds entity names + relation context, so "by illustrator X" and
  "same universe" cluster even before edges exist; explicit relations then
  sharpen "more like this" (`specs/search.md`).
- **Creator profiles** (platform users) link to **entities** (real-world credits)
  via `linked_user_id` — a translator's account and an author's catalog identity
  are different objects that can point at each other.
- **Gamification** rewards wiki curation; **moderation** patrols it; **roles**
  decide who can apply vs. propose.

---

## E. Roadmap fit

- **Phase 2**: entities + `entity_credits` + entity pages + `series_relations`
  panel (static, admin/trusted-seeded) — immediate IMDB-style value, feeds search.
- **Phase 3–4**: open the **wiki** (revisions, pending/approve, trust-gating,
  points tie-in) once moderation and gamification exist to keep it clean.

**Test checklist**: story_writer vs illustrator credited separately on one
series; entity page lists full bibliography by role; claim links entity↔user;
merge repoints credits + preserves alt_names; writing a `sequel` auto-creates
`prequel`; relations panel renders; low-trust edit goes `pending`, trusted edit
applies; revert restores prior revision; wiki points settle post-patrol and claw
back on revert; age-gate/state fields stay mod-only.
