# 28 — Symbol-aligned chunker

## Summary

Implement Ask's chunker per DESIGN §11: code chunks aligned to top-level symbols (with split
rules), wiki chunks per H2 section, deterministic ids, and secret-filter compliance.

## Context

Chunks are the retrieval unit for FTS (29), embeddings (30), and citations in answers (32).
Deterministic ids keep the index incrementally updatable via file-hash diffs.

## Scope

- In: `src/ask/chunk.ts`, Chunk zod schema (DESIGN §7.7).
- Out: indexing (29), embedding (30).

## Detailed Requirements

1. Eligibility matrix (locked): `binary` → skip; `secret-excluded` → skip (zero chunks);
   `too-large` → whole-file windowing via `readAnalyzable` content; `generated` → chunked
   normally (ranking keeps it low); everything else → symbol-aligned chunking.
2. Code chunks — TOP-LEVEL symbols only (kinds function/class/const/interface/type/enum;
   `method` symbols are NOT chunked separately — the class chunk covers them, per DESIGN §11):
   text = up to 3 leading comment lines + the symbol's span lines. Chunk `span` = the EXACT
   text range emitted (i.e. startLine = first included comment line when present); the id
   derives from that span. Files with facts but no top-level symbols (fallback languages,
   scripts) and `too-large` files → whole-file windows of 120 lines with 10-line overlap,
   `symbol: null`.
3. Split rule: symbol chunks over 120 lines split into 120-line windows with 10-line overlap;
   each window is its own chunk whose `span`/id reflect the window's real line range (distinct
   spans → distinct ids; no suffix scheme needed); every window keeps the symbol name.
   Example: symbol at lines 8–300 with 2 comment lines → chunks `c:src/a.ts:6-125`,
   `c:src/a.ts:116-235`, `c:src/a.ts:226-300`.
4. Wiki chunks: for each generated page, one chunk per H2 section (heading + body, prose-slot
   markers stripped, mermaid fences dropped); `kind: "wiki"`, path = wiki file repo-relative
   path, span = section line range, `symbol` = heading text.
5. Ids: `c:<path>:<startLine>-<endLine>` — regenerated identically for unchanged content;
   `hash` = sha256 of the STORED chunk text (i.e. post-truncation when rule 6 applies).
6. Size cap: chunk text hard-capped at 8 KB; over-cap text truncated at the cap with the exact
   marker `\n…[archmap:truncated]` appended; each truncation adds warning token
   `chunk-truncated` to the result (no logger side effects — the function stays pure).
7. API: `chunkRepo({inventory, factsByPath, renderedWiki, readAnalyzable}):
   Promise<{chunks: Chunk[], warnings: string[]}>` — `readAnalyzable` is issue 08's async
   accessor; output sorted by id; deterministic.

## Acceptance Criteria

- [ ] Golden chunk list for `fixture-ts` (stable across runs); symbol alignment verified (chunk
      text contains the symbol's first line); a class with methods yields ONE class chunk and
      no method chunks; comment-prefixed symbol's span/id start at the first comment line
      (exact-id assertion).
- [ ] 300-line function splits into overlapping windows matching the documented example ids.
- [ ] Eligibility matrix test: binary/secret-excluded skipped; too-large windowed; generated
      chunked.
- [ ] Wiki page with 4 H2s yields 4 wiki chunks; mermaid + slot markers absent from text.
- [ ] `fixture-secrets` under `exclude` yields no chunks from canary files (asserted by id scan);
      under `redact`, chunk text contains `[REDACTED:` and no canary values.
- [ ] Over-8KB chunk truncated with the exact marker, `chunk-truncated` warning returned, hash
      computed over stored text (test recomputes).
- [ ] Unchanged file → identical ids and hashes across runs (incremental precondition).

## Validation

```bash
npm run test -- ask/chunk
```

## Dependencies

08, 10, 11, 12, 19.

## Non-goals

Embedding/scoring, semantic chunking (v2), non-analyzable binary handling beyond skip.

## Design References

- DESIGN §11 (chunk rules), §7.7 (schema), §13.3 (B2 compliance)
