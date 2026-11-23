# Collection Retrieval

Retrieval is how an indexed collection turns a natural-language query into the few file chunks
(`top_k`) a chatbot gives its LLM as context. At query time, the system scores each chunk against
the query and returns the best matches.

Dense retrieval (called dense search in the code and in the rest of this page) always runs. It
converts the query and each chunk to embedding vectors and returns the chunks nearest the query.

Three optional stages build on it:

- **[Contextual retrieval](#indexing-contextual-retrieval)** adds a short context header to each
  chunk before it is embedded, at indexing time.
- **[Hybrid search](#hybrid-search-dense-and-lexical-fused)** adds a keyword (lexical) search and fuses
  its ranking with the dense one, at query time.
- **[Reranking](#reranking)** rescores the candidates against the query, at query time.

This page covers local indexes only (`Collection.is_remote_index=False`). Remote indexes
delegate storage and search to the provider's vector store, so none of the stages below apply to
them; see [Index Managers](index_managers.md).

## Overview

### The pipeline

```text
Indexing:  chunk ─► [add context header] ─► embed ─► store

Query:     query ─┬─► dense search (pgvector) ────────────────┐
                  │                                           ├─► [RRF fusion] ─► [rerank] ─► top_k chunks
                  └─► [lexical search (Postgres full-text)] ──┘
```

Brackets mark optional stages, which are behind a feature flag. Terms used above:

- **RRF (Reciprocal Rank Fusion)** merges the dense and lexical rankings using each chunk's rank
  position in each, not its score.
- **pgvector** is the PostgreSQL extension that stores embeddings and runs the vector search.
- **Postgres full-text search** is PostgreSQL's built-in keyword search, used for lexical search.

| Stage | When | Feature flag | Where |
| --- | --- | --- | --- |
| Dense search | Indexing (embed) and query (search) | None, always on | `apps/documents/retrieval.py` |
| Contextual retrieval | Indexing | `flag_contextual_retrieval` | `apps/service_providers/llm_service/contextualizer.py` |
| Hybrid search | Query | `flag_hybrid_search` | `apps/documents/retrieval.py` |
| Reranking | Query | `flag_hybrid_search` plus a reranker provider | `apps/documents/rerankers.py` |

Both flags are off by default, so with neither enabled only dense search runs.

The flags are Waffle feature flags (see [Feature Flags](../feature_flags.md)). Each can be enabled
per team, by percentage, or globally, and rolled back without a deploy.

### Entry point

`apps.documents.retrieval.search_collection` is the single entry point for retrieval. It is called
by the chat search tools (`SearchIndexTool`, `SearchCollectionByIdTool` in
`apps/chat/agent/tools.py`) and by the collection query preview in `apps/documents/views.py`.

## Indexing: contextual retrieval

A chunk embedded in isolation carries no signal about which document it came from — "revenue grew
3% over the previous quarter" does not say whose revenue or which quarter. When contextual
retrieval is on, `LocalIndexManager._embed_file` asks a chat model for a short header situating
each chunk in its document, stores it on `FileChunkEmbedding.context`, and embeds
`context + "\n\n" + text` rather than the chunk alone. `FileChunkEmbedding.text` is never
mutated: it stays the source of truth, and `contextualized_text` composes the two.

The document is placed in the system prompt so provider prompt caching can reuse it across every
chunk of the same file. A failed contextualizer call logs and returns an empty header, so a file
still indexes.

## Hybrid search: dense and lexical, fused

Dense search is the original behaviour: cosine distance between the query embedding and each
chunk's `embedding`, over the HNSW index.

Lexical search matches `FileChunkEmbedding.search_vector`, a `tsvector` built from `context` and
`text` at indexing time and covered by a GIN index. It is built with the collection's own
`search_language`, and queries are parsed with the same one — a chunk indexed as `spanish` and
queried as `english` matches nothing at all, which is indistinguishable from having no lexical
hits. **Changing `search_language` therefore requires re-indexing the collection.**

The two rankings are combined with weighted Reciprocal Rank Fusion:

```text
score(chunk) = w / (k + rank_dense) + (1 - w) / (k + rank_lexical)
```

`k` is `settings.DOCUMENT_SEARCH_RRF_K` (60) and `w` is `Collection.search_dense_weight` (0.7).
Ranks are fused rather than scores because cosine distances and `ts_rank_cd` values live on
incomparable scales, so score-level fusion would need per-query normalization. A chunk found by
only one of the two simply gets no contribution from the other; no imputation is involved.

`Collection.search_fetch_k` (40) is how many candidates each side contributes to the fusion.

## Reranking

Neither dense nor lexical search ever looks at a query and a chunk together: an embedding is
computed before the query exists, and `ts_rank_cd` only counts term overlap. A reranker scores
the pair directly, which is why it can reorder candidates that fusion ranked purely on how each
half happened to retrieve them.

When reranking is active, `search_collection` widens its candidate pool to
`Collection.rerank_top_n` (50), scores each `(query, contextualized_text)` pair, and returns the
best `top_k`. Reranked chunks carry a `rerank_score`.

Only a hosted reranker is implemented, backed by Voyage AI. Voyage rather than another provider
because OCS already depends on the `voyageai` client (used for Voyage embeddings through
`langchain-voyageai`) and already models Voyage credentials as an `LlmProvider`, so reranking adds
no third-party dependency and no new provider type. A local cross-encoder and Cohere are both
left as follow-ups, since each would add one.

The stage can only improve the ranking. A reranker that errors, times out, or answers with
something that does not describe the candidate list it was sent leaves the search with the
ranking it already had, logs, and returns that. `Collection.get_reranker()` likewise returns
`None` for every reason not to rerank rather than raising.

`Collection.reranking_enabled` answers whether the stage will run at all. The hybrid-search flag
must be active and both `reranker_provider` and `rerank_model` must be present. It answers from the
loaded collection instance without fetching the provider row.

### Context conditioning

`search_collection` accepts an optional `context` — the recent conversation turns, which
LangGraph injects into the chat search tools as part of the graph state. It is prepended to the
reranker's query so the reranker can tell which of several similar chunks answers the question
actually being asked: "how much does it cost" scores differently once the turn before it is
visible. Only the reranker reads it, so the tools skip formatting it unless reranking is active
for the collection.

The context is clipped to its tail (`MAX_RERANK_CONTEXT_CHARS`) because the recent turns are the
ones that disambiguate, and because the provider clips the query-document pair to the model's
context window — an unbounded context would push the query itself out of that window.

## Enabling reranking for a collection

1. Enable `flag_hybrid_search` for the team.
2. Create a Voyage AI LLM provider for the team, if it does not already have one.
3. On the collection, set `reranker_provider` to that provider. `rerank_model` defaults to
   `rerank-2` and `rerank_top_n` to 50.

All three are required. Leaving the provider unset leaves reranking off, silently by design;
the failures that do log are the ones an operator cannot predict, such as a provider with no
rerank endpoint, credentials the provider rejects, or a provider belonging to another team.

That last one is a runtime refusal rather than a form constraint: `reranker_provider` is
editable in the Django admin, which offers every team's providers, so `get_reranker()` checks
the provider's team itself. Reranking with another team's credentials would bill them and send
this collection's queries to their account.

The tuning fields (`search_language`, `search_dense_weight`, `search_fetch_k`, `reranker_provider`,
`rerank_model`, `rerank_top_n`) are model fields with no form or pipeline
node UI, to keep the node's configuration surface small while the defaults are being proven. The
node's existing `max_results` is what controls the final `top_k`.

Because nothing calls `full_clean()` on these fields, their validators never fire and the
database check constraints on `Collection` are the only thing that actually stops a bad value.

## Cost and latency

Contextualization is paid once per chunk at indexing time, never at query time. Hybrid search
adds one GIN-index query and an in-memory fusion over at most `2 * search_fetch_k` rows.
Reranking is the only stage with a per-query external cost: one provider call scoring
`rerank_top_n` candidates, which is what bounds it.
