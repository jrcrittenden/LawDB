# LawDB: First Amendment, Free Speech & Free Press Drift Study
## Architecture & Implementation Plan (v2)

Supersedes v1 (general-purpose IRAC search platform). Scope is now one research question.

**Purpose:** Establish a record of semantic drift in legal language, using the First Amendment as the testbed. It is chosen because it is among the most argued and commented-on parts of the Constitution, which gives a long, continuous, high-volume record in which drift can be detected. (That it is *the most* discussed is an assumption, not something we have measured, and the design does not depend on it; it only needs enough volume over a long enough span.) A successful method here could later be applied to other provisions.

**Question:** Has the way the Supreme Court discusses the First Amendment and free speech drifted semantically over time, and if so, along what dimensions (doctrine, rationale, vocabulary)?

## 1. Decisions

| Area | Decision |
|---|---|
| Corpus | ~1,000 SCOTUS opinions on the First Amendment, free speech, and free press, **full history (no start-year cutoff)**, from CourtListener |
| Extraction | Claude via the Claude Agent SDK (Python), structured output |
| Interface | CLI only |
| Budget | Configurable hard cap (USD) per run; run stops cleanly when reached |
| Embeddings | Over full opinion text and over each extracted component |
| Other models | Deferred until run 1 is analyzed and human-rated; extractor is model-agnostic from day one |

Dropped from v1 (not needed at 1k docs): PostgreSQL, Qdrant, Celery, Airflow, FastAPI, Docker. Use SQLite (or DuckDB) + Parquet + a flat numpy/FAISS index. Revisit if scale grows. *This is my judgement call, push back if you want the heavier stack.*

## 2. Verified assumptions

- CourtListener reachable from the dev environment. **Anonymous access is throttled to 125 requests/day**, so an API token is required (free account). The throttle is likely keyed on the shared cloud egress IP, not on this project; a token should move the limit to a per-account basis (untested until a token is provided).
- A search for `"freedom of speech" OR "First Amendment"`, court=scotus, precedential only, returns **1,761** hits. So 1k is available, but "mentions the First Amendment" is not "is a free speech case"; selection needs a relevance step (below).

- **Early history is thin.** Precedential SCOTUS hits for `"freedom of speech" OR "liberty of speech" OR "freedom of the press" OR "liberty of the press" OR "First Amendment"` before 1919: **47** (34 with only `"freedom of speech" OR "First Amendment"`). Before 1850: **2**. Cutting off at 1919 would hide any 1789-1919 drift, but that era will have few documents, so early bins carry wide uncertainty. Counts are search hits, not confirmed relevant cases.
- Terminology itself drifts ("liberty of the press", "freedom of the press", pre-incorporation cases decided as federal-only), so keyword queries must be broad and recall in the early era should be spot-checked by reading, not trusted.

Not yet verified: opinion text quality for older cases (OCR / HTML vs plain text), per-case token counts, actual cost per opinion.

## 3. Pipeline

```
fetch -> select -> extract -> embed -> analyze -> rate
```

1. **Fetch** (`lawdb fetch`): CourtListener v4 API, cluster + opinion text + metadata. Cache raw JSON on disk, idempotent and resumable.
   Fetch the **entire** keyword-matched candidate pool first (broad terms: speech, press, First Amendment, plus historical variants like "liberty of the press"). No selection or reranking happens until the whole pool is on disk.
2. **Select / rerank** (`lawdb select`): run once over the complete pool. Score relevance with a cheap Claude classification (is free speech / free press / First Amendment expression the central issue?), optionally combined with embedding similarity to the retrieved set. Erring toward accuracy: **take every pre-1919 candidate that classifies as relevant**, then fill to the ~1,000 target from later eras by rank, rather than sampling proportionally. Record scores, inclusion decisions and reasons. Borderline candidates are kept and tagged, not dropped. Since the pool is only ~1.8k, the cutoff can be revisited after seeing the score distribution.
3. **Extract** (`lawdb extract`): Agent SDK run per opinion, schema-validated output (section 4). Every extracted span must be a verbatim quote that is verified against the source text by offset; failures are retried, then flagged. Tracks tokens and cost per call in a ledger.
4. **Embed** (`lawdb embed`): see section 5.
5. **Analyze** (`lawdb analyze`): drift metrics and plots (section 6).
6. **Rate** (`lawdb rate`): human rating UI in the terminal (section 7).

Each stage reads and writes the store, keyed by `(opinion_id, stage, model, prompt_version)` so reruns with other models do not overwrite run 1.

## 4. Extraction schema

Per opinion, separately for majority / plurality, **each concurrence**, and each dissent (voice matters for drift; do not collapse them; concurrences are in scope for run 1, dissents are included on the same basis unless you say otherwise). Each separate opinion is its own record with author, joiners, and type:

- **IRAC core:** issue, rule (legal test or standard stated), application, holding/conclusion
- **Drift-oriented codings** (categorical, so drift can be measured without embeddings):
  - speech category (political, commercial, symbolic, obscenity, incitement, campaign finance, public employee, student, etc.)
  - standard of review (strict / intermediate / rational basis / categorical rule / balancing / none)
  - rationale for protecting speech (marketplace of ideas, self-governance, autonomy, tolerance, other, with a quote)
  - precedents relied on (cited cases, resolved to CourtListener IDs where possible)
  - outcome (speech protected / not protected / remanded)
- Every field carries `quote` + char offsets, a self-reported confidence, and `extraction_model` / `prompt_version`.

## 5. Embeddings

Anthropic has no embedding endpoint, so embeddings come from elsewhere. Default: a local sentence-transformers model (free, reproducible, frozen). Opinions are long, so chunk, embed chunks, and pool (mean) per opinion; also embed each extracted component text.

Because drift conclusions depend on the embedding model, run at least two (one general, one larger or legal-tuned) and report only drift that appears in both. Embedding model and version are stored with every vector.

## 6. Drift analysis

Embedding drift alone is easy to misread: legal prose style, citation formats, and topic mix all change over time. Controls:

- **Baseline (optional, deferred):** the confound is that SCOTUS prose changes over time for reasons unrelated to speech (style, citation habits, topic mix), so "speech opinions drifted" does not by itself mean "speech *discourse* drifted". A baseline is a random sample of non-speech SCOTUS opinions run through the same embedding step (no LLM extraction, so near-zero cost) to show how much drift to expect from general language change. Not required for the first analysis; add it if the speech drift looks interesting and needs that claim defended.
- **Component-level, not whole-document:** compare rules to rules, rationales to rationales. Strip citations and boilerplate before embedding.
- **Uneven eras:** bin by historical period (e.g. founding/Sedition Act, 19th century, 1919-1960s, Warren, Burger/Rehnquist, Roberts) rather than equal-width decades, report n per bin, and use bootstrap intervals so thin early bins are not over-read. Pre-incorporation (before 1925) cases are tagged since the doctrinal context differs.
- **Metrics:** period-binned centroid distance, nearest-neighbor era mixing (do neighbors come from the same era?), clustering stability, change-point detection.
- **Categorical cross-check:** the codings in section 4 give interpretable time series (e.g. share of strict scrutiny, shifts in rationale). Embedding drift that has no categorical or human-readable counterpart gets treated as suspect.
- **Qualitative:** pull the exemplar passages at each detected shift for manual reading.

## 7. Human rating (gate before other models)

- Stratified random sample by era (about 100 opinions to start), with each component shown beside its source quote.
- Rate per field: correct / partially / wrong, plus free-text notes. Stored with rater ID so inter-rater agreement is computable if there is more than one rater.
- Output: per-field accuracy by era. Run 2 (other models) starts only after this shows extraction quality is acceptable, and uses the same sample for head-to-head comparison.

## 8. Budget control

- `--max-usd` on every LLM-calling command, plus a default in config.
- Pre-run estimate from token counts (counted, not guessed) with confirmation if estimate exceeds the cap.
- Ledger table of every call (tokens in/out, cost, model); run halts when cumulative spend reaches the cap and resumes from where it stopped.
- Pilot first: 10 opinions end to end to measure real per-opinion cost before committing the full run.

## 9. Project structure

```
LawDB/
├── src/lawdb/
│   ├── cli.py            # entrypoint: fetch/select/extract/embed/analyze/rate
│   ├── config.py         # budget, models, paths
│   ├── courtlistener.py
│   ├── select.py
│   ├── extract/          # schema.py, prompts.py, runner.py (Agent SDK), verify.py
│   ├── embed.py
│   ├── analyze/          # drift.py, controls.py, plots.py
│   ├── rate.py
│   └── store.py          # SQLite/Parquet access, cost ledger
├── data/                 # gitignored: raw cache, db, vectors
├── tests/
├── pyproject.toml
└── ARCHITECTURE_PLAN.md
```

## 10. Phases

0. **Setup:** package skeleton, config, store, CourtListener token.
1. **Pilot:** fetch + extract 10 opinions; measure cost, text quality, quote-verification pass rate.
2. **Fetch the whole candidate pool,** then select/rerank over it; review selection decisions.
3. **Full extraction run** under budget cap.
4. **Embed + analyze** (with control corpus).
5. **Human rating,** accuracy report.
6. **Decision point:** add other models (extraction and embeddings) and compare.
7. **Candidate, not committed: external discourse corpus.** Law review articles (and possibly press for the early era) that comment on SCOTUS speech/press decisions, to test whether SCOTUS drift tracks, leads, or lags outside discourse, and whether drift on the same dimensions (tests, rationales) appears in a stream with different stylistic drivers. Open issues: bulk full-text availability and licensing (most law review archives are paywalled or ToS-restricted), uneven coverage (law reviews start in the 1880s; early eras would need press), and relevance filtering. Availability must be probed before this is planned further.

## 11. Open questions

1. ~~Date range~~ **Resolved:** full history; scope is First Amendment, free speech, and free press.
2. **Budget number:** default cap for the pilot and for the full run?
3. ~~Control topic~~ **Deferred:** optional non-speech baseline, embeddings only (see section 6).
4. **Raters:** unknown for now; rater ID is stored with every rating so this can be decided later.
5. ~~Concurrences~~ **Resolved:** included. Dissents: confirm they should be too.

---

**Status:** Draft v2, pending answers to section 11.
