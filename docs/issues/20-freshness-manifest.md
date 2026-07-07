# 20 — Freshness manifest writer

## Summary

Produce `_meta/manifest.json` (WikiManifest, DESIGN §7.6): for every rendered page, the exact
input files with hashes, the input-set hash, the render hash, and the tool/prompt/model/config
versions that freshness comparison (26) keys on.

## Context

The manifest is the mechanism behind pillar P1: `check` diffs it against reality. It must be
canonical, minimal-diff, and complete — a page not in the manifest cannot be checked.

## Scope

- In: `src/wiki/manifest/build.ts`, `src/wiki/manifest/config-hash.ts`, WikiManifest zod schema.
- Out: the check algorithm (26), writing to disk (25).

## Detailed Requirements

1. `build.ts`: `buildManifest({plan, renderedWiki, proseStatusBySlug, inventory, config,
   toolVersion, promptVersion, model}): WikiManifest`. `promptVersion` is a plain string
   argument supplied by the caller (issue 25 wires the prompt pack's const later; tests and
   no-LLM runs pass `"(none)"`) — this issue has NO dependency on issue 22.
   - Per page: `sources` = the page's PagePlan sources (globs or explicit file lists) persisted
     verbatim (sorted) — the freshness SCOPE that `check` (26) uses for new-file detection;
     `inputs` = concrete files matched by those sources against the inventory (sorted, with
     current fingerprints); `inputSetHash` = hash of the sorted fingerprint lines;
     `renderHash` = hash of the page content with all prose-slot bodies replaced by the token
     `@slot` (prose regeneration alone doesn't change renderHash — it tracks deterministic
     structure only); `prose` = true ONLY when at least one slot contains successful
     LLM-generated prose (from `proseStatusBySlug`; disabled markers, budget fallbacks, and
     no-LLM notices are `prose: false`).
   - Inventory fingerprint line format (locked, covers hashless `secret-excluded` entries):
     `<path>:<hash|"nohash">:<size>:<flags-joined>` — used for `inputs`, `inputSetHash`, and
     the aggregate `inventoryHash`.
   - Fixed pages with `sources: ["*"]`: inputs = the **aggregate** entry
     `{path: "(inventory)", hash: inventoryHash}` where `inventoryHash` = hash of the full
     sorted `path:hash` list — avoids exploding the manifest with every file for overview pages
     while still detecting any change. Dependencies page: concrete manifest paths (from the
     PagePlan sources) PLUS the aggregate `(inventory)` entry — its externals tables derive
     from the whole graph, so any file change must dirty it.
2. `config-hash.ts`: `freshnessConfigHash(config)` = hash of the canonical subset that affects
   output: `output.language`, `analysis.*`, `wiki.*`, `llm.enabled`,
   `security.secretScan`, `security.secretPolicy`. Documented exclusions (non-output-affecting):
   `viewer.*`, `ask.*`, `llm` connection fields (baseUrl/timeouts/concurrency),
   `security.allowRemoteLlm`. Changing an excluded key must NOT dirty the wiki (tested).
3. `model` field: literal model string or `"(none)"` for no-LLM runs; `promptVersion` recorded
   verbatim from the argument (`"(none)"` for no-LLM runs).
4. `generatedAt`: ISO 8601 UTC, informational (excluded from any comparison; documented).
5. Serialization: canonical pretty JSON, sorted pages by slug.
6. Invariant: every rendered `.md` page appears exactly once; `_meta/*` files themselves are not
   pages.

## Acceptance Criteria

- [ ] Manifest for golden fixture matches checked-in golden byte-for-byte (except `generatedAt`,
      normalized in the test) — including per-page `sources`.
- [ ] Prose-only change (different slot text, same facts) keeps `renderHash` and `inputSetHash`
      identical (test).
- [ ] `prose` semantics: page with successful LLM prose → true; disabled-notice page and
      budget-fallback page → false (three-state test).
- [ ] Deterministic-structure change (new symbol) changes `renderHash` and input hashes.
- [ ] Config matrix: `viewer.port` change → `configHash` unchanged; `output.language` change →
      flips; `security.secretScan` flip → flips.
- [ ] Fixture with a `secret-excluded` (hashless) `.env` entry: fingerprint lines use the locked
      `nohash` format; aggregate `inventoryHash` stable across runs.
- [ ] Overview page uses the aggregate inventory entry; dependencies page = manifests +
      aggregate; module pages list concrete files.

## Validation

```bash
npm run test -- wiki/manifest
```

## Dependencies

19.

## Non-goals

Staleness evaluation (26), disk IO (25).

## Design References

- DESIGN §7.6 (schema), §9 (what check compares), §5 (canonical form)
