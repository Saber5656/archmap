# 26 — `archmap check` freshness gate

## Summary

Implement `archmap check` per DESIGN §9: evaluate wiki freshness against the manifest using the
shared evaluator, report stale pages with reasons, detect hand-edits, and exit with the CI
contract codes.

## Context

This command IS pillar P1's enforcement point: it must be fast (≤10 s @300k LOC), precise about
*why* something is stale, and safe to wire into any CI.

## Scope

- In: `src/cli/check.ts` (CLI, reporting, version derivation, manifest loading/validation).
  The shared evaluator `src/wiki/manifest/evaluate.ts` is OWNED by issue 25 — this issue
  consumes it unchanged.
- Out: regeneration (25 is the fix), evaluator internals (25).

## Detailed Requirements

1. Evaluator consumption: import `evaluateFreshness` from issue 25 (static import — a test
   asserts 25 and 26 share the single implementation). New-file detection works from the
   manifest's persisted per-page `sources` (issue 20); fixed pages use the aggregate inventory
   fingerprint (any change dirties them — documented consequence of DESIGN §7.6).
2. Manifest loading (trust boundary — the manifest is a committed, tamperable file): validate
   against the WikiManifest zod schema; `page.file` resolved via `joinConfined(output.dir, …)`
   and input paths must be repo-relative (confinement via `joinConfined(repoRoot, …)`);
   malformed or path-escaping manifest → `E_CONFIG_INVALID` (exit 2) naming the offending
   field; traversal fixtures required.
3. Current version derivation (locked): `toolVersion` from the build constant;
   with `llm.enabled: false` → `promptVersion = model = "(none)"`; with LLM enabled →
   configured `llm.model` + the prompt pack's `PROMPT_VERSION`. Comparison rules: `toolVersion`
   major.minor only; `configHash` exact; `promptVersion`/`model` compared ONLY when the
   manifest has any page with `prose: true` (structural wikis never go stale from
   prompt/model drift).
4. `check.ts`: load manifest (missing → exit 4 `E_ARTIFACTS_MISSING` with hint to run
   generate); scan inventory (issue 07 `scan`, full-inventory hashing — the ≤10 s DESIGN §15
   target is met by the hash worker pool; a partial-hash fast path is a recorded v2
   optimization, not assumed here); evaluate; render:
   - human: table of stale pages (slug | reasons), then new/deleted file counts, hand-edited
     warnings; green "wiki is fresh" when clean.
   - `--json`: the standard issue-02/04 envelope, whose `data` payload is exactly
     `{stalePages: [{slug, reasons}], newFiles, deletedFiles, handEdited, versions:
     {tool, prompt, model, matches}}` (CI consumers read `.data.stalePages`); errors use the
     standard error envelope.
5. Exit: 3 when `stalePages.length > 0`; else 0 (hand-edits alone → 0 with warnings; strictness
   flag deferred to v2). 2/4 per taxonomy.

## Acceptance Criteria

- [ ] Fixture matrix: pristine → exit 0; edited source file → exit 3 with
      `input-changed:<path>`; deleted input → 3; added file in cluster scope (via manifest
      `sources`) → 3 `new-file`; hand-edited page body → exit 0 + handEdited warning;
      prose-slot manual edit → clean.
- [ ] Version matrix: patch bump clean, minor bump stale, config change stale;
      prompt/model changes stale ONLY when manifest has prose pages; structural manifest +
      `llm.enabled=false` run clean; structural manifest + LLM-enabled run clean
      (prose gate proven both ways).
- [ ] `--json` emits the standard envelope; `.data` validates against a checked-in JSON schema;
      error cases use the error envelope.
- [ ] Missing manifest → exit 4 with hint; missing single page file → stale with
      `page-file-missing`; tampered manifest (`page.file: "../../x"`, absolute input path,
      unknown schema fields) → exit 2 naming the field, no out-of-root fs access (spy).
- [ ] Shared evaluator: 25 and 26 import the same function (static import test).

## Validation

```bash
npm run test -- cli/check wiki/manifest/evaluate
node dist/cli/index.js check --json; echo "exit=$?"
```

## Dependencies

20, 25.

## Non-goals

Auto-fix/regenerate (that's generate), watch mode, strict hand-edit mode (v2).

## Design References

- DESIGN §9 (algorithm, exit codes), §7.6, §15 (perf target)
