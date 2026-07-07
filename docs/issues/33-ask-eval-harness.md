# 33 — Ask evaluation harness

## Summary

Build the informational eval harness for Ask quality per DESIGN §16: fixture QA sets with
mechanical grading (citation paths, keyword presence, refusal correctness), a scorecard runner,
and a recorded baseline per model.

## Context

Local-model quality is the top known unknown (DESIGN §2.4). The harness turns "does Ask work with
qwen3:8b?" into a number we can track across prompt versions and models. Non-CI-gating in v1.

## Scope

- In: `tests/ask-eval/questions/*.yaml`, `scripts/eval-ask.ts`, `npm run eval:ask`,
  `docs/eval/BASELINES.md` (results log).
- Out: prompt changes it might motivate (future issues).

## Detailed Requirements

1. Question format (YAML, one file per fixture repo):
   ```yaml
   - id: ts-retrieval-location
     question: "Where is hybrid retrieval implemented and what are its two stages?"
     mustCiteAny: ["src/ask/retrieve.ts"]      # ≥1 citation path must match (prefix match)
     mustMentionAny: ["FTS", "cosine"]          # answer text, case-insensitive
     mustNotMention: []                         # optional
     expectRefusal: false
   - id: ts-out-of-scope
     question: "What is the production deployment topology?"
     expectRefusal: true                        # insufficientEvidence expected
   ```
   ≥ 12 questions for `fixture-ts`, ≥ 8 for `fixture-py` (mix: symbol-location, behavior,
   architecture, dependency, 2 refusal cases each).
2. Runner `eval-ask.ts` — MUST drive the real pipeline components (config loader, retrieval,
   answer path with its LLM client, security gate, redaction); no bespoke bypass paths:
   - Modes: `--mock` (canned perfect + canned bad responses — validates the GRADER itself),
     `--ollama` (real local model; HARD-requires env `ARCHMAP_E2E_OLLAMA=1`, refuses otherwise;
     endpoint gate applies as everywhere), `--retrieval-only` (grades only `mustCiteAny`
     against retrieval top-K — no LLM; CI-runnable).
   - Grading semantics (locked): `mustCiteAny` = at least ONE citation path prefix-matches any
     listed path; `mustMentionAny` = at least ONE listed term appears (case-insensitive);
     `mustNotMention` = NONE appears; refusal questions (`expectRefusal: true`) are EXCLUDED
     from `retrievalHitRate`'s denominator and, in answer modes, pass iff
     `insufficientEvidence: true`.
   - Per question: run the real `ask` pipeline programmatically (not subprocess) → grade:
     `citation` pass/fail, `mention` pass/fail, `refusal` pass/fail; latency recorded.
   - Scorecard: table per suite (id, C/M/R, ms) + totals `{citationRate, mentionRate,
     refusalAccuracy, retrievalHitRate}`; `--json` output.
3. Baseline logging: `docs/eval/BASELINES.md` — append-style table (date, model, PROMPT_VERSION,
   suite, rates) maintained MANUALLY via `npm run eval:ask -- --record` which prints the
   ready-to-paste row (no auto-commit).
4. Thresholds: `retrievalHitRate ≥ 0.9` in `--retrieval-only` mode IS CI-gated (deterministic —
   no model involved; DESIGN §16 and ISSUE_PLAN §6 record this carve-out). All model-dependent
   rates (citation/mention/refusal in answer modes) are informational only in v1.
5. CI: `--retrieval-only` mode runs in CI; model modes never run in CI (guard: runner exits
   with `E_USAGE` if `--ollama` is used while `CI=true`).

## Acceptance Criteria

- [ ] Grader unit-tested via `--mock` (perfect → 100%, sabotaged → expected failures per rule,
      incl. `mustNotMention` and refusal-in-both-directions cases).
- [ ] `--retrieval-only` runs in CI and enforces retrievalHitRate ≥ 0.9 on fixtures; refusal
      questions excluded from its denominator (test).
- [ ] `--ollama` without `ARCHMAP_E2E_OLLAMA=1` refuses; with `CI=true` refuses; runner uses the
      real client/gate (a remote baseUrl without opt-in fails exit 6 through the harness).
- [ ] Real-model run produces the scorecard and `--record` row locally (documented example
      output committed to docs/eval/BASELINES.md from the implementer's machine, model named).
- [ ] This issue also runs the ask half of the real-model smoke (`npm run e2e:ollama` gains an
      ask step here — 27 shipped it generate-only).

## Validation

```bash
npm run eval:ask -- --mock
npm run eval:ask -- --retrieval-only
ARCHMAP_E2E_OLLAMA=1 npm run eval:ask -- --ollama --record   # local
```

## Dependencies

27, 32.

## Non-goals

LLM-as-judge grading (v2), CI-gating model quality, prompt optimization work itself.

## Design References

- DESIGN §16 (eval row), §2.4 (unknown #1), §11
