# Spec — Roles & Content Model

Status: draft · Owner: platform · Depends on: `PLAN.md` §4–6,
`specs/authentication.md`, `specs/user-verification.md`

Two things this spec pins down:

- **Part A — Roles**: the five roles, what each can do, how roles are granted and
  scoped.
- **Part B — Content model**: the `Category → Series → [Chapters · Notice ·
  Extra]` hierarchy, each object's states, and the creation/read flow.

---

# Part A — Roles & permissions

Roles are **additive** and some are **scoped**. A user holds a *set* of roles
(e.g. a Writer can also be a Moderator), plus scoped grants (a Publisher Rep is
tied to a specific publisher and its catalog; content ownership is tied to the
series a Writer created). Authorisation = "does any of this user's roles/grants
allow this action on this object?"

## A.1 The five roles

1. **Reader** — default on signup. Consumes content and participates socially.
2. **Writer (write / translate)** — creates and maintains series and chapters.
   Two flavours share one role, distinguished per-series by `origin`:
   - *original* (`write`) — own work;
   - *translation* (`translate`) — must attribute the source work/author and
     language. Same permissions either way.
3. **Moderator** — enforces community rules across the site: reports, comments,
   forum, user sanctions. No site configuration, no role assignment.
4. **Admin** — superuser: everything a Moderator can do, plus users/roles, site
   config, storage/lifecycle, DMCA final decisions, watermark-key custody.
5. **Licensed publisher representative** — a *verified* agent of a rights holder.
   Scoped to that publisher's catalog: claims ownership of titles, issues
   licensed-takedowns, and acts as a trusted DMCA claimant. **No** general
   moderation power over unrelated content.

## A.2 Permission matrix

`own` = only objects the user created/owns · `cat` = only the rep's claimed
catalog · `L1/L2` = required verification level (`specs/user-verification.md`) ·
`—` = not allowed.

| Capability | Reader | Writer | Moderator | Admin | Publisher Rep |
|---|---|---|---|---|---|
| Browse / read published | ✔ | ✔ | ✔ | ✔ | ✔ |
| Download (watermarked) | ✔ L1 | ✔ | ✔ | ✔ | ✔ |
| Comment / forum post | ✔ L1 | ✔ | ✔ | ✔ | ✔ |
| Report content | ✔ | ✔ | ✔ | ✔ | ✔ |
| File a DMCA notice | ✔¹ | ✔¹ | ✔¹ | ✔ | ✔ (streamlined, `cat`) |
| Create series / upload chapters | — | ✔ L2 | — | ✔ | —² |
| Edit / delete series & chapters | — | ✔ own | ✔ (any, audited) | ✔ | —² |
| Post a series **Notice** | — | ✔ own | ✔ any | ✔ | ✔ cat |
| Publish **Extra** | — | ✔ own | — | ✔ | —² |
| Moderate comments/forum (hide/lock/del) | — | — | ✔ | ✔ | — |
| Manage webboard (pin/lock/move) | — | — | ✔ | ✔ (+ boards) | — |
| Warn / suspend users | — | — | ✔ (temp) | ✔ (any) | — |
| Take content down pending review | — | — | ✔ | ✔ | ✔ cat (auto-escrow) |
| DMCA triage / decision | — | — | triage | decide | claimant only |
| Claim series ownership (rights) | — | — | — | approve | ✔ request, `cat` |
| Manage licensed catalog / official releases | — | — | — | ✔ | ✔ cat |
| Assign roles | — | — | — | ✔ | — |
| Site config / storage / keys / flags | — | — | — | ✔ | — |
| View analytics | — | own | moderation | all | cat |

¹ DMCA intake is open to **anyone**, including non-account external claimants —
it is a legal process, not a role privilege. Rows above just note in-app filing.
² A Publisher Rep is a *rights/takedown* actor, not an uploader by default. If a
publisher also distributes official content here, an Admin additionally grants
that rep the **Writer** role scoped to the publisher account — the two powers
stay separate and are granted separately.

## A.3 Granting & scoping roles

| Role | How obtained |
|---|---|
| Reader | automatic at signup |
| Writer | **self-serve** upgrade once verification **L2** is met and the uploader ToS (watermark disclosure, DMCA policy) is accepted |
| Moderator | appointed by an Admin |
| Admin | appointed by an Admin (first admin via seed/bootstrap) |
| Publisher Rep | **application → Admin verification** of the publisher affiliation (business/rights proof), then granted `publisher_id`-scoped |

- **Ownership grants** are created automatically when a Writer creates a series
  (`content_grants(series, user, role=owner)`), and can be shared (co-authors,
  a translation group) by the owner.
- **Publisher Rep scope** is `publisher_id`; their catalog is the set of series
  linked to that publisher (via approved rights claims). Actions outside `cat`
  are denied even though the role is powerful within it.
- **Least privilege**: destructive/global capabilities (role assignment, config,
  keys) are Admin-only; every moderator/admin/rep action writes an
  `audit_log` / `moderation_action` row (`PLAN.md` §6).

## A.4 Data (roles & grants)

```sql
roles (user_id, role text,             -- 'reader'|'writer'|'moderator'|'admin'|'publisher_rep'
       scope_publisher_id uuid null,    -- set for publisher_rep
       granted_by uuid, granted_at timestamptz)

content_grants (series_id uuid, user_id uuid,
                grant text,             -- 'owner'|'coauthor'|'translator'
                created_at)

publishers (id uuid pk, name text, verified_at timestamptz, verified_by uuid)
rights_claims (id, publisher_id, series_id, state, evidence, decided_by, decided_at)
```

---

# Part B — Content model & flow

```
Category  ──<  Series  ──<  Chapter   (the readable units, ordered)
                   │
                   ├──<  Notice       (announcements pinned to the series)
                   └──<  Extra        (bonus/supplementary units: omake, art, …)
```

## B.1 Classification — categories & tags

Two complementary systems, deliberately kept separate:

| | **Categories** | **Tags** |
|---|---|---|
| Purpose | curated **navigation spine** | fine-grained **folksonomy + facets** |
| Shape | shallow **tree** (category → sub-category) | flat, **namespaced**, high-cardinality |
| Who edits | Admin/Mod curated | Writer-applied, Mod-canonicalised |
| Cardinality per series | a few | many |
| Examples | *Manga → Romance → Yuri* | `theme:isekai`, `creator:circle_xyz`, `warning:gore` |

They're linked, not redundant: a navigation category may be **backed by a tag**
(e.g. the *Romance* category maps to `genre:romance`) so browsing the tree and
filtering by tag stay consistent.

### B.1.1 Categories (with sub-categories & descriptions)

A **Series belongs to one or more categories** (many-to-many). Categories form a
**shallow tree** via `parent_id` — a top-level category has sub-categories one or
two levels deep (e.g. *Manga → Doujinshi → Parody*). Each category carries
**presentation metadata** so category landing pages are real, described sections
rather than bare labels.

```sql
categories (
  id uuid pk,
  kind text,                   -- 'type' (manga|novel) | 'genre'
  slug text unique,
  title text,
  description text,            -- shown on the category landing page (markdown-lite)
  parent_id uuid null,         -- null = top level; else a sub-category
  cover_blob bytea null,       -- optional banner/icon
  backing_tag_id uuid null,    -- optional: keep in sync with a tag (B.1.2)
  sort_order int,
  is_listed bool default true, -- hide from nav without deleting
  created_at timestamptz )

series_categories (series_id uuid, category_id uuid,
                   primary_flag bool)   -- one primary category per series for canonical breadcrumb
```

- **type** is single-valued per series (a series is manga *or* novel); **genre**
  categories are multi.
- Creating/renaming a top-level category or sub-category is an **Admin/Mod**
  action; `description` and `cover_blob` let each (sub-)category read as a curated
  section. Sub-category inherits nothing implicitly — membership is explicit rows,
  but the tree drives breadcrumbs and faceted nav.
- `primary_flag` picks the one category used for the canonical URL/breadcrumb when
  a series sits in several.

### B.1.2 Tags

Tags are **namespaced** labels — a booru-style facet system that scales to the
long tail (characters, circles, source franchises, content warnings) that a
curated tree can't.

```sql
tags (
  id uuid pk,
  namespace text,              -- see below
  name text,                   -- canonical, normalised (lowercase, _-joined)
  description text,            -- wiki-style blurb shown on tag page / hover
  state text,                  -- 'active' | 'pending' | 'blocked'
  is_restricted bool,          -- only mods may apply (e.g. warning:*)
  usage_count int,             -- denormalised popularity, async-maintained
  created_by uuid, created_at timestamptz,
  unique (namespace, name) )

tag_aliases (alias text unique, tag_id uuid)         -- 'genderswap' -> gender_swap
tag_implications (tag_id uuid, implies_tag_id uuid)  -- shota -> warning:underage
series_tags (series_id uuid, tag_id uuid, added_by uuid, created_at)
```

**Namespaces** (the tag's `namespace`):

| Namespace | Meaning | Applied by |
|---|---|---|
| `genre` | action, romance, comedy … | Writer |
| `theme` | isekai, school, revenge … | Writer |
| `character` | named characters | Writer |
| `creator` | author / artist / **circle** | Writer |
| `source` | parodied franchise (for doujin/fan works) | Writer |
| `language` | text/scanlation language | Writer |
| `meta` | colored, long-strip, official, fan-translation | Writer |
| `warning` | gore, nsfw sub-types → **drives age-gating** | **Mod-restricted** |

**Mechanics**

- **Aliases** collapse synonyms/typos to one canonical tag, so filtering and
  counts don't fragment. Applying an alias resolves to its canonical tag.
- **Implications** auto-add entailed tags (`shota → warning:underage`,
  `long_strip → meta:webtoon`). Implication chains are resolved transitively at
  apply time and re-checked when implications change.
- **Moderation**: Writer-created tags enter `pending`; a Mod canonicalises,
  aliases, merges, or `blocked`s them. `warning:*` is `is_restricted` — writers
  can *request* one but only Mods confirm it, because these tags gate adult
  content and legal exposure (`PLAN.md` §7). Blocked tags are unappliable and
  hidden.
- **Search/browse**: include/exclude tags with AND/OR, faceted counts, and
  combine with the category tree. Store `series_tags` normalised **and** keep a
  denormalised `tag_id[]` on `series` with a **GIN index** for fast
  include/exclude filtering; `usage_count` powers popularity and autocomplete.
- **Tag pages**: each tag has a `description` (wiki blurb) and lists its series,
  aliases, and implications — same pattern categories use for their landing pages.

## B.2 Series

The work itself — a manga title or a novel title. Owned by a Writer (see
`content_grants`).

```sql
series (
  id uuid pk, title text, type text[manga|novel],
  origin text,                 -- 'original' | 'translation'
  source_ref text null,        -- for translations: original title/author/lang
  language text, description text, cover_blob bytea,
  status text,                 -- 'ongoing'|'complete'|'hiatus'|'licensed'
  state  text,                 -- lifecycle, see B.5
  rating text,                 -- content rating / age gate
  publisher_id uuid null,      -- set if under a licensed publisher
  created_by uuid, created_at timestamptz )
```

- `status` = author-facing publication status; `state` = platform lifecycle
  (moderation/DMCA). They're independent axes.
- A series aggregates readable **Chapters**, plus **Notices** and **Extras**.

## B.3 Chapter (the readable unit)

The actual content: manga pages or novel text. Ordered within a series.

```sql
chapters (
  id uuid pk, series_id uuid, number numeric,  -- allows 10.5 interludes
  title text, kind text,                        -- inherited manga|novel
  is_extra bool default false, extra_type text null,  -- see B.4
  state text,                                   -- draft|pending|published|hidden|dmca_disabled|deleted
  release_at timestamptz null,                  -- scheduled publish
  created_at )
-- page/body blobs referenced via manifest_entries (PLAN.md §6), CAS-deduped
```

Reading a chapter goes through the **download broker** (`PLAN.md` §3.4) so every
delivered copy is watermarked.

## B.4 Extra

Supplementary content — modelled as a **chapter with `is_extra = true`** rather
than a separate table, so it reuses upload, storage, watermarking, and reader
code. `extra_type ∈ { omake, afterword, illustration, cover, sidestory, misc }`.
Extras are excluded from main chapter numbering and can be surfaced in a separate
"Extras" tab.

## B.5 Notice

Lightweight text announcements pinned to a series — **not** watermarked content,
no blobs.

```sql
notices (id, series_id, author_id, type, body, pinned bool, state, created_at)
--  type ∈ { author_note, hiatus, recruitment, licensed_notice, mod_notice, system }
```

- Writers post `author_note`/`hiatus`/`recruitment` on their own series.
- `licensed_notice` is auto-created when a series enters `licensed_removed`
  (below), showing a "this title is now officially licensed — support the
  official release" message in place of the chapters.
- Mods/Admins can post `mod_notice`; `system` is platform-generated.

## B.6 Lifecycle / states

**Series & Chapter state machine** (chapters inherit constraints from series):

```
draft ──▶ pending_review ──▶ published ──▶ hidden ──▶ (published | deleted)
  │            │                 │
  │            └── rejected      ├──▶ dmca_disabled ──▶ (published | deleted)   [blob escrow]
  │                             └──▶ licensed_removed                          [publisher action]
  └──▶ deleted (soft)
```

- `pending_review` is **conditional**: on for new/low-trust Writers (post-less
  pre-moderation), off for established ones (post-moderation) — a policy flag,
  not hardcoded.
- `hidden` = moderator hold pending decision; reversible.
- `dmca_disabled` = blocked via `specs`/`PLAN.md` §5 DMCA; blob kept in escrow,
  not hard-deleted, pending counter-notice.
- `licensed_removed` = a verified Publisher Rep asserted rights (or bought the
  license); chapters go dark, a `licensed_notice` replaces them. Distinct from a
  DMCA strike — it's a legitimisation path, not an infringement penalty.
- `deleted` = soft delete; blobs GC'd when refcount hits zero (`PLAN.md` §2.1).

## B.7 The main content flow

**Authoring (Writer)**
```
1. Create/choose Series  → set type, origin (write|translate), categories, tags
2. Add Chapters          → upload manga pages / novel text → CAS ingest + transcode
3. (optional) add Extras → omake, afterword, illustrations …
4. (optional) post Notice → author note / hiatus / recruitment
5. Publish                → draft → [pending_review if gated] → published
                            (chapters may be scheduled via release_at)
```

**Reading (Reader)**
```
Browse Category ▶ open Series ▶ read Chapters (watermarked delivery)
                              ▶ see Notices (pinned announcements)
                              ▶ read Extras (separate tab)
                              ▶ comment / report
```

**Rights / takedown lanes**
```
Publisher Rep ▶ claim rights on Series (→ Admin approves) ▶ licensed_removed + licensed_notice
Anyone        ▶ DMCA notice ▶ Mod triage ▶ Admin decision ▶ dmca_disabled / restore
Moderator     ▶ report queue ▶ hidden / sanction ▶ audit_log
```

---

## C. Roadmap fit

- Roles/grants core + Reader/Writer + content model land in **Phase 0–1**
  alongside auth and upload.
- Moderator tooling with the report queue lands in **Phase 4** (comments/forum).
- Publisher Rep + rights-claims + `licensed_removed` land with **Phase 5** (DMCA),
  since they share the takedown/escrow machinery.

**Test checklist**: role additivity (writer+mod); publisher-rep scope denial
outside `cat`; writer self-serve gated on L2; ownership grant on series create +
co-author share; pending_review gate toggles by trust; DMCA vs licensed_removed
produce different notices; extra reuses chapter pipeline; soft-delete GC;
sub-category tree breadcrumbs + one `primary_flag` per series; category
description/cover render on landing page; tag alias resolves to canonical; tag
implication auto-adds transitively; `warning:*` restricted to mods and drives
age-gate; tag include/exclude filter hits the GIN index; `usage_count` updates.
