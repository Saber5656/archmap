# 25 — `archmap generate` orchestration and incremental mode

## Summary

Wire the full pipeline into `archmap generate`: scan → secrets → parse → graph → cluster → rank →
metadata → plan → prose (unless `--no-llm`) → render → manifest → Ask index build (when enabled,
via issues 28–31 once available — behind a feature flag until then) → atomic output swap, with
incremental regeneration and a run report.

## Context

This is the product's main verb (DESIGN §3.1, §3.2 workflows 1/4) and the point where all
contracts meet. Incremental mode is what makes the CI loop cheap (§9).

## Scope

- In: `src/cli/generate.ts`, `src/core/pipeline.ts` (staged orchestrator),
  `src/wiki/manifest/evaluate.ts` (the shared freshness evaluator — OWNED here, consumed by
  issue 26), `src/ask/index-hook.ts` (stable no-op hook replaced by issue 29), run-report
  rendering.
- Out: check CLI/reporting (26), Ask subsystem internals (28–31).

## Detailed Requirements

1. Flags: `--no-llm` (forces `llm.enabled=false` for the run), `--force` (ignore parse/summary
   caches and staleness — full rebuild), `--dry-run` (run analysis + plan only; print planned
   pages/diagram counts/estimated LLM calls; write nothing), `--json`.
2. Startup order: load config → if LLM enabled: security gate + preflight (fail exit 5/6 BEFORE
   any analysis) → scan → secret filter (08) — the pipeline contract is that EVERY downstream
   content sink (extractors, metadata, fact sheets/prompts, renderer inputs, chunker) reads
   content ONLY through the filter's `readAnalyzable`; `secret-excluded` files reach no sink.
3. Shared evaluator (owned here):
   `evaluateFreshness({manifest, inventory, currentVersions: {toolVersion, promptVersion,
   model, configHash}, wikiFiles: Map<relPath, string>}): FreshnessReport` — pure; per-page
   stale reasons as stable tokens `input-changed:<path>`, `input-deleted:<path>`,
   `new-file:<path>` (matched against the manifest's persisted per-page `sources`),
   `tool-version-changed`, `prompt-version-changed`, `model-changed`, `config-changed`,
   `page-file-missing`; plus `handEdited[]` detection via renderHash recomputation (prose slots
   normalized). Issue 26 imports THIS function; full reason-token semantics live here.
4. Incremental algorithm (default):
   - If manifest exists and `--force` absent: evaluate freshness with the shared evaluator.
   - Fresh pages: reuse their rendered content verbatim (including current prose-slot content —
     previously generated prose carries forward without LLM calls; hand edits inside slots of
     FRESH pages therefore survive until the page goes stale, per DESIGN §9).
   - Stale/new pages: full facts→prose→render path (slot content regenerated).
   - Deleted clusters: their pages removed from output (swap semantics handles it).
   - `_meta/*` rewritten; when ALL pages are reused unchanged, `generatedAt` is preserved from
     the previous manifest so the run is byte-identical end to end.
5. Parse cache + summary cache used per issues 09/23; `pruneParseCache` called at end.
6. Atomic write: assemble the complete output tree (fresh + regenerated pages) in memory /temp
   and `swapDirAtomic` into `output.dir` (05). Wiki dir contains ONLY generated files; foreign
   files found in existing output.dir → warn `foreign-files-in-output` listing them, and they are
   dropped from the new tree (documented; users keep hand-written docs outside output.dir).
7. Ask index hook (`src/ask/index-hook.ts`): pipeline calls
   `buildAskIndex(ctx): Promise<AskIndexStats | {skipped: string}>` AFTER the swap when
   `ask.enabled`. This issue ships the no-op implementation returning
   `{skipped: "not-implemented"}` with warning `ask-index-unavailable`; issue 29 replaces it.
   Degradation carve-out: `E_SECURITY_*` and `E_CONFIG_INVALID` raised through the hook
   PROPAGATE (exit 6/2 — a blocked remote embeddings endpoint must never be swallowed); only
   operational failures degrade with warning `ask-index-failed`.
8. Run report (human table + `--json` data): counts (files scanned/analyzed/excluded-secret,
   clusters, pages rendered/reused, diagrams), LLM stats (calls, cache hits, tokens est/reported,
   budget consumed, skipped-budget pages, failed pages), secret findings summary, ask index stats,
   duration per stage, warnings list. Exit 0 on success (including degraded warnings), exit
   5/6/2/1 per taxonomy.
9. Ordering guarantee: two consecutive runs with no repo changes → second run reports all pages
   reused, zero LLM calls, and produces a byte-identical wiki INCLUDING `_meta/manifest.json`
   (via the generatedAt-preservation rule) — the dogfood invariant.

## Acceptance Criteria

- [ ] E2E on `fixture-mixed` (mock LLM): full run renders golden wiki + manifest; second run
      reuses everything (0 calls, byte-identical output incl. manifest); touching one file in
      cluster X regenerates only X's page + `_meta` (+ overview pages per aggregate-input rule).
- [ ] Evaluator unit matrix (owned here): every reason token fires on its fixture (changed /
      deleted / new-in-scope via manifest `sources` / each version change / missing page file);
      hand-edit outside slots → `handEdited`, inside slots on a fresh page → clean.
- [ ] `--no-llm` produces the golden structural wiki; `--dry-run` writes nothing (fs snapshot
      unchanged) and reports plan; `--force` re-parses (cache spies).
- [ ] LLM preflight failure exits 5 before scanning (ordering asserted); remote URL without
      opt-in exits 6.
- [ ] Security (B2): on `fixture-secrets` under both policies, canary strings absent from
      rendered wiki, captured prompts, run report, and logs (index coverage lands with 29);
      `secret-excluded` files reach no sink (accessor spy).
- [ ] Ask hook: no-op hook warns `ask-index-unavailable`; a hook throwing
      `E_SECURITY_REMOTE_BLOCKED` propagates as exit 6 (not degraded).
- [ ] Foreign file in output.dir warned and absent after swap; interrupted run (kill during
      build) leaves previous wiki intact (atomic swap test reuse from 05).
- [ ] Run report golden (normalized durations) in json mode.

## Validation

```bash
npm run test -- cli/generate core/pipeline
node dist/cli/index.js generate --no-llm --json | jq .data.pages
```

## Dependencies

08, 19, 20, 23, 24.

## Non-goals

check UX (26), ask retrieval quality (31/32), viewer/MCP.

## Design References

- DESIGN §3.1–3.2, §9 (incremental), §5 (atomic swap), §10 (degradation)
