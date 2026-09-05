# ADR-005: Retrieval = SQLite FTS5 recall + in-process cosine re-rank

- Status: accepted
- Date: 2026-07-07

## Context

Ask needs retrieval over ~10k–60k chunks for repos up to ~300k LOC. Verified 2026-07: `sqlite-vec`
has platform problems (Windows DLL vs better-sqlite3 12.x SQLite; `node:sqlite` compiled with
`OMIT_LOAD_EXTENSION`) and a slow release cadence. A vector-DB dependency (Qdrant etc.) would
break the zero-infra, local-first posture.

## Decision

1. Storage is one SQLite file (`.archmap/index.db`) via `better-sqlite3`, which bundles FTS5 —
   no loadable extensions at all in v1.
2. Retrieval is two-stage: FTS5 keyword recall (top 500 candidates; identifier-split aux column so
   `getUserById` matches "get user by id") → cosine re-rank over just those candidates using
   embeddings stored as float32 BLOBs, computed in-process (500 × ≤1024 dims ≈ sub-millisecond).
3. Embeddings are optional: if `embeddings.enabled: false` or the endpoint is unavailable, Ask
   runs in documented `fts-only` mode — degraded relevance, zero broken functionality.
4. `sqlite-vec` (or successors) may be added in v2 as an optional accelerator behind the same
   retrieval interface (`retrieve(q, opts): RetrievalResult`).

## Consequences

- Zero native-extension risk across platforms; one dependency (`better-sqlite3`) already needed
  for FTS anyway; index is a single deletable file.
- Recall is bounded by keyword stage: purely semantic queries with no lexical overlap can miss.
  Accepted for v1; measured by the ask eval harness, and the interface allows swapping stage 1.
- No ANN index needed at this scale; brute-force-over-candidates keeps code trivial.

## Alternatives rejected

- `sqlite-vec` as required dependency: verified platform breakage contradicts "npx and go".
- External vector DB: violates zero-infra local-first posture.
- Full brute-force over all chunks in JS: ~180 MB RAM at 60k×768 floats; wasteful when FTS
  prefilter exists.
