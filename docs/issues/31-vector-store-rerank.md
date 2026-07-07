# 31 — Embedding store and hybrid retrieval

## Summary

Implement retrieval per ADR-005/DESIGN §11: FTS5 candidates → cosine re-rank with stored
embeddings → combined scoring with exact-symbol bonus → `RetrievalResult`, with automatic
`fts-only` degradation.

## Context

This is the single retrieval API used by `ask` (32) and MCP `ask_question` (35). Quality knobs
(candidates, topK, weights) are fixed constants/config per DESIGN.

## Scope

- In: `src/ask/retrieve.ts`.
- Out: answer synthesis (32), eval (33).

## Detailed Requirements

1. `retrieve(db, embedder: Embedder | null, query: string, opts: RetrieveOptions, logger):
   Promise<RetrievalResult>` where `RetrieveOptions = {topK?: number (default 12, bounds
   1..50), ftsCandidates?: number (default 500)}` — defaults resolved by the CALLER from
   `config.ask` (this module never imports config).
   - Stage 1: `queryFts` (29) top `ftsCandidates`.
   - Stage 2 (when embedder non-null AND `embedQuery` returns a vector): load candidate vectors
     in one query, restricted to rows whose `model == meta.embeddingModel`; validate BLOB
     length == dims·4 (invalid rows skipped + debug log); query-vs-stored dims mismatch →
     whole stage skipped, fts-only with warn `embeddings-dims-mismatch`. Cosine in-process
     (LE float32 loop, no deps); candidates lacking vectors get cosine 0 (count emitted as a
     debug log event — RetrievalResult stays per DESIGN §7.7).
   - Scoring: `finalScore = clamp(0.5·normFts + 0.5·max(cosine, 0) + bonus, 0, 1)` where
     `bonus = 0.15` when a query token equals `chunks.symbol` case-insensitively
     (post-identSplit tokens); norm = min-max within candidate set.
   - fts-only mode (no embedder/vector/stage-2 skip): `finalScore = clamp(normFts + bonus, 0,
     1)`; `mode: "fts-only"`.
   - Base result: top `topK` by (finalScore desc, chunkId ASC).
2. Wiki pull-up rule (exact algorithm): let W = wiki chunks in the base topK. If `|W| < 2` and
   wiki chunks exist within the top `3·topK` overall ranking, promote the highest-scored
   not-yet-included wiki chunks until the result has `min(2, available)` wiki chunks, each
   promotion evicting the lowest-scored non-wiki chunk. Final list is re-sorted by
   (finalScore desc, chunkId ASC). Fixture asserts the exact expected ordered ids.
3. Each result entry: `{id, path, span, symbol, kind, ftsScore, cosine|null, finalScore, text}`;
   `mode` field per §7.7.
4. Performance: ≤ 1.5 s excluding embedding call at 60k chunks (candidate-bounded math; DESIGN
   §15). This issue ALSO adds the retrieval row to `scripts/bench.ts` (in scope; replaces 27's
   `skipped (issue 31)` row).
5. Empty results: valid `RetrievalResult` with empty chunks (32 turns it into the honest refusal).

## Acceptance Criteria

- [ ] Hybrid ordering test: seeded db where pure-FTS order differs from hybrid order; cosine
      re-rank flips ranking as expected (hand-computed vectors).
- [ ] Symbol bonus: query `retrieve` ranks the `retrieve` symbol chunk first among near-ties;
      clamp proven at 1.0.
- [ ] Degradation matrix: null embedder / no vectors / embedQuery null / dims mismatch (warn
      token) / wrong-model rows excluded / invalid BLOB skipped → fts-only behavior per rule;
      partial vectors handled (cosine 0 for missing, debug event emitted).
- [ ] Pull-up rule: fixture where wiki chunks sit at ranks 15–18 → exactly the documented
      expected ordered id list results (promotion + eviction + re-sort proven).
- [ ] `topK` bounds enforced (0 and 51 rejected as E_USAGE by option validation).
- [ ] 60k-chunk synthetic index retrieval ≤ 1.5 s via `npm run bench` retrieval row
      (informational).

## Validation

```bash
npm run test -- ask/retrieve
npm run bench   # retrieval row now measured
```

## Dependencies

29, 30.

## Non-goals

Answer generation (32), reranker models (v2), user-tunable weights (constants in v1).

## Design References

- DESIGN §11 (retrieval), §7.7 (RetrievalResult), §15; ADR-005
