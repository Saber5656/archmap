# 30 — Embeddings client with cache

## Summary

Implement the embeddings client per DESIGN §11: OpenAI-compatible `/embeddings` endpoint (Ollama
default), batching, per-chunk-hash caching, dims/model bookkeeping, and graceful unavailability.

## Context

Embeddings upgrade retrieval from fts-only to hybrid (ADR-005). They are strictly optional: any
failure degrades, never blocks. Same security gate as the chat client (B1).

## Scope

- In: `src/ask/embed.ts`.
- Out: storage schema (29 owns), re-rank math (31).

## Detailed Requirements

1. `createEmbedder({embeddings, llm, security}, logger): Embedder | null` — returns `null` when
   `embeddings.enabled: false` (disabled mode: NO preflight, NO network, NO table mutation, no
   model-change wipe; consumers go fts-only silently).
   - Effective baseUrl: `embeddings.baseUrl ?? llm.baseUrl`. ALL requests go through issue 21's
     shared transport (`openAiRequest`) — gate, `redirect: "error"`, timeout, retry, and
     redaction come for free. API key: reuses `llm.apiKeyEnv` (v1 rule, even when
     `embeddings.baseUrl` is set; a separate embeddings key is a recorded v2 item).
   - `embedBatch(texts: string[]): Promise<Float32Array[]>` — `POST {base}/embeddings`
     `{model, input: texts}`, batch ≤ 64 (split internally). Accepted response shape:
     `{data: [{embedding: number[], index: number}], …}` — results reordered by `index`;
     length mismatch, non-numeric entries, or cross-batch dims mismatch → treated as
     invalid-response (failure matrix below), never a crash.
2. Failure matrix (locked — supersedes any inherited fail-fast semantics from 21):
   | Case | Behavior |
   |---|---|
   | Security gate violation (non-loopback w/o opt-in) | `E_SECURITY_REMOTE_BLOCKED` PROPAGATES (exit 6, never degraded) |
   | `embeddings.enabled: false` | `null` embedder, no network, fts-only |
   | Network error / timeout / 5xx / 429 after retries | degrade: `available=false`, single warn `embeddings-unavailable` |
   | Auth 4xx (401/403) | degrade with warn `embeddings-auth-failed` (chat-LLM auth failures still exit 5 via 21; embeddings NEVER exit 5) |
   | Invalid response / dims mismatch | degrade with warn `embeddings-invalid-response` |
   Generate builds the index fts-only on any degrade; `ask` falls back to fts-only mode.
3. Caching: staleness is structural, not hash-joined — issue 29's chunk replacement is
   DELETE + INSERT (fresh `rid`), so `ON DELETE CASCADE` clears stale vectors automatically.
   Lookup = "embedding row exists for `chunk_rid`" AND `meta.embeddingModel == embeddings.model`.
   Integration API: `ensureEmbeddings(db, embedder | null): Promise<{embedded: number,
   cached: number, failed: number, available: boolean}>` — reads missing-vector chunks from the
   db, embeds OUTSIDE any transaction (network first), then writes vectors in one short
   DB-write transaction.
4. Model change detection: `meta.embeddingModel` differs from config → wipe embeddings table,
   re-embed all (warn `embeddings-reembedded`), update meta. Only when embedder non-null.
5. Query-time: `embedQuery(text): Promise<Float32Array | null>` — null on disabled/unavailable
   (availability cached per process after first failure).
6. Vectors stored as little-endian float32 BLOBs (Buffer), dims column authoritative.

## Acceptance Criteria

- [ ] Mock endpoint: batching splits at 64; response reordered by `index`; cache hit skips
      resend (request capture proves only missing-vector chunks sent); model swap re-embeds all.
- [ ] Disabled mode: `createEmbedder` returns null; zero network calls, zero table mutations,
      no wipe (spies).
- [ ] Failure matrix: each row tested — gate violation propagates exit 6; network/5xx degrade
      `embeddings-unavailable`; 401 degrades `embeddings-auth-failed`; dims mismatch degrades
      `embeddings-invalid-response`; generate completes fts-only in all degrade cases
      (integration with 29's flow).
- [ ] API key: `llm.apiKeyEnv` header present when set, absent when null; key value never in
      captured logs; missing named env var → E_CONFIG_INVALID.
- [ ] Requests use 21's transport (single fetch site assertion) with `redirect: "error"`.
- [ ] BLOB round-trip: store → read → Float32Array equality.

## Validation

```bash
npm run test -- ask/embed
```

## Dependencies

21, 28, 29.

## Non-goals

Similarity math (31), local in-process embedding models (v2 idea), token-level truncation
(chunk cap from 28 suffices).

## Design References

- DESIGN §11 (embeddings), §13.2 (gate applies); ADR-001, ADR-005
