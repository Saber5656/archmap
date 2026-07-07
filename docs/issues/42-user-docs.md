# 42 — User documentation, README, SECURITY.md

## Summary

Write the public-facing documentation set: README rewrite (bilingual lead), quickstart, full
configuration reference generated from the schema, the guides index `docs/guides/README.md`,
SECURITY.md, CONTRIBUTING.md, and doc-accuracy tests.

## Context

Final wave-5 issue: everything exists; docs make it adoptable. The README carries the product
thesis (three pillars) and must stay honest about v1 limits (DESIGN §2.2/§2.3).

## Scope

- In: `README.md`, `docs/guides/README.md` (index: ordered links to quickstart, configuration,
  ci-recipes, mcp-clients, faq), `docs/guides/{quickstart.md,configuration.md,faq.md}`,
  `SECURITY.md`, `CONTRIBUTING.md`, doc tests. Existing guides `ci-recipes.md` (40) and
  `mcp-clients.md` (35) are linked and preserved — edited only for link-check fixes.
- Out: website (v2), translations beyond the README Japanese lead paragraph.

## Detailed Requirements

1. `README.md` (English body): tagline (EN + original JP line preserved:
   「コードから腐らないアーキテクチャ図を自動生成する」), 3-pillar pitch table vs
   DeepWiki/deepwiki-open (sourced from research doc, factual tone), 90-second quickstart using
   the DETERMINISTIC path (`npx archmap init --yes && npx archmap generate --no-llm && npx
   archmap serve`) followed by a separate "add prose with a local LLM" step that honestly
   states model-speed-bound duration (DESIGN §15), screenshot/gif placeholder with issue link,
   feature matrix (works with/without LLM columns), security posture summary (local-first,
   egress policy, secret filter — 5 lines linking SECURITY.md), CI badge row (ci + wiki-check
   from 40), v1 limits section (verbatim-derived from DESIGN §2.2), license.
2. `quickstart.md`: prerequisites (Node 22, git repo, optional Ollama with model pull commands),
   the full init→generate→serve→check→mcp walk-through with expected outputs (captured by the
   doctest harness below, trimmed), troubleshooting table (exit codes 2/4/5/6 with fixes — from
   DESIGN §14).
3. `configuration.md`: GENERATED table of every config key (path, type, default, description)
   from the zod schema via `scripts/gen-config-docs.ts` (descriptions come from `.describe()`
   annotations added to the schema — this issue adds them). Because `.describe()` changes the
   exported JSON schema, this issue regenerates and commits `schema/archmap.schema.json`
   together with the docs; the CI drift gate covers BOTH
   (`npm run build && git diff --exit-code schema/archmap.schema.json docs/guides/configuration.md`).
4. `faq.md`: ≥ 10 entries incl.: why is my wiki stale in CI; how do I keep code fully local;
   which Ollama models work; why no prose (`budget-exhausted`, `prose-failed`); monorepo tips
   (include globs); how big a repo can this handle (DESIGN §15 numbers).
5. `SECURITY.md`: supported versions, disclosure process (GitHub private vulnerability
   reporting enabled — instruction), and a B1–B5 guarantee matrix: one row per DESIGN §13.1
   boundary with columns Guarantee / Config knob / Evidence (REDTEAM.md test or 41 gate) /
   Non-goals — covering remote-LLM opt-in, secret filtering, MCP untrusted-content wrapping,
   viewer confinement, and supply-chain gates; links to DESIGN §13, `docs/security/REDTEAM.md`,
   and `npm run test:security`.
6. `CONTRIBUTING.md`: dev setup, test taxonomy (unit/golden/e2e/security + how to update
   goldens legitimately), issue workflow note (docs/ISSUE_PLAN.md is canonical), DCO/sign-off
   not required, code style (lint enforced).
7. Doctest harness (locked contract): every bash block in README/quickstart tagged
   `<!-- doctest -->` runs in CI against a TEMP git-init'd copy of `fixture-ts` using the built
   CLI with deterministic settings (`--no-llm`, `churnDays: 0`); `serve`/`mcp` blocks are
   probed then killed; each block's normalized stdout is compared against the adjacent
   expected-output block (normalization: durations/dates zeroed) — stale or invented outputs
   fail CI. Command allowlist enforced. Plus: config drift gate (req 3) and the relative-link
   checker (`scripts/check-doc-links.ts`).

## Acceptance Criteria

- [ ] Doctested quickstart passes on both OS runners (temp-copy harness; expected-output
      comparison proven to fail on a sabotaged snippet); link checker green across docs/.
- [ ] Config reference + exported JSON schema regenerate byte-identically in CI (joint drift
      gate); reference covers 100% of schema keys (count assert).
- [ ] README pitch claims each traceable to research doc or DESIGN (reviewer checklist in PR);
      no unverifiable superlatives; quickstart timed path uses `--no-llm`.
- [ ] SECURITY.md: B1–B5 matrix rows all present (doc test greps the five boundary ids +
      required links); private reporting path verified on repo settings (note for repo owner
      if permission needed).
- [ ] `docs/guides/README.md` links all five guides in order; FAQ answers reference real
      warning tokens/exit codes (grep test that each cited token exists in source).

## Validation

```bash
npm run test -- docs
node scripts/check-doc-links.ts
```

## Dependencies

41.

## Non-goals

Docs website/HTML publishing, Japanese full translation (v2 candidate), blog/launch material.

## Design References

- DESIGN §2 (scope honesty), §14 (exit codes), §17; research doc (pitch facts)
