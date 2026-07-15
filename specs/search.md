# Spec — Semantic Search (symmetric embeddings + LEANN, hybrid)

Status: draft · Owner: platform · Depends on: `PLAN.md` §2/§6,
`specs/roles-and-content-model.md` (tags/state), `specs/entities-and-relations.md`,
`docs/gaps-and-risks.md` #8

Resolves the open search question (`PLAN.md` §9) — and turns a weakness into a
strength. A Thai translation site's search must handle **Thai + Japanese +
English in one box**, tolerate typos and paraphrase, and power "more like this."
Keyword FTS alone can't (Thai has no word boundaries; JP mixes kana/kanji/romaji).
**LLM embedding search sidesteps tokenisation entirely** and is cross-lingual by
construction.

---

## 1. Approach: hybrid (lexical ⊕ vector), symmetric embeddings

**Retrieval priority — tags first, then name, then meaning.** This is the
E-Hentai/nhentai search model and it's deliberate:

1. **Tags are the first-class axis.** A namespaced, **community-voted**
   (`specs/tag-voting.md`) tag query is the most precise, most trusted signal —
   `artist:foo language:thai theme:isekai -warning:gore`. Tag filters are
   **structured predicates**, not ranked guesses: they include/exclude with
   AND/OR/NOT over the confirmed-tag GIN index and *constrain* everything below.
2. **Name** — titles, alt/romaji titles, entity names (lexical/BM25). Exact
   things users type verbatim.
3. **Meaning** — symmetric embeddings infer *intent/goal* for natural-language
   queries and power "more like this", when tags+name don't capture what the user
   means.

Under the hood these are the same two retrievers, fused — but **tags gate and
outrank** name, which outranks semantic:

- **Tag predicates** (structured): the hard include/exclude frame + a strong
  ranking boost. First citizen.
- **Lexical** (Postgres FTS / BM25): exact titles, entity names, tag slugs, IDs.
  Precise on verbatim input; blind to meaning and Thai/CJK segmentation.
- **Vector** (embeddings + ANN): semantic, paraphrase-tolerant, typo-tolerant,
  and **cross-lingual** — a Thai query finds an English-titled series because the
  multilingual model maps them near each other. The meaning-inference layer.

**Symmetric embeddings**: one model embeds *both* the query and the content into
the **same space**, so query↔item and item↔item are directly comparable. This is
exactly right for our two dominant search shapes:

- **"More like this"** (item↔item): series → nearest series. Symmetric is the
  native operation.
- **Descriptive / natural-language queries** ("wholesome slice-of-life with
  cooking"): query embedding → nearest series/description embeddings.

> Caveat we accept: pure-symmetric can trail asymmetric models on *very short
> keyword → long doc* retrieval. That's exactly what the **lexical** half covers,
> so the hybrid is robust where symmetric alone would wobble. Fuse with
> **Reciprocal Rank Fusion (RRF)** — no score calibration needed.

---

## 2. The index: LEANN (low-storage) — where it fits, honestly

**LEANN** (UC Berkeley/CUHK/AWS/UC Davis, arXiv 2506.08276) is a low-storage ANN
index: an aggressively pruned HNSW-style proximity graph that **discards stored
embeddings and recomputes them on the fly at search time**, keeping only a tiny
fraction of hub-node vectors cached. Storage overhead **< 5% of raw** — ~50×
smaller than a full HNSW (their result: ~6 GB vs ~201 GB on 60 M chunks).

**Why it suits us**: a donation-funded nonprofit that already optimises storage
(`PLAN.md` §2). If we do deep semantic search over **novel full text, wiki pages,
and every chapter** — a genuinely large text corpus — storing full vectors is a
real cost, and LEANN nearly eliminates it.

**The honest trade-off** (don't paper over it): LEANN moves cost from *storage*
to *query-time embedding recompute*. That requires:
- the **embedding model hosted hot** at query time (GPU or fast CPU), and
- tolerance for extra per-query latency vs. a stored-vector index.

So the recommendation is **tiered by corpus size**:

| Corpus | Index | Rationale |
|---|---|---|
| **Series-level** (~10²–10⁵ items: title/alt-titles/desc/tags/entities) | stored vectors — **pgvector HNSW** | small enough that vectors are cheap; lowest latency for the hottest path ("more like this", main search) |
| **Deep text** (novel chapters, wiki, all comments — large & growing) | **LEANN** | storage dominates here; recompute cost amortised, results are the "search inside" long tail |

Lead with pgvector for the main box (simple, fast, transactional with Postgres),
add LEANN for the deep-text tier where its storage win actually pays off. Both
speak the same embedding model, so they compose.

---

## 3. Embedding model (self-hosted, multilingual)

- **Multilingual model** so Thai/JP/EN share a space — e.g. BGE-M3,
  multilingual-e5-large, or LaBSE class. Cross-lingual retrieval is the headline
  feature for a translation community.
- **Self-hosted inference** (no per-query API cost — nonprofit): a small
  inference service (ONNX/GGUF on CPU, or a shared GPU) exposed to the search API
  and to LEANN's recompute path. Query embeddings are **cached** (Redis) since
  popular queries repeat.
- Pin the model version; a model change means a **full re-embed + reindex** —
  budget for it, version the index.

---

## 4. What we embed

| Doc type | Fields embedded | Index |
|---|---|---|
| **Series** | title + alt/romaji titles + description + tag names + credited entity names + relation context | pgvector |
| **Entity** (person/circle/publisher) | name + aliases + bio | pgvector |
| **Wiki page** | prose sections | LEANN |
| **Chapter (novel)** | canonical text, chunked | LEANN |
| Comments | *not embedded* (noise > value; lexical only) | — |

Embedding a series *with its entity and relation context* (from
`specs/entities-and-relations.md`) means "works by the same artist" and
"same-universe" cluster naturally even before explicit relations are drawn.

---

## 5. Query pipeline

```
1. parse query → split TAG PREDICATES (namespace:name, +include / -exclude,
   resolved through aliases) from free text. E-Hentai-style syntax:
   `artist:foo -warning:gore language:thai   cute cooking rivals`
2. TAG FRAME (first citizen): apply include/exclude over the confirmed-tag
   GIN index → the candidate set every later step is constrained to.
   A pure-tag query needs no embedding at all — just the index.
3. if free text remains: embed it (Redis-cached) ; run lexical (FTS) in parallel
4. ANN: pgvector (series/entity) [+ LEANN for deep-text scope], within the frame
5. FUSE via RRF with priority weight  tags ≫ name(lexical) ≫ semantic
6. HARD FILTERS (not re-ranking): drop unpublished/dmca/deleted state,
   blocked tags, and age-gated warning:* above the viewer's setting
   (specs/roles-and-content-model.md, history-follows-recommendations.md)
7. re-rank: popularity/recency signal (rankings) as a light boost
8. return with highlights + "why" (matched tags / title / semantic / by author X)
```

- **Only `confirmed` tags** (`specs/tag-voting.md`) participate in the tag frame
  and counts — voted-down/proposed tags never constrain or leak into results.
- **Tag predicates are hard constraints**; so are state/age filters — consistent
  with recs (age-gate never leaks through search).
- **Autocomplete**: lexical prefix on titles/entities/tags for instant
  suggestions; semantic kicks in on full submit.
- **"More like this"**: symmetric series→series ANN, blended with explicit
  `series_relations` (sequels/spin-offs rank first — known beats guessed).

---

## 6. Indexing lifecycle

- On publish/edit of a series/chapter/entity/wiki page → enqueue re-embed;
  update pgvector row transactionally.
- **LEANN is build-offline oriented** (graph pruned at build, embeddings
  discarded). Handle freshness with a **two-tier** pattern: periodic full LEANN
  rebuild (nightly/weekly) **+ a small fresh pgvector/HNSW delta index** for
  items added since the last build, queried alongside and merged. Log the
  rebuild cadence so "why isn't my new chapter searchable semantically yet" has
  an answer.
- Reindex is a first-class ops job (model bump, schema change) — versioned, with
  a shadow index and atomic swap.

---

## 7. Cost & ops (nonprofit lens)

- Storage: pgvector series index is small; LEANN keeps the big text corpus < 5%
  of raw — the whole point.
- Compute: the embedding service is the new moving part. Cache query embeddings,
  batch recompute, and keep the model small enough for CPU if no GPU budget.
- Metrics: search **latency p95**, **zero-result rate**, click-through, and
  **recompute load** (LEANN) — watch the storage↔compute trade in production.

---

## 8. Roadmap fit

- **Phase 2** (catalog): lexical FTS + **pgvector series search** + "more like
  this" + cross-lingual multilingual model. This alone beats Nekopost's
  category-only browse.
- **Phase 3+**: LEANN deep-text tier (search inside novels/wiki) once that corpus
  is large enough to justify it.

**Test checklist**: Thai query finds EN-titled series (cross-lingual); typo
tolerance; "more like this" returns relatable series; explicit `author:` filter
works; RRF beats either retriever alone on a sample set; age-gated result never
appears past the viewer's setting; new series searchable via delta index before
the LEANN rebuild; model-version bump triggers a clean reindex + atomic swap.

Sources: [LEANN (arXiv 2506.08276)](https://arxiv.org/abs/2506.08276).
