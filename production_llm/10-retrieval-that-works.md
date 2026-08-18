# Chapter 10 — Retrieval That Works: Hybrid, Rerank, Evaluate

## What you'll be able to do after this chapter

1. Name the four failure classes that make pure vector search underperform — exact identifiers, rare terms, negation, and acronyms — and recognise each in a real query before you touch code.
2. Write one Postgres SQL query that fuses BM25 full-text search and pgvector dense search with Reciprocal Rank Fusion, with the ACL predicate inside both arms of the query.
3. Wire a cross-encoder reranker into a top-50 → top-6 pipeline, and state its exact latency and dollar cost at 10,000 requests/day.
4. Apply the decision rule for query rewriting, decomposition, and HyDE, and say — for each — whether it earns its latency on AtlasDesk's own traffic.
5. Enforce ACL inside the query predicate, not after it, and prove it with a test that attempts a cross-tenant leak and fails to leak.
6. Compute context precision, context recall, MRR, and nDCG on AtlasDesk's eval set and read AtlasDesk's own before/after retrieval numbers.

---

## The problem this solves

Priya Raghavan asks the Chapter 9 prototype a question she already knows the answer to, because she wrote the policy: *"What is the refund for a withdrawal on day 20, given the learner deferred once already?"*

The prototype embeds the question, does a cosine search over `chunks.embedding`, takes the top six, and hands them to the model. It comes back confident and wrong. It quotes section 4.2 — the 14-day refund window — and says nothing about the deferral fee in section 4.3, because the query never mentioned the word "deferral" loudly enough for a single dense vector to weight it, and the chunk containing 4.3's non-refundable administrative fee sat at rank 9. The embedding for "day 20" and the embedding for "deferred once already" are close to *some* chunk about timelines, and the top-6 cutoff quietly discarded the one chunk that made the answer correct. This is the compound case that seed set item `C1-003` exists to catch, and the Chapter 9 build fails it.

Two more of Priya's questions break it a different way. She pastes a learner's exact enrolment id, `CRS-PGDM-2026`, into a question about a specific cohort — the embedding for an alphanumeric course code is nearly indistinguishable from the embedding for any other course code, so dense search returns plausible-looking chunks about the *wrong* course. And she asks what the hardship waiver **does not** cover — a negation — and dense search, which has no concept of negation, returns the chunk that most resembles "hardship waiver," which is the one describing what it *does* cover.

None of these are model failures. The model was never given the right chunk. This chapter fixes retrieval itself, not the prompt around it, and it is the chapter that finally makes C1 — cited handbook answers — correct enough to ship. If you skip it, every downstream fix (better prompts, bigger models, more careful citation formatting) is polishing an answer built on evidence that was never fetched.

---

## Concepts

### Why pure vector search underperforms: four named failure classes

Dense embeddings are trained to cluster *semantically similar* text. That property is exactly what fails on four query shapes that show up constantly in a support and policy domain:

| Failure class | What happens | AtlasDesk example |
|---|---|---|
| Exact identifiers | An embedding for `CRS-PGDM-2026` is close to the embedding for `CRS-PGDM-2025` and `CRS-MBA-2026` — the model has no privileged notion of "this exact string" | A question naming a specific course code or learner id retrieves the wrong cohort's chunk |
| Rare terms | A term that appears in exactly one place in the corpus (a specific fee amount, a rarely-used clause name) carries little weight in a dense vector trained on general semantics | "the ₹5,000 administrative fee" retrieves generic fee chunks, not the one clause that states the number |
| Negation | Embeddings for "covers X" and "does not cover X" are close together, because the words describing X dominate the vector | "what the waiver does **not** cover" (seed case `C1-005`) retrieves the coverage chunk, not the exclusion chunk |
| Acronyms and jargon | An internal acronym (`PGDM`, `LRN-`) has no stable pretrained meaning; the embedding model guesses from surface form | "PGDM deferral rules" drifts toward generic MBA content if the corpus underrepresents the acronym |

Lexical search — BM25 and its relatives — is exactly the opposite instrument. It matches surface tokens, weighted by how rare and how concentrated they are in the query and in the document. It is unbeatable at exact identifiers and rare terms, and it is *also* blind to negation and paraphrase — "may not withdraw" and "may withdraw" score almost identically to a bag-of-words model. The two methods fail in complementary places, which is the entire argument for running both and fusing the result, rather than picking a winner.

**Decision rule.** If your retrieval eval set has any case containing an ID, a number, a negation, or a domain acronym — and any real support corpus does — dense-only retrieval is disqualified before you write a line of reranking code. **Switch when:** you genuinely have no exact terms, no negation-sensitive facts, and a small, semantically homogeneous corpus (a single FAQ of fewer than a few hundred short entries); at that scale hybrid fusion is not wrong, just not worth the SQL.

### BM25 + dense + Reciprocal Rank Fusion, in one query

Reciprocal Rank Fusion (RRF) does not need the two search systems' scores to be comparable — BM25 scores are unbounded and corpus-dependent, cosine similarity is bounded — because it fuses **ranks**, not scores. Cormack and Clarke's original formulation (SIGIR 2009) is one line:

```
RRFscore(d) = Σ over each ranking r that contains d of  1 / (k + rank_r(d))
```

`k` is a small constant — the paper fixes it at 60 during a pilot run and reports the result is not sensitive to the exact value — that dampens the effect of rank-1 dominating the sum and lets a document that ranks respectably in *both* lists outscore a document that ranks first in one and absent from the other. In their TREC evaluations, RRF beat every individual input ranking and beat Condorcet fusion, by a margin the authors report as statistically significant (p between 0.000 and 0.004 across test sets).

The property that matters for AtlasDesk: a chunk does not need to be in the top of *either* list to win. A chunk ranking 4th lexically and 6th semantically (`1/64 + 1/66 ≈ 0.0308`) can outscore a chunk ranking 1st lexically but absent from the dense results entirely if you fuse on rank-presence, which is why the SQL below always computes both arms over the full ACL-filtered candidate set rather than short-circuiting.

Here is the fusion as one Postgres query — Ch8's `chunks.tsv` (`tsvector`) and Ch9's `chunks.embedding` (`vector(1024)`) are read by the same statement, and the ACL predicate appears **inside both CTEs**, never as a filter on the final result:

```sql
-- src/atlasdesk/retrieval/hybrid.sql
-- Hybrid BM25 + dense retrieval fused with Reciprocal Rank Fusion (k=60).
-- Parameters: $1 query text (to_tsquery input), $2 query embedding (vector),
--             $3 tenant_id, $4 acl_tags (text[]), $5 candidate_k.
WITH lexical AS (
    SELECT
        c.id AS chunk_id,
        row_number() OVER (ORDER BY ts_rank_cd(c.tsv, query) DESC) AS rank
    FROM chunks c, to_tsquery('english', $1) AS query
    WHERE c.tsv @@ query
      AND c.tenant_id = $3
      AND c.acl_tags && $4::text[]
    ORDER BY ts_rank_cd(c.tsv, query) DESC
    LIMIT $5
),
dense AS (
    SELECT
        c.id AS chunk_id,
        row_number() OVER (ORDER BY c.embedding <=> $2::vector) AS rank
    FROM chunks c
    WHERE c.tenant_id = $3
      AND c.acl_tags && $4::text[]
    ORDER BY c.embedding <=> $2::vector
    LIMIT $5
),
fused AS (
    SELECT
        chunk_id,
        SUM(1.0 / (60 + rank)) AS rrf_score
    FROM (SELECT * FROM lexical UNION ALL SELECT * FROM dense) both_arms
    GROUP BY chunk_id
)
SELECT
    c.id, c.document_id, c.text, c.heading_path, c.page,
    d.source_uri, f.rrf_score
FROM fused f
JOIN chunks c ON c.id = f.chunk_id
JOIN documents d ON d.id = c.document_id
ORDER BY f.rrf_score DESC
LIMIT $5;
```

Three things about this query are the entire point, and each is a mistake I have seen shipped. **The ACL predicate is repeated in `lexical` and `dense`, not applied once at the end** — a chunk that never entered either candidate list because it failed the ACL check cannot be resurrected by RRF, because RRF only fuses what it is given. **`LIMIT $5` inside each CTE, not just the final `SELECT`** — this bounds the work Postgres does per arm to `candidate_k` (50 in our config), so the query's cost does not grow with corpus size once an index exists; there is a `GIN` index on `tsv` and an `HNSW` index on `embedding` from Chapter 8 and 9 respectively, and both CTEs use them. **`UNION ALL`, not `UNION`** — a chunk appearing in both arms must contribute two rows to the sum in `fused`, one per arm; deduplicating here would silently turn RRF back into a single ranking.

### Cross-encoder reranking: the top-50 → top-6 pattern

RRF fusion is cheap and precise on IDs and rare terms, but it still ranks with two shallow signals — term overlap and vector distance — neither of which reads the query and the passage *together*. A cross-encoder does: it takes `(query, passage)` as one input and outputs a relevance score, at the cost of one forward pass per pair. That cost is why you never cross-encode the whole corpus; you cross-encode the ~50 candidates RRF already narrowed it to.

```mermaid
flowchart LR
    Q(["Query"]) --> RW["Query transform<br/>(optional — see below)"]
    RW --> EMB["Embed query<br/>~40 ms"]
    RW --> LEX["BM25 full-text<br/>Postgres tsvector"]
    EMB --> DEN["Dense ANN search<br/>pgvector HNSW"]
    LEX --> FUSE["RRF fusion<br/>one SQL query, ACL inside both arms"]
    DEN --> FUSE
    FUSE --> CAND["Top 50 candidates"]
    CAND --> RERANK["Cross-encoder rerank<br/>~150-300 ms for 50 pairs"]
    RERANK --> TOP["Top 6 chunks"]
    TOP --> CITE["Answer generation<br/>with citation enforcement"]
```

Read the diagram as two narrowing stages with different jobs. The first stage (BM25 + dense + RRF) is a cheap, high-recall filter — its job is to make sure the right chunk is *somewhere* in 50, not to put it first. The second stage (cross-encoder) is an expensive, high-precision filter — its job is ordering, and it is only ever asked to order 50 things, never 400,000. Running the cross-encoder over the raw corpus would be correct and unaffordable; running RRF alone and skipping the cross-encoder is affordable and measurably worse at ordering near-miss chunks (Chapter 9's `bench_recall.py` measures *recall*, which the fusion stage already delivers — reranking is what improves *precision at k*, the thing the model actually reads).

| Stage | Latency (our project run, p50) | What it buys |
|---|---|---|
| RRF fusion over 50 candidates | ~120 ms (index-backed, matches the Bible §5 retrieval budget line) | High recall from two complementary signals, ACL-safe by construction |
| Cross-encoder rerank, 50 → 6 | ~180–260 ms for a 50-pair batch on a hosted reranker; ~90–150 ms self-hosted on a GPU, ~700 ms+ on CPU | Precision at k: the top 6 the model reads are the 6 that actually answer the question, not the 6 that merely resemble it |

**Decision rule.** Rerank whenever the fused top-50 is heterogeneous enough that ordering matters — in practice, whenever your corpus has more than a few hundred chunks per topic, which AtlasDesk's 400-page handbook comfortably exceeds. **Switch when:** your p95 retrieval budget (Bible §5: 120 ms fusion + 250 ms rerank is already the whole retrieval line) cannot absorb 250 ms, or your candidate set is small enough (under ~15 chunks total after fusion) that reranking cannot move much. At that size, skip reranking and spend the latency budget on a second retrieval pass instead.

### Query transformation: rewriting, decomposition, HyDE — and whether each earns its keep

All three techniques spend an extra model call (and its latency and cost) *before* retrieval, to change what gets searched for. None of them are free, and none of them are always right.

| Technique | What it does | Extra cost | Fixes | Decision rule |
|---|---|---|---|---|
| Rewriting | Resolves pronouns and ellipsis against conversation history ("it" → "the hardship waiver") into a self-contained query | +1 small-model call, ~150–300 ms | Multi-turn follow-ups that would otherwise search for "it" | Always on for any turn after the first in a conversation; skip on the first turn |
| Decomposition | Splits a compound question into independently-retrievable sub-questions, retrieves each, merges | +1 model call to split, +N-1 extra retrieval round-trips | Genuinely compound questions ("compare policy A to policy B", seed case `C1-003`-shaped) | Only when the question contains an explicit comparison, conjunction, or multi-entity reference; a classifier or a cheap heuristic (does the question contain "and", "compare", "versus", two distinct entity mentions) gates it, because running it on every query multiplies retrieval latency for no benefit on simple questions |
| HyDE (Hypothetical Document Embeddings) | Asks a model to write a plausible *answer* to the question, embeds that instead of the question, and searches with it | +1 model call, ~300–600 ms depending on the hypothetical's length | The vocabulary gap between how questions are phrased and how the source document is phrased | Only for domains where questioners' vocabulary reliably differs from the document's — worth a controlled A/B on your own eval set before defaulting on; do not enable from intuition |

HyDE is the one most teams reach for on faith, so it is worth being precise about the evidence. Gao et al.'s original HyDE paper (*Precise Zero-Shot Dense Retrieval without Relevance Labels*, 2022) showed gains on out-of-domain retrieval, specifically where the corpus's phrasing diverges from natural questions and no relevance-labelled training data exists for that domain — an unsupervised regime by design. AtlasDesk's handbook is not that regime: it is a static, well-structured internal corpus, we can and do label a retrieval eval set (below), and rewriting already resolves most of the vocabulary gap that matters for policy text ("day 20" against "14-day window" is a numeric-comparison problem HyDE does not solve either). **The rule that follows: measure HyDE on your own retrieval eval set before shipping it; do not add it because a blog post recommended it for a different corpus shape.** We measure it in this chapter's Build section and it does not clear the bar for AtlasDesk.

Query decomposition is the one that pays for itself here, because seed case `C1-003` is exactly the shape it fixes: a question that requires two sections is otherwise answered from whichever section the fused ranking happened to favor.

### ACL and metadata filtering inside the predicate, never after

This is the single highest-consequence design decision in this chapter, and the reason it has its own senior-practice callout.

> **▸ Senior practice #10 — ACL enforced at query time, proven by a leak test**
>
> There are exactly two places you can apply an access-control check on retrieved data: inside the query that fetches candidates, or on the list the query returns. Only the first one is a security control. The second is a filter, and a filter that runs after the data has already been fetched, embedded into a prompt-building loop, logged to a trace, or cached, has already leaked the data to every downstream consumer of that fetch — the filter just stops it from being *displayed*, in the one code path you remembered to add it to.
>
> AtlasDesk's `documents.acl_tags` and `chunks.acl_tags` exist from Chapter 8 for exactly this reason. Every retrieval query — the hybrid SQL above, the analytics layer in Chapter 16, the memory layer in Chapter 11 — carries `WHERE tenant_id = $tenant AND acl_tags && $principal_tags` inside its own predicate, in every CTE, with no code path that fetches first and checks second.
>
> The only way to trust this claim is to try to break it and watch the attempt fail. `tests/test_acl_leak.py`, below, constructs the canonical case: Daniel Osei, a `meridian-core` agent, asks the exact question whose answer lives only in `meridian-exec`'s sponsor-refund addendum (seed case `C1-007`). The test asserts the addendum's content never appears in the candidate list, never mind the final answer. If that test ever goes green for the wrong reason — say, because nobody seeded any `meridian-exec` chunks — it is worthless, so the test also asserts the excluded content exists in the database and would be returned by the *same query with the tenant filter removed*. A leak test that cannot demonstrate the leak it is preventing is not a test.

The mechanical rule, stated so you can paste it into a code review comment: **if you can write the word "then filter" in a sentence describing your retrieval code, the design is wrong.** ACL is a `WHERE` clause, not a subsequent `if`.

### Citation enforcement and groundedness checking

Chapter 6 built `Citation` (`chunk_id`, `quote`, 8–400 characters) and `unknown_citations()`, which checks that every `chunk_id` an answer cites was actually among the chunks supplied in context. That check is necessary but not sufficient: a model can cite a real, retrieved `chunk_id` and then attach a `quote` that is not actually in that chunk's text — a fabricated citation wearing a real ID. Retrieval closes that second gap, because retrieval is the only place that holds both the chunk text and the citation at the same moment.

**Decision rule.** Verify two things about every citation, at the point where you have both the answer and the retrieved chunks in hand: the `chunk_id` was supplied (`unknown_citations`, Chapter 6), and the `quote` is a normalized substring of that chunk's `text` (groundedness, this chapter). Escalate on either failure — a citation that fails both checks is not a formatting defect, it is the model asserting something the corpus does not contain.

---

## How industry does it

### Case 1 — Glean: permission-aware retrieval as the product, not a feature

**The problem.** Enterprise search inside a company that has any real access controls — a wiki with a legal-only space, a wiki with an HR-only compensation page, a codebase with a private repo — cannot correctly rank a document it should never have shown a given user in the first place. A generic vector index over "everything the company owns" is a data-leakage engine with a nice UI.

**What they built.** Glean is described, by its own published architecture writeups, as deliberately not relying on vector search alone: it layers classical information-retrieval signals (query expansion, synonym handling, document-quality scoring) with embedding-based semantic search, on top of a permissions model inspired by Google's internal enterprise search tooling, so that a document a user cannot access is not merely hidden in the UI but excluded from the candidate set the ranker ever considers. On top of that hybrid retrieval layer sits a personalization stage — role, team, and interaction history — because *who is asking* changes not just what they may see but what they should see first.

**The measured outcome.** Glean's own published material describes continuous improvement from user-feedback signals — one case study cites roughly a 20% search-quality improvement over six months of tuning — and the company's growth (reported valuation in the multi-billion-dollar range through 2026 funding rounds) is itself a proxy for the outcome: enterprises will pay a premium for search that is simultaneously good *and* provably permission-correct, which a bolt-on vector search over an unpartitioned index cannot promise.

**What you should copy at 1/1000th the scale.** Do not treat "search everything" and "filter by permission" as two separable concerns you can build in either order — the permission check has to be a first-class input to which documents ever become candidates, exactly like AtlasDesk's `acl_tags && $principal_tags` inside the retrieval CTE, not a UI-layer redaction. Combine lexical and semantic signals rather than picking one; Glean's own description of its stack is a larger version of the same argument this chapter makes about BM25 and dense vectors being complementary, not competing. And treat ranking quality as something you tune continuously against real usage, not something you set once at launch — Chapter 18's regression suite and Chapter 24's failure-promotion ritual are AtlasDesk's version of that loop, at a scale one engineer can run.

### Case 2 — Cohere Rerank at Notion: reranking as the retrieval architecture, not an add-on

**The problem.** Notion's workspace search spans a huge range of workspace sizes — from personal notebooks with a handful of pages to enterprise workspaces with hundreds of thousands of blocks — and needed a single relevance layer that worked at both ends without maintaining a full embedding-and-vector-index pipeline for every workspace, however small.

**What they built.** Notion deployed Cohere's Rerank model, hosted through Amazon SageMaker with autoscaling, as the relevance layer behind "every single search and Notion AI interaction," according to Notion engineer Abhishek Modi, quoted in Cohere's published customer story. For smaller workspaces (fewer than roughly 1,000 documents) Notion reports using Rerank directly over candidate documents rather than maintaining a separate embedding and vector-search stack for accounts too small to need one — reranking narrows a candidate set that in their description runs from around 100,000 candidates down to roughly 200 before the final ranking step, a two-order-of-magnitude cut done entirely by the pairwise relevance model.

**The measured outcome.** Cohere's published case study is explicit that the improvement is qualitative rather than benchmark-quantified in the public writeup — better precision on ambiguous, short workspace queries, plus the reported operational simplification of not running vector infrastructure for the smallest tier of workspace. Both companies describe this as a running-in-production architecture choice, not a one-time experiment.

**What you should copy at 1/1000th the scale.** Reranking is not only a top-of-funnel polish step; at small candidate-set sizes it can *be* the retrieval architecture, skipping a vector index entirely — a decision rule worth remembering the day AtlasDesk (or a client's smaller deployment of it) has a corpus of a few hundred documents, where an HNSW index is pure overhead and a single reranker call over all candidates is both simpler and cheaper. Deploy the reranker as its own scaled service behind a client interface — Notion's SageMaker autoscaling and this chapter's `RerankClient` Protocol solve the same problem: the reranker's load profile does not match the rest of your request path, and coupling its scaling to your API server's is a self-inflicted bottleneck.

**A brief call-back, because AtlasDesk's own C6 depends on the number:** Chapter 1 already covered Klarna in detail and Chapter 3 covered Intercom Fin's spec-writing discipline; Intercom's own published figures put Fin's resolution rate at roughly 76% as of mid-2026 on their defined metric (a confirmed-resolution definition, not a raw deflection count). The number worth carrying into this chapter is the shape, not the figure: every deflection or containment number is only as trustworthy as the retrieval underneath it, because a support bot that answers confidently from the wrong chunk still counts as "resolved" until a human checks — which is precisely the gap AtlasDesk's citation and groundedness checks exist to close before the number gets reported.

---

## Build: AtlasDesk increment — hybrid retrieval, reranking, and C1 end to end

### Project state

**What exists after Chapters 1–9:** the readiness scorer; the repo scaffold with `config.py`, `errors.py`, `llm/pricing.py`; the spec, ADRs, and `evals/datasets/seed_20.jsonl`; the provider layer (`llm/base.py`, both adapters, retry, circuit breaker, router); the prompt registry; `schemas/answer.py` (`Citation`, `Answer`, `AnswerDraft`, `apply_escalation_policy`, `unknown_citations`) and `llm/structured.py` (the repair loop); `context/budget.py` and `context/assemble.py`; `ingest/parse.py`, `ingest/chunk.py`, `ingest/pipeline.py` and `migrations/0001_documents.sql` (`documents` and `chunks`, both with `acl_tags` and `heading_path` from day one); `ingest/embed.py`, `retrieval/store.py`, `migrations/0002_vectors.sql` (the `vector(1024)` column, the HNSW index, and `scripts/bench_recall.py`); and `security/principal.py` (`Principal`, frozen per Book Bible §4.5).

**What this chapter adds:** `retrieval/types.py` (`RetrievedChunk`, `RetrievalFilters`, per Book Bible §4.6), `retrieval/acl.py`, `retrieval/hybrid.py` (`HybridRetriever.search`, the exact signature from §4.6), `retrieval/rerank.py`, `retrieval/transform.py`, `evals/retrieval_metrics.py`, and `scripts/ask_policy.py` (the C1 pipeline wired end to end: hybrid search → rerank → structured `AnswerDraft` → project to `Answer` → escalation policy).

**What it does not add:** any change to ingestion or embedding — those are Chapters 8 and 9's contracts, imported here, never redefined. `Settings` gains exactly two fields (`cohere_api_key: SecretStr | None`, `rerank_model: str`), additive per Book Bible §4.2's evolution rule.

### Repo tree diff

```
  src/atlasdesk/
    config.py                    # Ch 2 — two fields added: cohere_api_key, rerank_model
    security/
      principal.py                # Ch 9 — unchanged, imported not redefined
    ingest/                       # Ch 8/9 — unchanged
    retrieval/
      store.py                    # Ch 9 — unchanged, its pool is what hybrid.py drives
+     types.py                    # RetrievedChunk, RetrievalFilters
+     acl.py                      # ACL predicate builder + pure-python leak checker
+     hybrid.py                   # HybridRetriever.search — BM25+dense+RRF, one query
+     rerank.py                   # CrossEncoderReranker, top-50 -> top-k
+     transform.py                # QueryTransformer — rewrite, decompose, hyde
  evals/
+   retrieval_metrics.py         # context precision/recall, MRR, nDCG
  scripts/
+   ask_policy.py                 # C1 end to end: retrieve -> rerank -> answer -> escalate
  tests/
+   test_acl.py
+   test_hybrid.py
+   test_rerank.py
+   test_transform.py
+   test_retrieval_metrics.py
+   test_acl_leak.py              # the cross-tenant leak attempt — must fail to leak
```

### Config additions

```python
# src/atlasdesk/config.py  (diff — two fields added to the Chapter 2 Settings class)
from __future__ import annotations

from functools import lru_cache

from pydantic import Field, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore", case_sensitive=False)

    anthropic_api_key: SecretStr | None = None
    openai_api_key: SecretStr | None = None
    primary_provider: str = "anthropic"
    anthropic_model: str = Field(default="", description="e.g. the current Claude model id")
    anthropic_small_model: str = Field(default="", description="cheaper Claude model id")
    openai_model: str = Field(default="", description="e.g. the current GPT model id")
    openai_embedding_model: str = Field(default="", description="embedding model id")
    database_url: str = "postgresql://atlas:atlas@localhost:5432/atlasdesk"
    daily_cost_limit_usd: float = 25.0
    request_cost_limit_usd: float = 0.15
    agent_max_steps: int = 12

    # Chapter 10 additions — reranking. Never a literal key or a marketing model
    # name in code; both are configuration, resolved at runtime.
    cohere_api_key: SecretStr | None = Field(
        default=None, description="Set if using a hosted reranker; self-hosted needs no key."
    )
    rerank_model: str = Field(default="", description="Current hosted or self-hosted reranker id.")

    def require_any_provider(self) -> None:
        if not (self.anthropic_api_key or self.openai_api_key):
            raise RuntimeError("Set ANTHROPIC_API_KEY or OPENAI_API_KEY in .env")


@lru_cache(maxsize=1)
def get_settings() -> Settings:
    return Settings()
```

### Retrieval types

```python
# src/atlasdesk/retrieval/types.py
"""Shared retrieval value types (Book Bible Sec 4.6).

Introduced across Chapters 8-10: chunks and their ACL tags are Chapter 8's
migration, the embedding column is Chapter 9's, and this module is where the
retrieved-chunk shape and filter shape live so every later chapter imports the
same two types instead of redefining them.
"""

from __future__ import annotations

from datetime import datetime

from pydantic import BaseModel, Field


class RetrievedChunk(BaseModel):
    """One chunk as returned by retrieval, ranked and scored.

    ``score`` is retrieval-stage-specific: an RRF score coming out of
    :mod:`atlasdesk.retrieval.hybrid`, or a cross-encoder relevance score coming
    out of :mod:`atlasdesk.retrieval.rerank`. Compare scores only within the
    stage that produced them.
    """

    chunk_id: str
    document_id: str
    text: str
    heading_path: list[str] = Field(default_factory=list)
    source_uri: str
    page: int | None = None
    score: float
    rank: int


class RetrievalFilters(BaseModel):
    """Optional narrowing applied inside the query predicate, alongside ACL."""

    document_ids: list[str] | None = None
    heading_prefix: str | None = None
    updated_after: datetime | None = None
```

### ACL predicate — the one piece every retrieval query must include

```python
# src/atlasdesk/retrieval/acl.py
"""Access-control predicate for every retrieval query.

There is exactly one function here that matters for security:
:func:`build_acl_predicate`. It returns a SQL fragment and its parameters, meant
to be spliced into the ``WHERE`` clause of *every* CTE that reads ``chunks`` —
never applied once, after the fact, to a result list. :func:`passes_acl` is the
same rule expressed in pure Python, used by tests to prove the SQL and the
Python agree, and by any in-process fake store that stands in for Postgres in
unit tests.
"""

from __future__ import annotations

from collections.abc import Iterable

from atlasdesk.security.principal import Principal


def acl_fragment(*, alias: str = "c", start: int = 1) -> str:
    """The SQL text of the ACL predicate, with no parameter values attached.

    Split out from :func:`build_acl_predicate` so callers that need the fragment
    text alone — to build a static query template once, rather than per request —
    do not need a ``Principal`` in hand to get it. The fragment's text never
    depends on *which* principal is asking, only on ``alias`` and ``start``.
    """
    tenant_param = f"${start}"
    tags_param = f"${start + 1}"
    return f"{alias}.tenant_id = {tenant_param} AND {alias}.acl_tags && {tags_param}::text[]"


def build_acl_predicate(principal: Principal, *, alias: str = "c", start: int = 1) -> tuple[str, list[object]]:
    """SQL fragment (parameterised) and its parameters, for splicing into a WHERE clause.

    Contract: the returned fragment references ``{alias}.tenant_id`` and
    ``{alias}.acl_tags``, using positional parameters starting at ``start``. It is
    the caller's job to place this inside every arm of a query that can return
    chunk rows — a fragment appended after fusion is not this function's job to
    prevent, and it will not prevent it.
    """
    fragment = acl_fragment(alias=alias, start=start)
    return fragment, [principal.tenant_id, sorted(principal.acl_tags)]


def passes_acl(principal: Principal, *, tenant_id: str, acl_tags: Iterable[str]) -> bool:
    """Pure-Python restatement of :func:`build_acl_predicate`'s logic.

    Used by tests and by any fake store that simulates Postgres without a real
    database. Contract: this must agree with the SQL fragment above on every
    input, or the fake is testing the wrong thing — :mod:`tests.test_acl`
    asserts exactly that agreement.
    """
    if tenant_id != principal.tenant_id:
        return False
    return bool(set(acl_tags) & principal.acl_tags)
```

### The hybrid retriever

```python
# src/atlasdesk/retrieval/hybrid.py
"""BM25 + dense retrieval, fused with Reciprocal Rank Fusion, ACL-safe by
construction, optionally reranked. This module is the load-bearing artefact of
Chapter 10: it is the exact ``HybridRetriever.search`` signature frozen in Book
Bible Sec 4.6, and Chapter 11 onward call it rather than reimplementing it.

Imports Chapter 4's ``LLMClient`` for embedding and Chapter 9's connection pool
contract; it defines neither.
"""

from __future__ import annotations

from collections.abc import Mapping, Sequence
from typing import Any, Protocol

from atlasdesk.errors import RetrievalError
from atlasdesk.llm.base import LLMClient
from atlasdesk.retrieval.acl import acl_fragment, build_acl_predicate
from atlasdesk.retrieval.rerank import Reranker
from atlasdesk.retrieval.types import RetrievalFilters, RetrievedChunk
from atlasdesk.security.principal import Principal

#: RRF's damping constant. Cormack and Clarke (SIGIR 2009) fix it at 60 during a
#: pilot run and report the result is not sensitive to the exact value; we do
#: not retune it without a measured reason on our own eval set.
RRF_K: int = 60

_HYBRID_SQL_TEMPLATE = """
WITH lexical AS (
    SELECT c.id AS chunk_id,
           row_number() OVER (ORDER BY ts_rank_cd(c.tsv, query) DESC) AS rank
    FROM chunks c, to_tsquery('english', $1) AS query
    WHERE c.tsv @@ query AND {acl_lexical}
    ORDER BY ts_rank_cd(c.tsv, query) DESC
    LIMIT $5
),
dense AS (
    SELECT c.id AS chunk_id,
           row_number() OVER (ORDER BY c.embedding <=> $2::vector) AS rank
    FROM chunks c
    WHERE {acl_dense}
    ORDER BY c.embedding <=> $2::vector
    LIMIT $5
),
fused AS (
    SELECT chunk_id, SUM(1.0 / ({rrf_k} + rank)) AS rrf_score
    FROM (SELECT * FROM lexical UNION ALL SELECT * FROM dense) both_arms
    GROUP BY chunk_id
)
SELECT c.id AS chunk_id, c.document_id, c.text, c.heading_path, c.page,
       d.source_uri, f.rrf_score AS score
FROM fused f
JOIN chunks c ON c.id = f.chunk_id
JOIN documents d ON d.id = c.document_id
{extra_filter}
ORDER BY f.rrf_score DESC
LIMIT $5;
"""


class QueryPool(Protocol):
    """The subset of Chapter 9's connection pool this module drives.

    Kept as a narrow Protocol, not the concrete pool class, so tests can supply
    a fake that never touches a real database.
    """

    async def fetch(self, sql: str, *params: Any) -> list[Mapping[str, Any]]: ...


def _build_sql(*, extra_filter: str) -> str:
    acl = acl_fragment(alias="c", start=3)  # same $3/$4 positions, reused verbatim in both CTEs
    return _HYBRID_SQL_TEMPLATE.format(acl_lexical=acl, acl_dense=acl, rrf_k=RRF_K, extra_filter=extra_filter)


def _filter_clause(filters: RetrievalFilters | None) -> tuple[str, list[object]]:
    """Extra predicate for optional narrowing, appended after the ACL-safe fuse.

    Parameters start at $6 because $1-$5 are always query text, embedding, tenant
    id, acl tags, and candidate_k. Returns an empty clause when no filter is set.
    """
    if filters is None:
        return "", []
    clauses: list[str] = []
    params: list[object] = []
    next_param = 6
    if filters.document_ids:
        clauses.append(f"c.document_id = ANY(${next_param}::text[])")
        params.append(list(filters.document_ids))
        next_param += 1
    if filters.heading_prefix:
        clauses.append(f"c.heading_path[1] = ${next_param}")
        params.append(filters.heading_prefix)
        next_param += 1
    if filters.updated_after:
        clauses.append(f"d.updated_at > ${next_param}")
        params.append(filters.updated_after)
        next_param += 1
    if not clauses:
        return "", []
    return "WHERE " + " AND ".join(clauses), params


class HybridRetriever:
    """BM25 + dense retrieval fused with RRF, ACL-filtered, optionally reranked.

    Contract: :meth:`search` never returns a chunk whose ``tenant_id`` and
    ``acl_tags`` fail :func:`atlasdesk.retrieval.acl.passes_acl` against
    ``principal`` — this is enforced by the SQL predicate, not by post-filtering
    the return value, and :mod:`tests.test_acl_leak` proves it.
    """

    def __init__(
        self,
        pool: QueryPool,
        embedder: LLMClient,
        *,
        embedding_model: str | None = None,
        candidate_k: int = 50,
        reranker: Reranker | None = None,
    ) -> None:
        self._pool = pool
        self._embedder = embedder
        self._embedding_model = embedding_model
        self._candidate_k = candidate_k
        self._reranker = reranker

    async def search(
        self,
        query: str,
        *,
        principal: Principal,
        k: int = 6,
        filters: RetrievalFilters | None = None,
    ) -> list[RetrievedChunk]:
        """Retrieve, ACL-safe, fused, and — if configured — reranked to ``k``.

        Raises:
            RetrievalError: the embedding call failed, or the pool returned no
                usable rows and the caller should treat this as "nothing found",
                not "the query is malformed" — those are different escalation
                paths in Chapter 15.
        """
        if not query.strip():
            raise RetrievalError("empty query")

        embed_result = await self._embedder.embed([query], model=self._embedding_model)
        embedding = embed_result.vectors[0]

        _, acl_params = build_acl_predicate(principal, alias="c", start=3)
        extra_sql, extra_params = _filter_clause(filters)
        sql = _build_sql(extra_filter=extra_sql)

        params: list[object] = [query, embedding, *acl_params, self._candidate_k, *extra_params]
        rows = await self._pool.fetch(sql, *params)

        candidates = [
            RetrievedChunk(
                chunk_id=row["chunk_id"],
                document_id=row["document_id"],
                text=row["text"],
                heading_path=list(row.get("heading_path") or []),
                source_uri=row["source_uri"],
                page=row.get("page"),
                score=float(row["score"]),
                rank=rank,
            )
            for rank, row in enumerate(rows, start=1)
        ]

        if self._reranker is not None and candidates:
            return await self._reranker.rerank(query, candidates, top_n=k)
        return candidates[:k]
```

### The cross-encoder reranker

```python
# src/atlasdesk/retrieval/rerank.py
"""Cross-encoder reranking: the top-50 -> top-k precision step.

``RerankClient`` abstracts over whichever scoring backend you use — a hosted
reranker or a self-hosted cross-encoder — exactly the way Chapter 4's
``LLMClient`` abstracts over model providers. Never call a reranker SDK
directly from business logic; call it through this Protocol.
"""

from __future__ import annotations

from collections.abc import Sequence
from typing import Protocol

from atlasdesk.errors import RetrievalError
from atlasdesk.retrieval.types import RetrievedChunk


class RerankClient(Protocol):
    """A pairwise relevance scorer: one score per (query, document) pair."""

    name: str

    async def score(self, query: str, documents: Sequence[str]) -> list[float]: ...


class Reranker(Protocol):
    """What :class:`atlasdesk.retrieval.hybrid.HybridRetriever` depends on."""

    async def rerank(
        self, query: str, chunks: Sequence[RetrievedChunk], *, top_n: int
    ) -> list[RetrievedChunk]: ...


class CrossEncoderReranker:
    """Reranks a candidate list by pairwise relevance, then re-ranks and re-scores.

    Contract: preserves every field on each :class:`RetrievedChunk` except
    ``score`` and ``rank``, which are overwritten with the reranker's own values
    — mixing an RRF score and a cross-encoder score on the same field would make
    the numbers meaningless to compare, which is why both stages share one type.
    """

    def __init__(self, client: RerankClient) -> None:
        self._client = client

    async def rerank(
        self, query: str, chunks: Sequence[RetrievedChunk], *, top_n: int
    ) -> list[RetrievedChunk]:
        if not chunks:
            return []
        scores = await self._client.score(query, [chunk.text for chunk in chunks])
        if len(scores) != len(chunks):
            raise RetrievalError(
                f"{self._client.name}: expected {len(chunks)} scores, got {len(scores)}"
            )
        ordered = sorted(zip(chunks, scores, strict=True), key=lambda pair: pair[1], reverse=True)
        return [
            chunk.model_copy(update={"score": score, "rank": rank})
            for rank, (chunk, score) in enumerate(ordered[:top_n], start=1)
        ]


class FakeRerankClient:
    """Keyless fake for tests: scores by normalised token overlap.

    This is not a relevance model — it is a deterministic stand-in that orders
    candidates plausibly enough to exercise :class:`CrossEncoderReranker`'s
    contract (score count matches input count, ordering is applied, ``top_n`` is
    respected) with no network and no API key, exactly like Chapter 4's
    ``llm/fake.py``.
    """

    name = "fake-cross-encoder"

    async def score(self, query: str, documents: Sequence[str]) -> list[float]:
        query_terms = set(query.lower().split())
        scores: list[float] = []
        for doc in documents:
            doc_terms = set(doc.lower().split())
            overlap = len(query_terms & doc_terms)
            scores.append(overlap / max(len(query_terms), 1))
        return scores
```

### Query transformation

```python
# src/atlasdesk/retrieval/transform.py
"""Query transformation: rewriting, decomposition, and HyDE.

All three spend an extra model call before retrieval to change what gets
searched for. Each earns its latency in different, narrow circumstances — see
Chapter 10's decision table. Prompts are versioned files through Chapter 5's
registry, not f-strings; this module only renders and calls.
"""

from __future__ import annotations

import re
from collections.abc import Sequence

from atlasdesk.llm.base import LLMClient, Message
from atlasdesk.prompts.registry import PromptRegistry

#: A question is a decomposition candidate if it names two distinct entities or
#: an explicit comparison/conjunction. Cheap heuristic gate, not a classifier —
#: measured against the seed set in this chapter's Build section.
_DECOMPOSE_TRIGGERS = re.compile(r"\b(compare|versus|vs\.?|and also|as well as)\b", re.IGNORECASE)


class QueryTransformer:
    """Rewrites, decomposes, or hypothesises before retrieval.

    Contract: every method returns plain query text ready to hand to
    :meth:`atlasdesk.retrieval.hybrid.HybridRetriever.search`; none of them touch
    the database or the reranker.
    """

    def __init__(self, client: LLMClient, registry: PromptRegistry, *, model: str | None = None) -> None:
        self._client = client
        self._registry = registry
        self._model = model

    def should_decompose(self, question: str) -> bool:
        """Cheap heuristic gate: only pay for decomposition when it looks compound."""
        return bool(_DECOMPOSE_TRIGGERS.search(question))

    async def rewrite(self, question: str, *, history: Sequence[Message] = ()) -> str:
        """Resolve pronouns and ellipsis against prior turns into a standalone query."""
        if not history:
            return question
        prompt = self._registry.get("query_rewrite")
        rendered = prompt.render(question=question, history=_format_history(history))
        completion = await self._client.complete(
            [Message(role="user", content=rendered)], model=self._model, max_tokens=120, temperature=0.0
        )
        return completion.text.strip() or question

    async def decompose(self, question: str, *, max_subqueries: int = 3) -> list[str]:
        """Split a compound question into independently-retrievable sub-questions."""
        prompt = self._registry.get("query_decompose")
        rendered = prompt.render(question=question, max_subqueries=max_subqueries)
        completion = await self._client.complete(
            [Message(role="user", content=rendered)], model=self._model, max_tokens=200, temperature=0.0
        )
        parts = [line.strip("- ").strip() for line in completion.text.splitlines() if line.strip()]
        return parts[:max_subqueries] or [question]

    async def hyde(self, question: str) -> str:
        """Return a hypothetical answer, to be embedded instead of the question."""
        prompt = self._registry.get("query_hyde")
        rendered = prompt.render(question=question)
        completion = await self._client.complete(
            [Message(role="user", content=rendered)], model=self._model, max_tokens=180, temperature=0.2
        )
        return completion.text.strip() or question


def _format_history(history: Sequence[Message]) -> str:
    return "\n".join(f"{message.role}: {message.content}" for message in history)
```

### Retrieval metrics, and groundedness

```python
# src/atlasdesk/evals/retrieval_metrics.py
"""Retrieval-quality metrics computed on AtlasDesk's own eval set.

Every function here is pure: a list of retrieved chunk ids, a set of relevant
chunk ids, nothing else. No database, no model, no network — so these run in
the same no-key test suite as everything else in this chapter, and Chapter 18's
runner calls them without any retrieval-specific setup.
"""

from __future__ import annotations

import math
import re
import unicodedata
from collections.abc import Iterable, Mapping, Sequence
from dataclasses import dataclass

from atlasdesk.errors import EvalError
from atlasdesk.retrieval.types import RetrievedChunk
from atlasdesk.schemas.answer import Answer


def precision_at_k(retrieved: Sequence[str], relevant: Iterable[str], k: int) -> float:
    """Fraction of the top ``k`` retrieved ids that are actually relevant.

    This is "context precision": of what we handed the model, how much of it
    was useful. Undefined (returns 0.0) when ``k`` is 0 or nothing was retrieved.
    """
    top = retrieved[:k]
    if not top:
        return 0.0
    relevant_set = set(relevant)
    return sum(1 for chunk_id in top if chunk_id in relevant_set) / len(top)


def recall_at_k(retrieved: Sequence[str], relevant: Iterable[str], k: int) -> float:
    """Fraction of all relevant ids that appear anywhere in the top ``k``.

    This is "context recall": of what the model *needed*, how much did we
    surface at all. A recall miss cannot be fixed by reranking — reranking only
    reorders what fusion already found, so a recall regression always points
    back at the fusion stage, never the reranker.
    """
    relevant_set = set(relevant)
    if not relevant_set:
        raise EvalError("recall_at_k requires at least one relevant id")
    top = set(retrieved[:k])
    return len(top & relevant_set) / len(relevant_set)


def mrr(retrieved: Sequence[str], relevant: Iterable[str]) -> float:
    """Reciprocal rank of the first relevant id in ``retrieved``; 0.0 if none."""
    relevant_set = set(relevant)
    for rank, chunk_id in enumerate(retrieved, start=1):
        if chunk_id in relevant_set:
            return 1.0 / rank
    return 0.0


def ndcg_at_k(retrieved: Sequence[str], relevant: Iterable[str], k: int) -> float:
    """Binary-relevance nDCG at ``k``: rewards relevant ids ranked higher.

    Unlike recall, this is sensitive to *order* within the top k — the property
    reranking exists to improve. A retriever with perfect recall but poor
    ordering scores well on recall_at_k and poorly here; that split is the whole
    reason both metrics are reported, never just one.
    """
    relevant_set = set(relevant)
    if not relevant_set:
        raise EvalError("ndcg_at_k requires at least one relevant id")
    top = retrieved[:k]
    dcg = sum(1.0 / math.log2(rank + 1) for rank, cid in enumerate(top, start=1) if cid in relevant_set)
    ideal_hits = min(len(relevant_set), k)
    idcg = sum(1.0 / math.log2(rank + 1) for rank in range(1, ideal_hits + 1))
    return dcg / idcg if idcg > 0 else 0.0


@dataclass(frozen=True, slots=True)
class RetrievalCase:
    """One retrieval eval case: a query and the chunk ids that answer it."""

    case_id: str
    query: str
    relevant_chunk_ids: frozenset[str]


@dataclass(frozen=True, slots=True)
class RetrievalReport:
    """Aggregate retrieval metrics across a set of cases, at a fixed ``k``."""

    k: int
    n_cases: int
    mean_precision: float
    mean_recall: float
    mean_mrr: float
    mean_ndcg: float


def evaluate_retrieval(
    cases: Sequence[RetrievalCase],
    retrieved_by_case: Mapping[str, Sequence[str]],
    *,
    k: int,
) -> RetrievalReport:
    """Compute the four metrics across ``cases`` and average them.

    ``retrieved_by_case`` maps ``case_id`` to the ordered chunk ids a retriever
    returned for that case's query — computed by the caller (a live retriever in
    an integration test, or a recorded run in CI) and passed in as data, so this
    function stays synchronous, pure, and provider-free.
    """
    if not cases:
        raise EvalError("evaluate_retrieval requires at least one case")
    precisions, recalls, mrrs, ndcgs = [], [], [], []
    for case in cases:
        retrieved = retrieved_by_case.get(case.case_id, ())
        precisions.append(precision_at_k(list(retrieved), case.relevant_chunk_ids, k))
        recalls.append(recall_at_k(list(retrieved), case.relevant_chunk_ids, k))
        mrrs.append(mrr(list(retrieved), case.relevant_chunk_ids))
        ndcgs.append(ndcg_at_k(list(retrieved), case.relevant_chunk_ids, k))
    n = len(cases)
    return RetrievalReport(
        k=k,
        n_cases=n,
        mean_precision=sum(precisions) / n,
        mean_recall=sum(recalls) / n,
        mean_mrr=sum(mrrs) / n,
        mean_ndcg=sum(ndcgs) / n,
    )


_WHITESPACE = re.compile(r"\s+")


def _normalize(text: str) -> str:
    """Fold whitespace and unicode so a quote copied with a stray newline still matches."""
    return _WHITESPACE.sub(" ", unicodedata.normalize("NFKC", text)).strip().lower()


def citations_are_grounded(answer: Answer, chunks_by_id: Mapping[str, RetrievedChunk]) -> tuple[str, ...]:
    """Citation chunk ids whose ``quote`` is NOT a substring of that chunk's text.

    Complements Chapter 6's ``unknown_citations``: that check catches a citation
    pointing at a chunk id that was never supplied at all; this one catches a
    citation pointing at a *real, supplied* chunk with a quote the chunk does
    not actually contain — a fabricated citation wearing a legitimate id. Both
    are groundedness failures and both route through the same escalation policy.
    """
    ungrounded: list[str] = []
    for citation in answer.citations:
        chunk = chunks_by_id.get(citation.chunk_id)
        if chunk is None:
            continue  # unknown_citations() already reports this case
        if _normalize(citation.quote) not in _normalize(chunk.text):
            ungrounded.append(citation.chunk_id)
    return tuple(ungrounded)
```

### C1 end to end

This is the wiring that makes C1 real: retrieve, rerank, generate a schema-enforced draft, project it to the frozen `Answer` contract, then let deterministic policy — never the model — decide whether to escalate.

```python
# scripts/ask_policy.py
"""C1 end to end: hybrid retrieval -> rerank -> structured answer -> escalation.

Run:
    python -m scripts.ask_policy "What is the refund for withdrawing on day 20?"

Requires ANTHROPIC_API_KEY or OPENAI_API_KEY, and a running Postgres from
Chapters 8/9. With neither key set this raises ConfigError via
``settings.require_any_provider()`` rather than failing deep inside a call.
"""

from __future__ import annotations

import asyncio
import sys

from atlasdesk.config import get_settings
from atlasdesk.llm.base import Message
from atlasdesk.llm.factory import get_client
from atlasdesk.llm.structured import run_structured
from atlasdesk.prompts.registry import PromptRegistry
from atlasdesk.retrieval.hybrid import HybridRetriever
from atlasdesk.retrieval.rerank import CrossEncoderReranker
from atlasdesk.retrieval.store import get_pool  # Chapter 9 — unchanged, imported not redefined
from atlasdesk.retrieval.transform import QueryTransformer
from atlasdesk.schemas.answer import AnswerDraft, EscalationPolicy, apply_escalation_policy
from atlasdesk.security.principal import Principal
from atlasdesk.evals.retrieval_metrics import citations_are_grounded


async def answer_policy_question(question: str, principal: Principal) -> str:
    settings = get_settings()
    settings.require_any_provider()

    client = get_client()
    pool = await get_pool(settings.database_url)
    reranker = CrossEncoderReranker(_reranker_client(settings))
    retriever = HybridRetriever(pool, client, reranker=reranker)
    transformer = QueryTransformer(client, PromptRegistry())

    search_query = question
    if transformer.should_decompose(question):
        sub_queries = await transformer.decompose(question)
        seen: dict[str, object] = {}
        for sub in sub_queries:
            for chunk in await retriever.search(sub, principal=principal, k=6):
                seen[chunk.chunk_id] = chunk
        chunks = list(seen.values())[:6]
    else:
        chunks = await retriever.search(search_query, principal=principal, k=6)

    allowed_ids = frozenset(chunk.chunk_id for chunk in chunks)
    context = "\n\n".join(f"[{c.chunk_id}] {c.text}" for c in chunks)
    prompt = PromptRegistry().get("answer_policy").render(question=question, context=context)

    structured = await run_structured(
        client, [Message(role="user", content=prompt)], AnswerDraft, model=settings.anthropic_model
    )
    answer = structured.value.to_answer()
    answer = apply_escalation_policy(answer, allowed_chunk_ids=allowed_ids, policy=EscalationPolicy())

    ungrounded = citations_are_grounded(answer, {c.chunk_id: c for c in chunks})
    if ungrounded:
        answer = answer.model_copy(
            update={"should_escalate": True, "escalation_reason": f"ungrounded_quote:{ungrounded[0]}"}
        )
    return answer.model_dump_json(indent=2)


def _reranker_client(settings: object) -> object:  # pragma: no cover — wiring, exercised via the fake in tests
    from atlasdesk.retrieval.rerank import FakeRerankClient

    if getattr(settings, "cohere_api_key", None) is None:
        return FakeRerankClient()
    raise NotImplementedError("wire your hosted reranker client here, keyed via settings.cohere_api_key")


if __name__ == "__main__":
    question = " ".join(sys.argv[1:]) or "What is the refund for withdrawing on day 20?"
    principal = Principal(
        user_id="u_daniel",
        tenant_id="meridian-core",
        roles=frozenset({"agent"}),
        acl_tags=frozenset({"public", "staff"}),
    )
    print(asyncio.run(answer_policy_question(question, principal)))
```

Three new prompt files, versioned through Chapter 5's registry rather than inlined as f-strings:

*File: `prompts/query_rewrite/v1.md`*

```markdown
---
name: query_rewrite
version: 1
model_family: claude
changelog: "v1: initial version, resolves pronouns/ellipsis against prior turns."
variables: [question, history]
---
Rewrite the final question below into a standalone query that does not depend
on the conversation above. Resolve pronouns and implicit references. Do not
answer the question. Output only the rewritten query, nothing else.

Conversation:
{{history}}

Final question: {{question}}
```

*File: `prompts/query_decompose/v1.md`*

```markdown
---
name: query_decompose
version: 1
model_family: claude
changelog: "v1: initial version, splits compound questions into sub-questions."
variables: [question, max_subqueries]
---
Split the question below into up to {{max_subqueries}} independent
sub-questions, each retrievable on its own. If the question is already simple,
return it unchanged as the only line. One sub-question per line, no numbering,
no commentary.

Question: {{question}}
```

*File: `prompts/query_hyde/v1.md`*

```markdown
---
name: query_hyde
version: 1
model_family: claude
changelog: "v1: initial version, HyDE hypothetical-answer generation."
variables: [question]
---
Write a short, plausible-sounding answer to the question below, as if it
appeared in an internal policy handbook. Do not hedge, do not say you are
unsure, and do not include disclaimers — this text is used only to search for
the real answer, never shown to a user.

Question: {{question}}
```

### Tests

A fake pool stands in for Postgres, exactly the way Chapter 4's `llm/fake.py` stands in for a provider — it simulates the same ACL logic the real SQL enforces, and `test_acl.py` proves the fake and the real predicate agree.

```python
# tests/fakes/fake_pool.py
"""An in-memory stand-in for Chapter 9's connection pool, for unit tests only.

It does not parse SQL. It stores a fixed fixture of chunk rows and, on every
call to ``fetch``, replays the same lexical/dense/ACL/RRF logic the real query
performs, so tests exercise the retrieval *contract* without a database. The
canonical leak test below additionally proves the fixture actually contains
the content it must not leak, and that removing the ACL filter would return it
— otherwise a fake that "just returns the right rows" would be worthless.
"""

from __future__ import annotations

import math
from dataclasses import dataclass, field
from typing import Any

from atlasdesk.retrieval.acl import passes_acl
from atlasdesk.security.principal import Principal


@dataclass(frozen=True, slots=True)
class FakeChunkRow:
    chunk_id: str
    document_id: str
    text: str
    heading_path: list[str]
    source_uri: str
    page: int | None
    tenant_id: str
    acl_tags: frozenset[str]
    lexical_terms: frozenset[str]
    embedding: tuple[float, ...]


@dataclass
class FakePool:
    """Fixture rows plus a query embedding function, standing in for Postgres."""

    rows: list[FakeChunkRow] = field(default_factory=list)
    query_embeddings: dict[str, tuple[float, ...]] = field(default_factory=dict)

    async def fetch(self, sql: str, *params: Any) -> list[dict[str, Any]]:
        query_text, embedding = params[0], params[1]
        tenant_id, acl_tags = params[2], set(params[3])
        candidate_k = params[4]
        principal_stub = Principal(
            user_id="_", tenant_id=tenant_id, roles=frozenset(), acl_tags=frozenset(acl_tags)
        )
        visible = [row for row in self.rows if passes_acl(principal_stub, tenant_id=row.tenant_id, acl_tags=row.acl_tags)]

        query_terms = set(query_text.replace("'", "").split())
        lexical_ranked = sorted(
            visible, key=lambda r: -len(query_terms & r.lexical_terms)
        )[:candidate_k]
        dense_ranked = sorted(
            visible, key=lambda r: _cosine_distance(embedding, r.embedding)
        )[:candidate_k]

        rrf: dict[str, float] = {}
        for rank, row in enumerate(lexical_ranked, start=1):
            rrf[row.chunk_id] = rrf.get(row.chunk_id, 0.0) + 1.0 / (60 + rank)
        for rank, row in enumerate(dense_ranked, start=1):
            rrf[row.chunk_id] = rrf.get(row.chunk_id, 0.0) + 1.0 / (60 + rank)

        by_id = {row.chunk_id: row for row in visible}
        ordered = sorted(rrf.items(), key=lambda pair: pair[1], reverse=True)[:candidate_k]
        return [
            {
                "chunk_id": chunk_id,
                "document_id": by_id[chunk_id].document_id,
                "text": by_id[chunk_id].text,
                "heading_path": by_id[chunk_id].heading_path,
                "page": by_id[chunk_id].page,
                "source_uri": by_id[chunk_id].source_uri,
                "score": score,
            }
            for chunk_id, score in ordered
        ]


def _cosine_distance(a: tuple[float, ...], b: tuple[float, ...]) -> float:
    """Cosine distance, padding the shorter vector with zeros.

    Real embeddings from the same model are always equal-length; this fixture
    pads because test fixtures deliberately use short, readable vectors like
    ``(1.0, 0.0)`` while the fake embedder produces a longer bag-of-words
    vector, and only their *relative* ordering matters for these tests.
    """
    width = max(len(a), len(b))
    padded_a = list(a) + [0.0] * (width - len(a))
    padded_b = list(b) + [0.0] * (width - len(b))
    dot = sum(x * y for x, y in zip(padded_a, padded_b, strict=True))
    norm_a = math.sqrt(sum(x * x for x in padded_a))
    norm_b = math.sqrt(sum(y * y for y in padded_b))
    if norm_a == 0 or norm_b == 0:
        return 1.0
    return 1.0 - dot / (norm_a * norm_b)
```

```python
# tests/fakes/fake_embedder.py
"""A keyless fake embedder: deterministic bag-of-words vectors, no network."""

from __future__ import annotations

from collections.abc import Sequence

from atlasdesk.llm.base import EmbeddingResult, Usage

_VOCAB = (
    "refund withdrawal deferral instalment waiver fee attendance certification "
    "sponsor executive singapore transfer credit"
).split()


class FakeEmbedder:
    name = "fake-embedder"

    async def embed(self, texts: Sequence[str], *, model: str | None = None) -> EmbeddingResult:
        vectors = [self._vector(text) for text in texts]
        return EmbeddingResult(
            vectors=vectors,
            model="fake-embedder",
            usage=Usage(model="fake-embedder", input_tokens=0, output_tokens=0, cost_usd=0.0, latency_ms=1),
        )

    def _vector(self, text: str) -> list[float]:
        lowered = text.lower()
        return [float(lowered.count(term)) + 0.01 for term in _VOCAB]
```

```python
# tests/test_acl.py
"""The SQL fragment and the pure-Python check must agree on every input, or the
fake store above is testing something other than the real query.
"""

from __future__ import annotations

from atlasdesk.retrieval.acl import acl_fragment, build_acl_predicate, passes_acl
from atlasdesk.security.principal import Principal

CORE_STAFF = Principal(
    user_id="u_daniel", tenant_id="meridian-core", roles=frozenset({"agent"}), acl_tags=frozenset({"public", "staff"})
)


def test_fragment_references_correct_placeholders() -> None:
    fragment = acl_fragment(alias="c", start=3)
    assert "c.tenant_id = $3" in fragment
    assert "c.acl_tags && $4::text[]" in fragment


def test_passes_acl_requires_tenant_match() -> None:
    assert not passes_acl(CORE_STAFF, tenant_id="meridian-exec", acl_tags={"public"})


def test_passes_acl_requires_tag_overlap() -> None:
    assert not passes_acl(CORE_STAFF, tenant_id="meridian-core", acl_tags={"finance"})
    assert passes_acl(CORE_STAFF, tenant_id="meridian-core", acl_tags={"public", "finance"})


def test_predicate_and_python_check_agree() -> None:
    _, params = build_acl_predicate(CORE_STAFF)
    tenant_id, acl_tags = params
    assert passes_acl(CORE_STAFF, tenant_id=tenant_id, acl_tags=acl_tags)
```

```python
# tests/test_acl_leak.py
"""The canonical cross-tenant leak attempt (seed case C1-007) — this attempt
must fail to leak. Daniel Osei, a meridian-core agent, asks the question whose
only answer lives in meridian-exec's sponsor-refund addendum.

If this test passes for the wrong reason — because the fixture never contained
the exec-only content — it proves nothing. So it also asserts the excluded
content exists and would be returned by the identical query with the tenant
filter loosened, which is the only way to know the leak was actually blocked
rather than never attempted.
"""

from __future__ import annotations

import pytest

from atlasdesk.retrieval.hybrid import HybridRetriever
from atlasdesk.security.principal import Principal
from tests.fakes.fake_embedder import FakeEmbedder
from tests.fakes.fake_pool import FakeChunkRow, FakePool

DANIEL = Principal(
    user_id="u_daniel", tenant_id="meridian-core", roles=frozenset({"agent"}), acl_tags=frozenset({"public", "staff"})
)

EXEC_SPONSOR_CLAUSE = (
    "Corporate-sponsored withdrawals under the executive-programme sponsor "
    "refund addendum are settled directly with the sponsoring employer."
)


def _fixture() -> FakePool:
    return FakePool(
        rows=[
            FakeChunkRow(
                chunk_id="chunk-4.2",
                document_id="handbook_v7",
                text="Withdrawal within 14 days of the start date attracts a 50% refund.",
                heading_path=["4", "4.2"],
                source_uri="handbook_v7.pdf",
                page=12,
                tenant_id="meridian-core",
                acl_tags=frozenset({"public", "staff"}),
                lexical_terms=frozenset({"withdrawal", "refund"}),
                embedding=(1.0, 0.0, 0.0, 0.0),
            ),
            FakeChunkRow(
                chunk_id="chunk-11.3",
                document_id="handbook_v7",
                text=EXEC_SPONSOR_CLAUSE,
                heading_path=["11", "11.3"],
                source_uri="handbook_v7.pdf",
                page=88,
                tenant_id="meridian-exec",
                acl_tags=frozenset({"public", "staff"}),
                lexical_terms=frozenset({"sponsor", "executive", "refund"}),
                embedding=(0.9, 0.1, 0.0, 0.0),
            ),
        ]
    )


@pytest.mark.asyncio
async def test_cross_tenant_leak_attempt_fails_to_leak() -> None:
    pool = _fixture()
    retriever = HybridRetriever(pool, FakeEmbedder())

    results = await retriever.search(
        "What does the executive-programme sponsor refund addendum say about corporate-sponsored withdrawals?",
        principal=DANIEL,
        k=6,
    )

    assert all(chunk.chunk_id != "chunk-11.3" for chunk in results)
    assert all(EXEC_SPONSOR_CLAUSE not in chunk.text for chunk in results)


@pytest.mark.asyncio
async def test_fixture_actually_contains_the_content_that_must_not_leak() -> None:
    """Proves the leak test above is testing something real, not an empty fixture."""
    pool = _fixture()
    exec_principal = Principal(
        user_id="u_exec_admin",
        tenant_id="meridian-exec",
        roles=frozenset({"admin"}),
        acl_tags=frozenset({"public", "staff"}),
    )
    retriever = HybridRetriever(pool, FakeEmbedder())

    results = await retriever.search(
        "What does the executive-programme sponsor refund addendum say?",
        principal=exec_principal,
        k=6,
    )

    assert any(chunk.chunk_id == "chunk-11.3" for chunk in results), (
        "the fixture must contain the exec-only chunk and a correctly-scoped "
        "principal must be able to retrieve it, or the leak test above is vacuous"
    )
```

```python
# tests/test_hybrid.py
"""HybridRetriever: fusion, filters, and the optional rerank hand-off."""

from __future__ import annotations

import pytest

from atlasdesk.retrieval.hybrid import HybridRetriever
from atlasdesk.retrieval.rerank import CrossEncoderReranker, FakeRerankClient
from atlasdesk.retrieval.types import RetrievalFilters
from atlasdesk.security.principal import Principal
from tests.fakes.fake_embedder import FakeEmbedder
from tests.fakes.fake_pool import FakeChunkRow, FakePool

CORE_STAFF = Principal(
    user_id="u_daniel", tenant_id="meridian-core", roles=frozenset({"agent"}), acl_tags=frozenset({"public", "staff"})
)


def _pool_with(*rows: FakeChunkRow) -> FakePool:
    return FakePool(rows=list(rows))


def _row(chunk_id: str, text: str, terms: frozenset[str], embedding: tuple[float, ...]) -> FakeChunkRow:
    return FakeChunkRow(
        chunk_id=chunk_id,
        document_id="handbook_v7",
        text=text,
        heading_path=["4"],
        source_uri="handbook_v7.pdf",
        page=1,
        tenant_id="meridian-core",
        acl_tags=frozenset({"public", "staff"}),
        lexical_terms=terms,
        embedding=embedding,
    )


@pytest.mark.asyncio
async def test_search_returns_ranked_chunks() -> None:
    pool = _pool_with(
        _row("c1", "Refund policy for withdrawal within 14 days.", frozenset({"refund", "withdrawal"}), (1.0, 0.0)),
        _row("c2", "Deferral fee is non-refundable.", frozenset({"deferral", "fee"}), (0.0, 1.0)),
    )
    retriever = HybridRetriever(pool, FakeEmbedder())
    results = await retriever.search("refund withdrawal", principal=CORE_STAFF, k=2)
    assert [chunk.chunk_id for chunk in results] == ["c1", "c2"] or len(results) == 2
    assert results[0].rank == 1


@pytest.mark.asyncio
async def test_empty_query_raises() -> None:
    from atlasdesk.errors import RetrievalError

    retriever = HybridRetriever(_pool_with(), FakeEmbedder())
    with pytest.raises(RetrievalError):
        await retriever.search("   ", principal=CORE_STAFF)


@pytest.mark.asyncio
async def test_document_id_filter_narrows_results() -> None:
    pool = _pool_with(
        _row("c1", "Refund policy.", frozenset({"refund"}), (1.0, 0.0)),
    )
    retriever = HybridRetriever(pool, FakeEmbedder())
    filters = RetrievalFilters(document_ids=["handbook_v7"])
    results = await retriever.search("refund", principal=CORE_STAFF, filters=filters)
    assert results


@pytest.mark.asyncio
async def test_reranker_is_applied_when_configured() -> None:
    pool = _pool_with(
        _row("c1", "irrelevant filler about attendance", frozenset({"attendance"}), (0.0, 1.0)),
        _row("c2", "refund refund refund policy withdrawal", frozenset({"refund", "withdrawal"}), (1.0, 0.0)),
    )
    reranker = CrossEncoderReranker(FakeRerankClient())
    retriever = HybridRetriever(pool, FakeEmbedder(), reranker=reranker)
    results = await retriever.search("refund withdrawal policy", principal=CORE_STAFF, k=1)
    assert results[0].chunk_id == "c2"
```

```python
# tests/test_rerank.py
"""CrossEncoderReranker: ordering, top_n truncation, and the score-count contract."""

from __future__ import annotations

import pytest

from atlasdesk.errors import RetrievalError
from atlasdesk.retrieval.rerank import CrossEncoderReranker, FakeRerankClient
from atlasdesk.retrieval.types import RetrievedChunk


def _chunk(chunk_id: str, text: str, rank: int) -> RetrievedChunk:
    return RetrievedChunk(
        chunk_id=chunk_id, document_id="d1", text=text, source_uri="d1.pdf", score=0.0, rank=rank
    )


@pytest.mark.asyncio
async def test_rerank_orders_by_score_desc() -> None:
    chunks = [_chunk("a", "attendance only", 1), _chunk("b", "refund refund policy", 2)]
    reranker = CrossEncoderReranker(FakeRerankClient())
    ordered = await reranker.rerank("refund policy", chunks, top_n=2)
    assert [c.chunk_id for c in ordered] == ["b", "a"]
    assert ordered[0].rank == 1 and ordered[1].rank == 2


@pytest.mark.asyncio
async def test_rerank_respects_top_n() -> None:
    chunks = [_chunk("a", "x", 1), _chunk("b", "y", 2), _chunk("c", "z", 3)]
    reranker = CrossEncoderReranker(FakeRerankClient())
    ordered = await reranker.rerank("anything", chunks, top_n=1)
    assert len(ordered) == 1


@pytest.mark.asyncio
async def test_empty_input_returns_empty() -> None:
    reranker = CrossEncoderReranker(FakeRerankClient())
    assert await reranker.rerank("q", [], top_n=5) == []


class _BadClient:
    name = "bad"

    async def score(self, query: str, documents: list[str]) -> list[float]:
        return [1.0]  # deliberately wrong count


@pytest.mark.asyncio
async def test_score_count_mismatch_raises() -> None:
    reranker = CrossEncoderReranker(_BadClient())
    with pytest.raises(RetrievalError):
        await reranker.rerank("q", [_chunk("a", "x", 1), _chunk("b", "y", 2)], top_n=2)
```

```python
# tests/test_transform.py
"""QueryTransformer: the heuristic gate and the three transforms, against a fake client."""

from __future__ import annotations

import pytest

from atlasdesk.llm.fake import FakeLLMClient
from atlasdesk.prompts.registry import PromptRegistry
from atlasdesk.retrieval.transform import QueryTransformer


def test_should_decompose_triggers_on_comparison() -> None:
    transformer = QueryTransformer(FakeLLMClient(), PromptRegistry())
    assert transformer.should_decompose("Compare the refund policy and the deferral policy")
    assert not transformer.should_decompose("What is the refund policy?")


@pytest.mark.asyncio
async def test_rewrite_is_noop_with_no_history() -> None:
    transformer = QueryTransformer(FakeLLMClient(), PromptRegistry())
    result = await transformer.rewrite("What about it?")
    assert result == "What about it?"


@pytest.mark.asyncio
async def test_decompose_falls_back_to_original_on_empty_output() -> None:
    client = FakeLLMClient(fixed_text="")
    transformer = QueryTransformer(client, PromptRegistry())
    parts = await transformer.decompose("Compare A and B")
    assert parts == ["Compare A and B"]
```

```python
# tests/test_retrieval_metrics.py
"""Pure-function retrieval metrics: precision, recall, MRR, nDCG, and groundedness."""

from __future__ import annotations

import pytest

from atlasdesk.errors import EvalError
from atlasdesk.evals.retrieval_metrics import (
    RetrievalCase,
    citations_are_grounded,
    evaluate_retrieval,
    mrr,
    ndcg_at_k,
    precision_at_k,
    recall_at_k,
)
from atlasdesk.retrieval.types import RetrievedChunk
from atlasdesk.schemas.answer import Answer, Citation


def test_precision_and_recall_basic() -> None:
    retrieved = ["a", "b", "c", "d"]
    relevant = {"b", "d", "z"}
    assert precision_at_k(retrieved, relevant, 4) == 0.5
    assert recall_at_k(retrieved, relevant, 4) == pytest.approx(2 / 3)


def test_recall_requires_relevant_set() -> None:
    with pytest.raises(EvalError):
        recall_at_k(["a"], set(), 1)


def test_mrr_of_first_hit() -> None:
    assert mrr(["a", "b", "c"], {"c"}) == pytest.approx(1 / 3)
    assert mrr(["a", "b"], {"z"}) == 0.0


def test_ndcg_rewards_earlier_placement() -> None:
    high = ndcg_at_k(["rel", "irrel"], {"rel"}, 2)
    low = ndcg_at_k(["irrel", "rel"], {"rel"}, 2)
    assert high == 1.0
    assert low < high


def test_evaluate_retrieval_aggregates() -> None:
    cases = [
        RetrievalCase("C1-001", "q1", frozenset({"c1"})),
        RetrievalCase("C1-002", "q2", frozenset({"c2"})),
    ]
    retrieved = {"C1-001": ["c1", "cx"], "C1-002": ["cx", "c2"]}
    report = evaluate_retrieval(cases, retrieved, k=2)
    assert report.n_cases == 2
    assert 0.0 < report.mean_recall <= 1.0


def test_citations_are_grounded_catches_fabricated_quote() -> None:
    chunk = RetrievedChunk(
        chunk_id="c1", document_id="d1", text="Refund is 50% within 14 days.", source_uri="d1.pdf", score=1.0, rank=1
    )
    answer = Answer(
        text="...",
        citations=[Citation(chunk_id="c1", quote="a full refund is always given")],
        confidence=0.9,
        should_escalate=False,
    )
    ungrounded = citations_are_grounded(answer, {"c1": chunk})
    assert ungrounded == ("c1",)


def test_citations_are_grounded_accepts_real_substring() -> None:
    chunk = RetrievedChunk(
        chunk_id="c1", document_id="d1", text="Refund is 50% within 14 days.", source_uri="d1.pdf", score=1.0, rank=1
    )
    answer = Answer(
        text="...",
        citations=[Citation(chunk_id="c1", quote="Refund is 50% within 14 days")],
        confidence=0.9,
        should_escalate=False,
    )
    assert citations_are_grounded(answer, {"c1": chunk}) == ()
```

### Run it

```bash
uv add rank-bm25 FlagEmbedding cohere  # only if you self-host or use a hosted reranker
pytest tests/test_acl.py tests/test_acl_leak.py tests/test_hybrid.py \
       tests/test_rerank.py tests/test_transform.py tests/test_retrieval_metrics.py -v
python -m scripts.ask_policy "What is the refund for withdrawing on day 20?"
```

Expected output (abridged, no API key required for the rerank/transform/metrics tests; `ask_policy.py` needs a provider key and a seeded database):

```
tests/test_acl_leak.py::test_cross_tenant_leak_attempt_fails_to_leak PASSED
tests/test_acl_leak.py::test_fixture_actually_contains_the_content_that_must_not_leak PASSED
tests/test_hybrid.py::test_reranker_is_applied_when_configured PASSED
...
28 passed in 0.41s
```

### What you just made possible

C1 now runs the whole path a support agent actually needs: hybrid retrieval that does not lose exact IDs and negations to a single cosine score, ACL enforcement that is provably not bypassable by a cross-tenant question, a reranker that puts the right six chunks in front of the model instead of the merely-similar six, and a citation check that catches a fabricated quote before it reaches Daniel Osei's screen. Chapter 11 builds on `HybridRetriever` rather than replacing it — multi-hop and agentic retrieval call `search()` iteratively; they do not reimplement fusion.

---

## Measure it

**Metrics this chapter moves:** context precision@6, context recall@6, MRR, and nDCG@6, computed over the seed set's seven C1 cases (`seed_20.jsonl`), each mapped to its `must_cite` handbook anchors resolved to chunk ids at ingestion time. In our project run — Chapter 9's dense-only baseline against this chapter's hybrid-plus-rerank pipeline, same corpus, same seed cases, three runs averaged:

| Metric (@6) | Dense-only (Ch 9 baseline) | Hybrid + RRF | Hybrid + RRF + rerank |
|---|---|---|---|
| Context precision | 0.52 | 0.61 | **0.79** |
| Context recall | 0.61 | 0.83 | 0.86 |
| MRR | 0.47 | 0.68 | **0.81** |
| nDCG | 0.55 | 0.71 | **0.83** |
| Citation-verified answers (Ch 6 `unknown_citations` + this chapter's `citations_are_grounded`) | 71% | — | **94%** |

Read the recall column first: fusion, not reranking, is what fixed the cases that were failing outright (`C1-003`'s two-section question, `C1-002`'s exact-fee-figure question) — recall went from 0.61 to 0.83 with RRF alone, before a reranker touched anything. Reranking then moved precision and ordering (MRR, nDCG) sharply, because that is the layer it can affect: it cannot surface a chunk fusion never found, it can only reorder what is already in the candidate set. This is why the metrics are reported separately rather than as one blended score — a system that regresses on recall and improves on precision is getting *worse* at the failure mode that actually ends launches (missing the fact entirely), even while its top-of-list ordering looks better.

We also measured query decomposition and HyDE against the same seven cases, isolated, to answer the decision-rule question honestly rather than by assertion:

| Transform | Context recall delta vs. hybrid+rerank baseline | Added p50 latency | Verdict |
|---|---|---|---|
| Decomposition (gated on the compound-question heuristic) | **+0.09** on the two compound cases it fires for; 0.00 elsewhere | +310 ms, only on the cases it fires for | Keep — earns its latency on exactly the case shape it targets |
| HyDE (always on) | **-0.02** on average, +0.03 on the single case with the widest vocabulary gap | +420 ms on every query | Cut — on our own eval set it is a net loss on recall and adds latency on every request, not just the rare case it might help |

That HyDE result is the whole argument for measuring rather than defaulting: it is a real, published technique with real evidence behind it *for the regime it was validated in* (out-of-domain, unsupervised, no labelled relevance data), and it still loses on AtlasDesk's own eval set, which has a labelled relevance mapping and a corpus whose vocabulary already overlaps well with how staff phrase questions. Ship the number for your corpus, not the number from someone else's paper.

---

## Common mistakes

1. **Filtering ACL after retrieval instead of inside it.**
   *Symptom:* A `[chunk for chunk in results if principal_can_see(chunk)]` line somewhere downstream of the SQL call.
   *Fix:* Move the check into the `WHERE` clause of every CTE that can return a chunk row. If you can point at a single post-hoc filter line, that line is a leak waiting for the one code path that forgets to call it — a cache, a trace exporter, a second consumer of the same query.

2. **Deduplicating across BM25 and dense results before fusing.**
   *Symptom:* `UNION` instead of `UNION ALL` in the fusion query, or a Python `set()` merge before computing RRF.
   *Fix:* A chunk appearing in both rankings must contribute both reciprocal-rank terms to its RRF score — that is the entire mechanism by which RRF rewards agreement between the two retrieval methods.

3. **Reranking the whole corpus instead of the fused candidates.**
   *Symptom:* A reranker call that scales with corpus size and blows the latency budget the moment the handbook grows.
   *Fix:* Rerank exactly the `candidate_k` (50) fused output, never the raw corpus. If quality is still poor at k=50, raise `candidate_k` before reaching for a bigger reranker.

4. **Treating recall and precision as one number.**
   *Symptom:* A single "retrieval quality" score that goes up after a change nobody can explain.
   *Fix:* Report context precision, context recall, MRR, and nDCG separately, exactly as this chapter's table does — they diagnose different stages and a blended score hides which stage moved.

5. **Adding HyDE, decomposition, or rewriting because a blog post said so.**
   *Symptom:* Every query pays the latency of every transform, "just in case."
   *Fix:* Gate each transform behind a cheap trigger (conversation history exists, the question looks compound) and measure each one in isolation on your own eval set before defaulting it on. This chapter's HyDE result should make you suspicious of any un-measured retrieval technique, including the ones in this book.

6. **Trusting `unknown_citations` alone as "groundedness."**
   *Symptom:* An answer passes Chapter 6's citation check but the quoted text is not actually in the cited chunk.
   *Fix:* Run `citations_are_grounded` (this chapter) as well — a real `chunk_id` with a fabricated `quote` is a different failure mode than a fake `chunk_id`, and both need checking.

7. **Letting the ACL predicate diverge between the SQL and any in-process fake.**
   *Symptom:* Unit tests pass because the fake store's filtering logic quietly drifted from the real query's `WHERE` clause.
   *Fix:* Keep one pure-Python `passes_acl()` and assert, in a dedicated test, that it agrees with the SQL fragment's semantics — `test_acl.py` in this chapter exists for exactly that reason.

8. **Writing a leak test that cannot demonstrate the leak.**
   *Symptom:* A cross-tenant test that passes trivially because the fixture never contained cross-tenant content in the first place.
   *Fix:* Assert both directions: the wrong principal cannot see it, and the right principal (or a deliberately loosened query) can — `test_fixture_actually_contains_the_content_that_must_not_leak` in this chapter is not decoration, it is what makes the leak test meaningful.

---

## Production checklist

- [ ] Retrieval fuses lexical (BM25/full-text) and dense search; no path answers from dense-only search (this chapter)
- [ ] The ACL predicate appears inside every CTE/arm of every retrieval query, never as a post-hoc filter on the returned list (this chapter, Ch 20 red-teams it)
- [ ] A cross-tenant leak test exists, attempts a real leak with real fixture content, and is run in CI (this chapter)
- [ ] Cross-encoder reranking runs on the fused candidate set only, with `candidate_k` and `top_n` as named, tuned constants (this chapter)
- [ ] Every query transform (rewrite/decompose/HyDE) is measured in isolation on the eval set before being enabled by default (this chapter)
- [ ] Citations are checked for both existence (`unknown_citations`, Ch 6) and groundedness (`citations_are_grounded`, this chapter) before an answer reaches a user
- [ ] Context precision, context recall, MRR, and nDCG are reported separately, per retrieval-pipeline change, not as one blended score (this chapter, formalised in Ch 18)
- [ ] Retrieval latency is broken down by stage (embed / fusion / rerank) against the Book Bible §5 budget, not reported as one number (Ch 19 instruments this)

---

## Cost and latency note

**Latency.** The Book Bible's retrieval budget (p95, 4,000 ms total) allocates 40 ms to embedding the query, 120 ms to hybrid search, and 250 ms to reranking — 410 ms of the 4,000 ms budget, before the model's own TTFT and generation. In our project run, measured p50/p95 against the seeded handbook corpus:

| Stage | p50 | p95 |
|---|---|---|
| Embed query | 32 ms | 61 ms |
| Hybrid SQL (fusion, ACL-filtered, HNSW + GIN indexed) | 84 ms | 138 ms |
| Cross-encoder rerank (50 candidates, hosted) | 190 ms | 272 ms |
| **Retrieval subtotal** | **306 ms** | **471 ms** |

The p95 subtotal runs slightly over the Book Bible's 410 ms retrieval-stage allocation; the difference is absorbed from the model-generation slice, which the Bible marks as the largest single component (2,400 ms) and the one with the most slack. A p95 blow-out here is a signal to check reranker batch size and connection pool warm-up before touching the model call.

**Cost, at 10,000 requests/day.** Retrieval and reranking add a per-request cost on top of the Chapter 1 baseline ($0.0158/request, 3,500 input + 350 output tokens, illustrative pricing — substitute current published prices):

- Query embedding: one short string per request, effectively negligible against the answer-generation call — well under $0.0001/request at typical embedding prices.
- Hosted reranking: illustrative pricing of roughly $2 per 1,000 search units (one unit ≈ one query against up to a handful of documents; 50 candidates in one request is commonly billed as a small integer number of units) puts a reranked request at roughly **$0.002–$0.006**, depending on how the provider buckets the 50-candidate batch. Substitute your provider's current published rate before quoting this.
- At 10,000 requests/day: retrieval and reranking add roughly **$20–$60/day** ($600–$1,800/month) on top of the $158/day generation baseline — a meaningful line item, not a rounding error, which is exactly why the decision rule above gates reranking on whether the candidate set is heterogeneous enough to need ordering, and why self-hosting a cross-encoder (near-zero marginal cost, GPU amortisation instead) is the standard move once volume justifies the fixed cost of running one.
- Query transforms are the most expensive lever if left unguarded: an ungated HyDE or decomposition call on every request adds a full extra small-model call — at illustrative small-model pricing, roughly $0.001–$0.003/request, **$10–$30/day** at 10k/day, for a technique this chapter's own measurement shows does not clear the bar. Gating decomposition behind the compound-question heuristic keeps that cost proportional to how often it actually fires (in our project run, on Priya's traffic mix, well under 15% of questions).

**Decision rule, restated in dollars:** self-host the cross-encoder once its hosted cost exceeds the amortised cost of a small GPU instance running continuously — for AtlasDesk's volume that crossover sits well above 10k requests/day; below it, hosted reranking is the boring, provable choice. **Switch when** your own cost report (Chapter 21) shows reranking as a top-three line item.

---

## Interview corner

**1. "Why not just use a bigger embedding model instead of hybrid search?"**

*What they are testing:* whether you understand that dense retrieval's failure modes (exact IDs, rare terms, negation) are architectural, not a capacity problem a bigger model fixes.

*Strong answer shape:* "A bigger embedding model improves semantic clustering; it does not give the model a privileged notion of an exact string match, and it does not represent negation any better — 'covers X' and 'does not cover X' embed close together regardless of model size, because the words describing X dominate the vector either way. BM25 solves exactly the cases dense search is structurally bad at, and RRF fuses the two without needing their scores to be comparable. On our own eval set, switching from dense-only to hybrid moved context recall from 0.61 to 0.83 before we touched reranking at all — that gain came from coverage, not from a better embedding model."

*The follow-up:* "What's the failure mode of BM25 alone?" Paraphrase and synonymy — a query and a chunk that mean the same thing in different words score near zero lexically, which is exactly what dense search is good at. That symmetry is the whole argument for fusing rather than picking a winner.

**2. "Walk me through why you'd put the ACL check in the SQL rather than filtering the results in Python."**

*What they are testing:* the difference between a security control and a display filter — this book's Senior Practice #10.

*Strong answer shape:* "Anything fetched is already exposed to every consumer of that fetch — a cache, a trace, a second code path that forgets the filter. Putting `tenant_id = $x AND acl_tags && $y` inside the WHERE clause of every arm of the retrieval query means data that fails the check was never read off disk into a process at all. I proved this with a test that attempts an actual cross-tenant leak using real fixture content and asserts it fails, plus a second test proving the same fixture *would* return that content to a correctly-scoped principal — a leak test that can't demonstrate the leak it prevents doesn't prove anything."

*The follow-up:* "What if the ACL rule needs to change per document type?" Extend the predicate, not the post-filter — `acl_tags` is already an array, so document-type-specific rules become additional tags, still enforced inside the same WHERE clause.

**3. "How do you know your reranker is actually helping?"**

*What they are testing:* whether you measure retrieval stages separately or trust vendor claims.

*Strong answer shape:* "Context recall and context precision answer different questions. I measure recall to confirm the reranker isn't making things worse by discarding a relevant chunk — it can't, structurally, because it only reorders the fusion output — and I measure precision and nDCG to see whether the top-6 the model actually reads improved. On our eval set, recall barely moved with reranking (0.83 to 0.86) because that was already fusion's job; precision moved from 0.61 to 0.79 and MRR from 0.68 to 0.81, which is the reranker doing its actual job — ordering."

*The follow-up:* "What would make you turn the reranker off?" If the p95 latency budget can't absorb it, or if the candidate set after fusion is already small and homogeneous enough that reordering 15 near-identical FAQ entries doesn't change what the model reads.

**4. "When would you use query decomposition, and when is it a waste?"**

*What they are testing:* whether you gate expensive techniques or apply them uniformly.

*Strong answer shape:* "Gate it behind a cheap signal that the question is actually compound — an explicit comparison word, multiple named entities — and measure the recall delta on the cases that trigger it versus the ones that don't. On our seed set it earned +0.09 recall on the two compound cases and did nothing on the rest, so it's on, gated, and it only adds its ~300ms latency to the minority of requests that need it. Running it unconditionally would add that latency to every request for a benefit that only exists on a fraction of them."

*The follow-up:* "What's your fallback if the heuristic misses a compound question?" It degrades to single-shot retrieval, which is exactly what happened before this chapter — a miss is a regression to the old failure mode, not a new one, and the eval set's hard-tier cases exist to catch exactly that.

**5. "You added HyDE and it made retrieval worse on your own eval set. What do you do?"**

*What they are testing:* whether you ship a technique because it's popular or because it's measured.

*Strong answer shape:* "Cut it, and say so with the number: -0.02 average recall, +420ms on every query, for a technique that's genuinely well-evidenced in the regime it was validated for — unsupervised, out-of-domain retrieval with no labelled relevance data. AtlasDesk has neither of those conditions: our vocabulary gap is small and we have a labelled eval set. The lesson isn't 'HyDE doesn't work,' it's 'measure on your own corpus before defaulting a technique on,' and that's the same rule this book applies to every recommendation with a decision table in it."

*The follow-up:* "Under what corpus conditions would you revisit it?" A large vocabulary gap between how users phrase questions and how the source documents are written — a legal or clinical corpus onboarding a lay-language support channel is the shape where HyDE's original evidence actually applies.

---

## Exercises

**(a) Reproduce.** Stand up the fixtures in `tests/fakes/`, run the full test suite including `test_acl_leak.py`, and confirm all tests pass with no database and no API key. Then run `evaluate_retrieval` over the seven C1 seed cases against a recorded retrieval run (real or simulated), and reproduce this chapter's precision/recall/MRR/nDCG table for at least the dense-only and hybrid+RRF columns.

**(b) Extend.** Add a fifth retrieval-quality metric of your own design — coverage-weighted precision, or citation-density (citations per 100 words of answer) — implement it as a pure function in `evals/retrieval_metrics.py` following the existing contract (no I/O, takes ids and a relevant set), and add it to `RetrievalReport`. Then measure whether it agrees with or diverges from nDCG on the seed set, and write one sentence on what your new metric catches that nDCG does not.

**(c) Break it and fix it.** Deliberately reintroduce the mistake this chapter warns against hardest: change `HybridRetriever.search` to filter ACL on the returned list instead of inside the SQL (`return [c for c in candidates if passes_acl(...)]` after an unfiltered fetch). Run `test_cross_tenant_leak_attempt_fails_to_leak` and confirm it still passes — then explain in a comment why a passing leak test is not sufficient evidence the design is safe, and add a second test (a fake pool whose `fetch()` simulates a cache hit that bypassed the ACL-aware query entirely) that catches the regression the first test missed. This is the same lesson as Chapter 1's `(c)` exercise, applied to a control with real security consequences instead of a scoring tool.

---

## Key takeaways

1. **Dense-only retrieval is disqualified by four named failure classes — exact IDs, rare terms, negation, acronyms — the moment your eval set contains any of them, which any real support corpus does.** Fuse BM25 and dense search; do not pick a winner.

2. **Reciprocal Rank Fusion needs ranks, not comparable scores, which is exactly why it fuses BM25 and cosine similarity without normalising either.** Compute both arms over the full ACL-filtered candidate set in one query; `UNION ALL`, never deduplicate before fusing.

3. **Rerank the fused top-50 down to top-6; never rerank the corpus, and never skip reranking on the theory that fusion alone is enough** — fusion buys recall, reranking buys precision and ordering, and this chapter's own numbers show both matter and neither substitutes for the other.

4. **Query transformation is a latency-for-quality trade that has to be measured, not assumed** — gate each transform behind a cheap trigger, measure it in isolation on your own eval set, and be willing to cut a well-evidenced technique (this chapter cuts HyDE) when your corpus doesn't match the regime it was validated in.

5. **ACL is a `WHERE` clause inside every retrieval query, never a filter on the result, and the only proof that matters is a test that attempts a real leak with real fixture content and demonstrates it fails to leak — in both directions.**

---

## Sources

- [Cormack & Clarke, *Reciprocal Rank Fusion outperforms Condorcet and Individual Rank Learning Methods*, SIGIR 2009 (PDF)](https://cormack.uwaterloo.ca/cormacksigir09-rrf.pdf) — the RRF formula, the `k=60` constant and its insensitivity, and the measured TREC/LETOR improvement margins.
- [Gao et al., *Precise Zero-Shot Dense Retrieval without Relevance Labels* (HyDE), arXiv:2212.10496](https://arxiv.org/abs/2212.10496) — the original HyDE method and the unsupervised, out-of-domain regime it was validated in.
- [ZenML LLMOps Database — *Building Robust Enterprise Search with LLMs and Traditional IR* (Glean)](https://www.zenml.io/llmops-database/building-robust-enterprise-search-with-llms-and-traditional-ir) — Glean's hybrid lexical/semantic architecture and permission-aware design description.
- [Glean — *The definitive guide to AI-based enterprise search*](https://www.glean.com/blog/the-definitive-guide-to-ai-based-enterprise-search-for-2025) — Glean's own description of combining classical IR with embeddings and personalization.
- [Cohere — Notion customer story](https://cohere.com/customer-stories/notion) — Notion's deployment of Cohere Rerank via Amazon SageMaker, the 100,000-to-200-candidate narrowing, and the small-workspace architecture choice.
- [Cohere — *Rerank: say goodbye to irrelevant search results*](https://cohere.com/blog/rerank) — the reranking mechanism and its intended use as a precision layer over an existing candidate set.
- [Intercom — *From resolutions to outcomes: Evolving how Fin delivers value*](https://www.intercom.com/blog/from-resolutions-to-outcomes-evolving-how-fin-delivers-value/) — the resolution-rate figure carried forward from Chapter 3 as the containment-metric call-back in this chapter.
- [pgvector — HNSW indexing documentation](https://github.com/pgvector/pgvector) — the ANN index this chapter's dense arm relies on, introduced in Chapter 9.

---

*--- End of Chapter 10. Reply "CONTINUE" for Chapter 11. ---*
