# 29 — SQLite FTS5 chunk index

## Summary

Implement the keyword index per ADR-005: `.archmap/index.db` schema (chunks + FTS5 +
identifier-split aux column + meta), incremental upsert/delete by file-hash diff, and the query
API returning scored candidates.

## Context

FTS5 is retrieval stage 1 and the always-available fallback (`fts-only` mode). It also powers MCP
`search_wiki` (34) with no LLM required.

## Scope

- In: `src/ask/store.ts` (db open/migrate/schema), `src/ask/index.ts` (build/update/query FTS),
  the real `src/ask/index-hook.ts` implementation (replacing issue 25's no-op — pipeline
  integration is IN scope here, incl. its e2e test).
- Out: embeddings (30), re-ranking (31).

## Detailed Requirements

1. `store.ts`: open `.archmap/index.db` via `better-sqlite3`; `PRAGMA journal_mode=WAL,
   synchronous=NORMAL, foreign_keys=ON`; schema v1 — exact DDL (locked, external-content FTS):
   ```sql
   CREATE TABLE chunks(rid INTEGER PRIMARY KEY, id TEXT NOT NULL UNIQUE, kind TEXT NOT NULL,
     path TEXT NOT NULL, start_line INT, end_line INT, symbol TEXT, hash TEXT NOT NULL,
     text TEXT NOT NULL, ident_text TEXT NOT NULL);
   CREATE INDEX chunks_path ON chunks(path);
   CREATE VIRTUAL TABLE chunks_fts USING fts5(text, ident_text,
     content='chunks', content_rowid='rid', tokenize='unicode61 remove_diacritics 2');
   CREATE TABLE embeddings(chunk_rid INTEGER PRIMARY KEY REFERENCES chunks(rid)
     ON DELETE CASCADE, model TEXT NOT NULL, dims INT NOT NULL, vector BLOB NOT NULL);
   CREATE TABLE meta(key TEXT PRIMARY KEY, value TEXT NOT NULL);
   ```
   External-content maintenance is explicit: after chunk INSERT/UPDATE/DELETE, issue the
   corresponding `INSERT INTO chunks_fts(rowid, …)` / `INSERT INTO chunks_fts(chunks_fts,
   rowid, …) VALUES('delete', …)` statements (document each in code). `queryFts` joins
   `chunks_fts` rowid → `chunks.rid` to return `chunks.id`. `meta` stores `schemaVersion`,
   `embeddingModel`, `builtAt`, `toolVersion`. `migrate()`: schemaVersion mismatch in v1 →
   drop + rebuild (no migrations yet).
2. Identifier splitting: `identSplit(text)` — camelCase, PascalCase, snake_case, kebab-case,
   digit boundaries → space-joined lowercase (`getUserById` → `get user by id`); stored in
   `chunks.ident_text` at insert.
3. `index.ts`:
   - `updateIndex(db, fullChunkSet: Chunk[]): {inserted, updated, deleted}` — the argument is
     ALWAYS the complete post-generate chunk set: new id → insert; same id + changed hash →
     replace as DELETE row + INSERT anew (fresh `rid`, so `embeddings` CASCADE-clears stale
     vectors — issue 30 relies on this); any existing id absent from `fullChunkSet` → delete.
     One transaction.
   - `queryFts(db, query, limit): {chunkId, ftsScore}[]` — prepared statements ONLY (no string
     interpolation into SQL); MATCH expression built exclusively from double-quoted terms
     (each whitespace token quoted with embedded quotes doubled — `OR`/`NOT`/`NEAR` become
     literal terms) OR'd across `text`/`ident_text`, plus the whole phrase quoted; query length
     cap 1000 chars; score = `-bm25(chunks_fts)` min-max normalized within the result set;
     limit default `ask.ftsCandidates`.
   - Empty/short query (< 2 chars post-normalization) → typed error `E_USAGE`.
4. Build integration: implement `buildAskIndex` (25's hook contract) — chunk via 28, then
   `updateIndex`; stamp `meta.toolVersion`; return AskIndexStats `{chunks, inserted, updated,
   deleted, warnings}`; runs AFTER the wiki swap; operational failure → hook returns the error
   for 25's `ask-index-failed` degradation (security/config errors propagate per 25).
5. Corruption handling is mode-split: during `generate` (writer), a corrupted db is deleted and
   rebuilt from the current full chunk set with warning `index-rebuilt` (pure cache, ADR-002);
   read-only consumers (`ask` 32, MCP 34) NEVER delete — they surface exit 4 /
   structured `index-corrupt` with a regenerate hint.
6. Determinism: not byte-level (SQLite internals), but query results deterministic given equal
   content (ORDER BY score DESC, chunkId ASC tie-break).

## Acceptance Criteria

- [ ] `getUserById` chunk found by query "get user"; exact identifier query also hits;
      snake_case fixture equivalent.
- [ ] Incremental: re-index after single-file change touches only that file's rows (row count
      audit); deleted file's chunks gone including FTS rows (orphan check via count join
      between `chunks` and `chunks_fts`).
- [ ] Query hardening matrix: `foo OR bar`, `NOT x`, `NEAR(a b)`, quotes, semicolons,
      `"; DROP TABLE chunks;--` — all execute as literal-term searches via prepared statements
      (no error, no schema change); 1001-char query → E_USAGE.
- [ ] Corruption modes: writer path rebuilds with `index-rebuilt`; a read-only open of the same
      corrupt file surfaces `index-corrupt` without deleting it (file still present).
- [ ] Pipeline e2e: `generate` with `ask.enabled` builds the index after swap (hook stats in
      run report); with 30/31 absent this still works fts-only.
- [ ] Tie-break ordering stable across runs.

## Validation

```bash
npm run test -- ask/store ask/index
npm run test -- e2e -- --grep ask-index
```

## Dependencies

25 (hook contract), 28.

## Non-goals

Vector math (31), snippet highlighting (34 renders snippets from chunk text directly).

## Design References

- DESIGN §11 (FTS rules), §7.7; ADR-005
