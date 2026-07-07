# 32 — `archmap ask` with validated citations

## Summary

Implement `archmap ask "<question>"`: retrieve → `askAnswer` prompt → validate citations against
retrieved ids → render answer + citations (or the honest refusal), per DESIGN §11 and the
AnswerRecord contract.

## Context

Ask is the conversational surface of the wiki. Its trust property (ADR-003): the model cannot cite
anything it wasn't shown; insufficient evidence is a first-class outcome, not a failure.

## Scope

- In: `src/cli/ask.ts`, `src/ask/answer.ts`, `scripts/ask-smoke.mjs` (deterministic
  fixture+mock smoke used in Validation).
- Out: retrieval internals (31), MCP exposure (35).

## Detailed Requirements

1. Precondition matrix (locked, checked in this order):
   | Condition | Result |
   |---|---|
   | `ask.enabled: false` | exit 2 `E_CONFIG_INVALID`, hint `set ask.enabled: true and re-run archmap generate` |
   | index missing/empty | exit 4 `E_ARTIFACTS_MISSING`, hint `run archmap generate first` |
   | index corrupt (29 read-only mode) | exit 4 with regenerate hint |
   | normal ask + `llm.enabled: false` | exit 2 `E_CONFIG_INVALID` (config problem, not availability) |
   | remote endpoint w/o opt-in | exit 6 (gate) |
   | endpoint down / auth failed (preflight) | exit 5 |
   `--retrieve-only` skips ONLY the chat-LLM path: no gate/preflight for chat, and retrieval
   runs with `embedder = null` (forced fts-only — ZERO network in retrieve-only mode; locked).
2. Flags: `--top-k <n>` (override config, bounds 1..50), `--json`, `--retrieve-only` (print
   retrieval hits table; works with LLM disabled).
3. `answer.ts`: `answerQuestion({query, retrieval, client, language}): Promise<AnswerRecord>`:
   - Build `askAnswer` prompt (22) with chunks as `{id, header: "<path>:<span> (<symbol|kind>)",
     text}`; validate with 22's `parseModelJson`; on `{ok: false}` retry ONCE via
     `repairPrompt`; second failure → canonical insufficient record with warning
     `answer-invalid` (never a crash).
   - Citation validation: `citedChunkIds ⊆ retrieved ids` — unknown ids DROPPED with warning
     `citation-dropped`; zero valid citations AND `insufficientEvidence=false` → treated as
     insufficient (defense against confident-but-uncited answers).
   - Field mapping: the model's `answerMarkdown` (22's schema) becomes AnswerRecord.`answer`
     (DESIGN §7.7) after sanitization.
   - Canonical insufficient AnswerRecord (locked; used for model-flagged, all-dropped, and
     repair-failure paths alike): `insufficientEvidence: true`, `citations` = valid ones only,
     `answer` = the localized refusal template text (the model's unsupported prose is
     DISCARDED — it must not leak via `--json`).
   - Question length cap 2000 chars (`E_USAGE` beyond); question text fenced (uniform rule).
   - Answer markdown sanitized with 19's `sanitizeProse` before display/emission.
4. Rendering (human): answer markdown to stdout; `Sources:` list with `path:span` lines; wiki
   chunks additionally show their page slug derived mechanically: path under `output.dir` with
   the `.md` suffix stripped (e.g. `docs/wiki/modules/ask.md` → `modules/ask`); insufficient →
   refusal template + top-3 hits table ("closest material").
5. `--json`: AnswerRecord as the envelope `data`; warnings in the envelope `warnings`.
6. Latency: print a stderr spinner via logger.progress during LLM call (TTY only).

## Acceptance Criteria

- [ ] Mock-LLM happy path: cited ids resolve to shown chunks; human + json renders golden; wiki
      chunk source line shows the derived page slug.
- [ ] Contract-repair path: malformed canned JSON → repair prompt → success; double failure →
      canonical insufficient record + `answer-invalid` warning.
- [ ] Fabricated citation id → dropped + warning; all-fabricated → canonical insufficient
      record; `--json` golden proves the model's unsupported prose is absent.
- [ ] `insufficientEvidence: true` canned → refusal template with closest-material table (human)
      and canonical record (json).
- [ ] Precondition matrix: every row tested with its exact exit code, including
      `ask.enabled: false` → 2 and LLM-disabled normal ask → 2.
- [ ] `--retrieve-only`: zero network calls (fetch spy — no gate, no preflight, no embedQuery);
      works with `llm.enabled: false`.
- [ ] Long question → exit 2; remote URL unauthorized → exit 6.
- [ ] Answer markdown passes sanitizeProse (hostile canned answer: html/absolute links
      neutralized).

## Validation

```bash
npm run test -- cli/ask ask/answer
# deterministic smoke: fixture + mock LLM
npm run build && node scripts/ask-smoke.mjs   # scaffolds fixture-ts copy, generate, mock ask
```

## Dependencies

22, 31.

## Non-goals

Multi-turn chat, DeepResearch loops (v2), answer caching.

## Design References

- DESIGN §11 (answering rules), §7.7 (AnswerRecord), §13.4 (sanitize), §14 (exit codes)
