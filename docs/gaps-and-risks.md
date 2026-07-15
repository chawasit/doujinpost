# Gaps & Risks — what the plan is missing

The specs so far are solid on *mechanism* (storage, watermark, roles, tags,
recs) and nearly silent on the things that actually decide whether this platform
survives contact with reality: **law, money, and cost.** This doc is the
backlog of what's missing, ordered by how badly it hurts.

---

## Tier 0 — existential (get this wrong and there is no platform)

### 1. Illegal-content detection is completely absent — and a tag actively invites it
A user-generated adult manga site **will** receive CSAM and drawn
underage/"loli/shota" content. There is currently **no** hash-matching
(PhotoDNA/NCMEC), no scanning pipeline, no mandated reporting flow
(US 18 U.S.C. §2258A requires providers to report to NCMEC), and no
takedown-preservation path. Worse: `specs/roles-and-content-model.md` lists
`warning:underage` as a supported **tag implication** — i.e. the taxonomy is
built to *file* content that is outright illegal in the UK, Canada, Australia,
and prosecutable under the US PROTECT Act. That's not a feature, that's Exhibit A.
**Needed**: inbound CSAM hashing + human-review queue, hard-blocked content
classes (not tag-gated — *refused*), NCMEC reporting, evidence preservation, and
a legal content policy that names what is banned outright.

### 2. No age verification / 2257 record-keeping
Adult content implies real obligations: age-gating that actually verifies age
(not a "yes I'm 18" checkbox) where required, and for content depicting real
people, US 18 U.S.C. §2257 record-keeping. The plan has a `rating` field and a
hand-wave to "age gating." That's not compliance.

### 3. Payments — DESCOPED (open-source, nonprofit, no monetization)
**Resolved by decision, not by build.** The project is open-source and nonprofit
with **no payments/subscriptions/tips/ads/payouts**, so the
adult-content-processor trap (Visa/MC/PayPal/Stripe bans), payout tax rails, and
EU/UK VAT all **go away**. Good — that was the ugliest commercial problem.

What replaces it, though, is a **sustainability** problem, not a nonexistent one:
- **Hosting/egress cost** is now funded by donations/grants, which makes the
  Tier-1 watermark-CDN cost bomb (#5) a *budget* threat, not just an engineering
  one. Strong reason to watermark downloads/exports only, not in-app reads.
- Even donations touch a processor eventually (Stripe/PayPal/OpenCollective for a
  *nonprofit* is far less fraught than *adult commerce*, but "adult site" can
  still trip provider ToS) — keep donation handling **off-platform** (a fiscal
  host / OpenCollective / Ko-fi) so the app itself never processes money.
- Add **open-source project hygiene** the plan hasn't: a LICENSE (AGPL vs.
  permissive — AGPL keeps hosted forks open), CONTRIBUTING, CoC, a governance
  model, self-host docs, and a clear line on who is the legal operator of the
  flagship instance (a nonprofit entity needs to actually exist to hold
  safe-harbour and receive notices).

> Nonprofit status does **not** grant legal immunity. Tier-0 items 1, 2, 4 below
> apply in full — arguably harder, because a volunteer org has no compliance
> budget. Build them cheap, don't skip them.

### 4. GDPR/CCPA vs. your own watermark and audit logs — a direct contradiction
The plan promises **right-to-erasure**-shaped privacy *and* forensic
`download_grants` that trace leaks *and* append-only audit/DMCA logs. These fight
each other: erasing a user vs. retaining the evidence that attributes their
leaks. Nobody reconciled retention windows, lawful basis, DPA, data export, or
cookie consent. Pick the boundary deliberately or a regulator will pick it for you.

---

## Tier 1 — will bankrupt or break you architecturally

### 5. Per-user watermarking defeats the CDN — the cost bomb
Every download is watermarked **per user**, so it is **uncacheable at the CDN
edge**. Manga is bandwidth-heavy and every read becomes an origin hit through the
CPU-heavy Python DCT worker. `PLAN.md` §3.4 waves at "cache-friendly at the shard
level," but the honest math is: egress + per-download compute scales with reads,
linearly, forever. Model this cost *now* — it may force a hybrid (cache clean
renditions at edge; watermark only downloads/exports, not in-app reads).

### 6. Your watermark has a known break you didn't defend: collusion
Traitor-tracing's canonical attack is **collusion** — several users diff their
copies to locate and scrub the mark. RS + fountain gives you *robustness*, not
*collusion resistance*. The literature's answer is **Tardos / collusion-secure
fingerprinting codes**. Without them, a small colluding group defeats attribution
entirely. Right now the scheme is robust to a lazy pirate and naked to an
organized one.

### 7. No backups, DR, observability, or capacity story
No mention of Postgres PITR/backups, object-store durability/replication,
disaster recovery, monitoring, alerting, SLOs, incident response, or on-call. A
content platform that loses the CAS store loses everything. Storage also grows
unbounded with no capacity/retention model.

---

## Tier 2 — the product doesn't actually work yet

### 8. Search is marked "if needed" — it's core, and it's CJK/Thai
For a catalog site, search *is* the product, not a Phase-later maybe. And the
hard part is unaddressed: **Japanese and Thai tokenization** (no spaces in Thai;
romaji/kana/kanji variance in JP), typo tolerance, and faceted filtering. Postgres
FTS won't do CJK/Thai well out of the box.

### 9. Uploads assume loose pages; the real world ships archives
Doujin is distributed as **CBZ/CBR/ZIP/PDF/EPUB**, not page-by-page uploads.
There's no bulk/archive ingest, no batch UX, no "updated a chapter with better
scans" **content versioning**, and no cross-user repost/duplicate-series
detection beyond per-page pHash.

### 10. The reader — your actual core product — is one line
Manga needs RTL/LTR/**vertical webtoon** modes, double-page spreads, preloading,
zoom/fit. Novels need font/size/theme, TTS, and real EPUB features. This is what
users touch every second and it's specified in a single clause.

### 11. i18n/l10n is barely there on a translation site
The site's entire premise is translation, yet there's no UI localization, no
per-user language, no translated metadata, no CJK/Thai typography/RTL story.

### 12. Scanlation reality isn't modeled
Translation **groups/teams** as first-class, per-chapter staff credits
(translator/cleaner/typesetter), joint releases, credit pages. "Writer role +
translator grant" is too thin for how scanlation actually works.

---

## Tier 3 — plumbing you forgot exists

- **Trust & Safety depth**: ban evasion / sockpuppets / sybil resistance,
  **appeals process**, moderator-abuse audit, anti-scraping (your whole catalog
  is scrapable; the watermark is your only deterrent and it's bypassed for
  in-app reads).
- **Email deliverability**: SPF/DKIM/DMARC, CAN-SPAM unsubscribe, digest vs.
  transactional separation.
- **Account lifecycle**: deletion/export/deactivation, and the orphan problem
  (a translator deletes their account — what happens to series others follow?).
- **Accessibility**: WCAG, screen readers (novels especially), keyboard nav.
- **SEO/discovery** vs. adult-content indexing constraints; sitemaps, social
  cards.
- **Creator & business analytics**: views/reads/retention/revenue dashboards
  (Phase 6 mentions cost dashboards only).
- **CI/CD, staging, migration strategy, feature flags, load testing** — named
  nowhere concrete.

---

## Where you *over*-built

- **Three overlapping comms systems**: `comments`, series `notices`, and the
  `webboard`. Justify why that's three schemas and not two — notices vs. a pinned
  comment thread is a thin distinction.
- **The watermark is genuinely impressive and possibly premature.** DCT
  spread-spectrum + RS + fountain codes is a lot of hard DSP to build in Phase 2
  when you have no users, no content, and (per above) no CSAM pipeline. It's the
  most polished part of the plan and arguably the least urgent. Ship the catalog,
  the reader, and the T&S/legal spine first; the fancy fingerprint can wait.

---

## Suggested reprioritisation

The current roadmap builds the cool crypto before the things that keep you out of
court. With payments descoped (nonprofit), a saner order:

1. **Legal/T&S spine first** — content policy, CSAM detection + NCMEC reporting,
   age-gating, DMCA (already planned), GDPR retention model. Cheap-by-design,
   because there's no compliance budget.
2. **Core product** — archive ingest, a real reader, real search, i18n.
3. **Sustainability & OSS hygiene** — LICENSE/governance/self-host docs, a real
   nonprofit operator entity, off-platform donations, and a hosting **cost model**
   (egress + watermark compute) the donation budget can actually carry.
4. **Ops** — backups/DR/observability.
5. **Then** the watermark — with **collusion-secure codes** and a **CDN cost
   model**, and probably only on *downloads/exports*, not in-app reads.
