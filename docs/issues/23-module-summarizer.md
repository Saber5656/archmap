# 23 — Module prose summarizer (map step)

## Summary

For each module page, assemble a compact deterministic fact sheet, call the LLM through the
prompt pack, validate/repair, and cache results by `(factsHash, model, PROMPT_VERSION, language)`
per DESIGN §10.

## Context

This is the map step of prose generation (ADR-003): fact sheets in, two prose fields out, page
budget respected, failures isolated per page.

## Scope

- In: `src/llm/synthesize/fact-sheet.ts`, `src/llm/synthesize/module-summarizer.ts`,
  `src/llm/synthesize/cache.ts`.
- Out: reduce step (24), orchestration (25).

## Detailed Requirements

1. `fact-sheet.ts`: `moduleFactSheet(clusterId, facts: ModuleFactArtifacts, budgetTokens):
   {text, factsHash}` with the locked input type
   `ModuleFactArtifacts = {cluster (id, label, metrics), files: [{path, loc, fanIn, fanOut,
   doc?, score}], symbols: [{name, kind, signature?, doc?, filePath, fileScore, spanStart}],
   depsIn: clusterId[], depsOut: clusterId[], externals: [{name, importers}],
   entrypoints: [{path, reason}], readmeH2: string[]}` (assembled by the caller from
   post-secret-filter pipeline artifacts — issue 08's accessor is the only content source).
   Sheet sections and packing (locked): cluster header; files table (cap 15, order: score desc,
   path ASC); exported symbols (cap 60, order: exported first, fileScore desc, spanStart ASC);
   deps in/out (caps 10 each, id/name ASC); externals (cap 10, importers desc, name ASC);
   entrypoints (all, path ASC); readmeH2 = first 3 titles verbatim (no relevance heuristic).
   Pack greedily in that section order until `estimateTokens(text) ≤ budgetTokens · 0.6`;
   truncation marker `…(+N omitted)` per cut section. Deterministic: same inputs → same text.
   `factsHash = hashJson(the PACKED sheet's structured inputs — the post-cap, post-truncation
   lists + budgetTokens)`, so cache keys track exactly what shaped the prompt.
2. Code snippets: NONE in v1 fact sheets (signatures + docs only) — keeps prompts small and
   reduces secret/injection surface; documented decision.
3. Fencing is 22's job (the template fences the sheet as one ARCHMAP_DATA block); this issue
   must pass the raw sheet text only into the template — verified by prompt-capture test.
4. `module-summarizer.ts` (locked API):
   ```ts
   summarizeModules(opts: {plan, factsByCluster: Map<clusterId, ModuleFactArtifacts>,
     client, budget, llmEnabled: boolean, language: string, model: string,
     forceCache: boolean, cacheDir: string, logger}): Promise<ModuleProseOutcome>
   type ModuleProseResult =
     | {status: "ok", responsibility: string, keyFlow: string}
     | {status: "llm-disabled" | "skipped-budget" | "failed", reason?: string}
   type ModuleProseOutcome = {bySlug: Map<pageSlug, ModuleProseResult>,
     stats: {llmCalls: number, cacheHits: number, skippedBudget: number, failed: number}}
   ```
   - Iterate module pages in PLAN order (= rank order; best pages win budget); `llmEnabled:
     false` → all pages `llm-disabled` with zero calls.
   - Cache lookup first (`cache.ts`: `.archmap/cache/summaries/<key>.json`, key =
     `sha256(factsHash + model + PROMPT_VERSION + language)`); hit → no call, no budget;
     `forceCache: true` bypasses reads (issue 25 `--force`).
   - Budget protocol: the client (21) consumes budget inside `complete()` for the initial AND
     the repair call separately; `BudgetExhaustedError` on either → that page and all remaining
     pages become `skipped-budget` (warning `budget-exhausted`), run continues per DESIGN §10.
   - Validate via 22 (`parseModelJson`); one repair round via `repairPrompt`; second failure →
     `{status: "failed"}` (warn `prose-failed:<slug>`), continue. `stats.llmCalls` counts
     actual HTTP calls (repairs included).
   - Concurrency via client's semaphore; page slug passed as `label`.
5. Renderer/24 mapping (locked): `ok` → slots `responsibility`/`keyFlow` filled; every other
   status → both slots disabled-marked; issue 24's digest uses `ok` results only (others fall
   back to file-doc lines marked `(structural)`).

## Acceptance Criteria

- [ ] Fact sheet golden test on `fixture-mixed` cluster: byte-stable, per-section caps and
      tie-breaks proven, budget-respecting truncation at a tiny budget with per-section marker.
- [ ] Mock-LLM pipeline test: N pages → responses cached; second run makes zero LLM calls
      (spy) with identical output; `forceCache` re-calls.
- [ ] Budget exhaustion mid-run (including exhaustion ON a repair call): high-rank pages
      summarized, remainder `skipped-budget`, run completes with warning; deterministic
      ordering; `stats.llmCalls` counts repairs.
- [ ] Malformed model output → repair path → success; double failure → `failed` isolated to
      that page.
- [ ] Cache key changes when any of factsHash/model/PROMPT_VERSION/language changes (4 tests).
- [ ] Security (B2/B3): on `fixture-secrets` (exclude AND redact policies), canary strings are
      absent from fact sheets, captured prompts, summary cache files, and logs; prompt capture
      shows exactly one fenced data block per call with no sheet text outside fences.

## Validation

```bash
npm run test -- llm/synthesize/module
```

## Dependencies

08, 17, 22.

## Non-goals

Overview synthesis (24), writing prose into pages (19's slots via 25).

## Design References

- DESIGN §10 (caching, budget, isolation), §8.1 (module slots); ADR-003
