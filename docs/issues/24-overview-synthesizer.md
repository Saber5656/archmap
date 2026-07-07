# 24 — Overview/architecture prose synthesizer (reduce step)

## Summary

Produce repo-level prose (overview page, architecture narrative, cluster one-liners, dependency
notes) in a single reduce call over module summaries + repo metadata, with the same validation,
caching, and degradation rules as the map step.

## Context

DESIGN §8.1 assigns five prose slots across index/architecture/dependencies pages; the `overview`
template (22) returns them in one response to keep small-model context coherent and calls minimal.

## Scope

- In: `src/llm/synthesize/repo-fact-sheet.ts`, `src/llm/synthesize/module-digest.ts`,
  `src/llm/synthesize/overview-synthesizer.ts`.
- Out: slot injection into pages (25).

## Detailed Requirements

1. `repo-fact-sheet.ts`: `repoFactSheet(inputs, budgetTokens): {text, factsHash}` —
   deterministic sheet with: repo name/description/license (16), stats (files, loc, languages
   histogram), cluster table (id, label, files, loc, score, top-3 file docs), entrypoints with
   reasons, external deps top 15 by importers, README extract (H1, first paragraph, H2 list).
   Packed under `overview` page budget · 0.6 with issue 23's packing/tie-break style;
   `factsHash = hashJson(packed structured inputs + budgetTokens)` (same convention as 23).
2. `module-digest.ts`: `moduleDigest(planModulePages, moduleProse): {text, digestHash}` — one
   line per module page in plan (rank) order, cap 40 lines: `ok` results contribute
   `clusterId: first sentence of responsibility` (≤ 160 chars); all other statuses contribute
   `clusterId: <top file doc line> (structural)`. `digestHash = hashJson(ordered digest lines)`.
3. `overview-synthesizer.ts`: one LLM call via `overview` template; cache key
   `sha256(factsHash + digestHash + model + PROMPT_VERSION + language)`; validate → one repair →
   degrade (22's caller protocol; budget consumed inside the client per 21, including the
   repair call). Budget exhausted before/at this call → all five slots disabled with reason
   `skipped-budget`.
4. `clusterOneLiners` — canonical key is `clusterId` (matches 22's prompt schema). Model entries
   whose key is not a planned module page's clusterId are DROPPED (hallucination filter).
   The synthesizer then COMPLETES the map itself: every planned module cluster missing an LLM
   one-liner gets a deterministic structural fallback (its top file's doc line, or its label
   when no doc exists). Renderer 19 always receives a complete map.
5. Output (locked):
   ```ts
   type SlotResult = {markdown: string} | {disabled: true, reason: string}
   interface OverviewSynthesisResult {
     slots: Record<"whatIsThis"|"whereToStart"|"architectureNarrative"|"dependencyNotes", SlotResult>
     clusterOneLiners: Record<clusterId, {text: string, source: "llm" | "structural"}>
     stats: {llmCalls: number, cacheHits: number, failed: number}
   }
   ```
   Double validation failure → all four prose slots `{disabled, reason: "prose-failed"}` while
   `clusterOneLiners` still returns the complete structural map.

## Acceptance Criteria

- [ ] Repo fact sheet + digest goldens on `fixture-mixed`; digest cap and `(structural)` lines
      proven; both hashes stable across runs.
- [ ] Mock-LLM: full response fills all four slots + one-liners; hallucinated cluster key
      dropped; missing one-liner completed with structural fallback (`source: "structural"`).
- [ ] Cache hit on second run (zero calls); key sensitive to digest change (module prose change
      → new overview).
- [ ] Budget-exhausted-before-reduce → all four slots `skipped-budget`, run continues.
- [ ] Double validation failure → synthesizer returns `prose-failed` slot records + complete
      structural one-liners (rendered-page integration is issue 25's test).
- [ ] Security (§13.4): overview prompt capture shows repoFactSheet AND moduleDigest each inside
      ARCHMAP_DATA sentinels, nothing outside; logs carry metadata only.

## Validation

```bash
npm run test -- llm/synthesize/overview
```

## Dependencies

23.

## Non-goals

Per-cluster narrative beyond one-liners (module pages own that), multi-call refinement loops.

## Design References

- DESIGN §8.1 (slot inventory), §10 (reduce step, caching); ADR-003
