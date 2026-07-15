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

## B.1 Category

Top-level organisation for discovery. A **Series belongs to one or more
categories** (many-to-many), so a work can sit under both a content-type and a
genre facet.

- Two facet kinds: **type** (`manga`, `novel`) and **genre/tag** (action,
  romance, doujin-circle, fan-translation, …). Type is single-valued per series;
  genres are multi.
- Curated by Admins/Mods (creating a top-level category is an Admin action);
  free-form **tags** on a series are Writer-set and moderatable.
- `categories(id, kind[type|genre], slug, title, parent_id null)` — allows a
  shallow tree (e.g. genre → subgenre). `series_categories(series_id, category_id)`.

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
produce different notices; extra reuses chapter pipeline; soft-delete GC.
