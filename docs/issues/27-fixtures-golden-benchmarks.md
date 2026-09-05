# 27 — Fixture repos, golden tests, benchmarks

## Summary

Build the shared test estate: the five fixture repos, golden wiki outputs, the mock
OpenAI-compatible server, cross-platform golden CI, the real-Ollama opt-in smoke, and the
performance benchmark harness for DESIGN §15 targets.

## Context

Earlier issues each added narrow fixtures; this issue consolidates the product-level E2E estate
that waves 3–5 (and the dogfood job, 40) build on. Determinism claims (ADR-002/003) become
enforced here.

## Scope

- In: `tests/fixtures/{fixture-ts,fixture-py,fixture-mixed,fixture-secrets,fixture-inject}/`
  (finalized), `tests/golden/**`, `tests/helpers/mock-llm.ts`, `tests/e2e/*.test.ts`,
  `scripts/bench.ts`, synthetic large-repo generator `scripts/gen-large-fixture.ts`.
- Out: security assertions on fixtures (39 uses them), CI dogfood wiring (40).

## Detailed Requirements

1. Fixtures (committed, small, license-header-free):
   - `fixture-ts`: ~15 files, tsconfig paths alias, workspaces-free, bin entry, dynamic import,
     generated file, nested gitignore.
   - `fixture-py`: ~12 files, `src/pkg` layout, `__main__.py`, `__all__`, relative imports,
     pyproject scripts.
   - `fixture-mixed`: TS + Py + Go + shell + binary + oversized + `.env` + README with H2s —
     the primary golden source (already partially exists from earlier issues; finalize here).
   - `fixture-secrets` (from 08) and `fixture-inject`: README + a TS file containing prompt
     injection payloads ("ignore previous instructions", fake ARCHMAP_DATA sentinels, mermaid
     breakouts, HTML/script in docstrings).
2. Golden outputs: full `docs/wiki/` trees for fixture-ts/py/mixed in `--no-llm` mode, en +
   (mixed only) ja. Regeneration script `npm run golden:update` (explicitly manual; CI compares
   only). Comparator rule (locked): byte comparison AFTER normalizing `generatedAt` to a fixed
   value on both sides — everything else byte-exact.
3. `mock-llm.ts`: in-process http server implementing `/v1/chat/completions`, `/v1/models`, and
   `/v1/embeddings` with this locked minimal contract (unblocks issue 30's tests): request
   `{model, input: string[]}` → response `{data: [{embedding: number[16], index}], model,
   usage: {prompt_tokens}}`, vectors deterministic (seeded from sha256 of each input), dims 16.
   Scriptable per-test (response queue, malformed-JSON mode, 500/429 modes, latency injection);
   records all requests + bodies for later assertions. Prompt-content SECURITY verdicts (canary
   absence, sentinel integrity) are issue 39's scope — this issue only ships the capture
   plumbing.
4. E2E tests: generate→check fresh loop; incremental single-file touch; `--no-llm` goldens on
   macOS + Linux CI (byte-equal after the comparator rule); prose path with mock LLM (canned
   JSON → slots filled; goldens with prose for fixture-ts); `fixture-inject` generates
   successfully with mock LLM (deep injection assertions live in 39).
5. Real-model smoke (NOT CI): `ARCHMAP_E2E_OLLAMA=1 npm run e2e:ollama` — GENERATE-ONLY on
   fixture-ts against local Ollama; asserts structural invariants (slots non-empty, prose
   inside markers), never content equality. (The ask smoke belongs to issue 33.)
6. Benchmarks (`scripts/bench.ts`, informational output table, no CI gate):
   - `gen-large-fixture.ts` synthesizes a ~300k LOC repo (deterministic seed): 3k TS files
     across 60 clusters with realistic import density + git history (scripted commits).
   - Measures NOW: `generate --no-llm` cold/warm, `check`. Other DESIGN §15 rows print as
     `skipped (issue 31)` / `skipped (issue 38)` — those issues extend this script.
7. CI wiring: golden + e2e jobs added to the matrix (macOS + Linux); large-fixture bench runs on
   a manual `workflow_dispatch` job only (runtime cost).

## Acceptance Criteria

- [ ] All goldens identical across the two OS runners under the locked comparator (proven by CI).
- [ ] Mock LLM supports the scripted failure modes incl. the locked embeddings contract; at
      least one test exercises each mode; request capture exposes bodies to other suites.
- [ ] Incremental e2e proves the "0 calls, byte-identical" dogfood invariant (25 AC reused at
      product level).
- [ ] Bench table prints measured numbers for the available §15 rows and explicit
      `skipped (issue NN)` rows for the rest; cold no-LLM generate ≤ 60 s on CI hardware
      recorded in the PR description (informational).
- [ ] `fixture-inject` generates successfully with mock LLM (deep assertions live in 39).

## Validation

```bash
npm run test -- e2e
npm run bench   # informational
ARCHMAP_E2E_OLLAMA=1 npm run e2e:ollama   # local only, requires Ollama; generate-only
```

## Dependencies

25, 26.

## Non-goals

Security verdicts (39), CI dogfood job (40), ask eval (33).

## Design References

- DESIGN §15 (targets), §16 (strategy rows 1–4), §2.4 (scale unknowns)
