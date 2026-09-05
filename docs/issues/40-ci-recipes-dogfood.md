# 40 — CI recipes and dogfooding job

## Summary

Ship the user-facing CI recipe for the freshness gate and wire archmap's own dogfood job: this
repository generates its own wiki (`--no-llm`) and `archmap check` gates every PR.

## Context

Pillar P1 is only credible if archmap itself lives under its own freshness contract (DESIGN §16
row 6). The recipe doubles as documentation users copy into their repos.

## Scope

- In: `.github/workflows/wiki-check.yml` (dogfood), `docs/guides/ci-recipes.md`,
  `examples/github-action-check.yml` (copy-paste template), archmap.config.json for this repo,
  committed `docs/wiki/` baseline for this repo.
- Out: publishing a marketplace Action (v2), release workflow (41).

## Detailed Requirements

1. This repo's config: `archmap.config.json` with `llm.enabled: false` (CI-deterministic wiki),
   `output.dir: "docs/wiki"`, excludes for `docs/issues`, `tests/golden`. Generate and commit the
   baseline wiki as part of this issue.
2. Dogfood workflow `wiki-check.yml` — exact PR contract: on `pull_request` (never
   `pull_request_target`), `permissions: {contents: read}`, third-party actions pinned by FULL
   commit SHA, zero secret-bearing env vars: checkout → setup Node → `npm ci && npm run build`
   → step A `node dist/cli/index.js check --json` (exit 3 → fail with a step summary listing
   stale pages parsed into `$GITHUB_STEP_SUMMARY` + remediation line) → step B
   `node dist/cli/index.js generate --no-llm && git diff --exit-code -- docs/wiki` (proves the
   committed wiki is byte-reproducible; a diff fails the job).
3. Churn determinism (locked): this repo's dogfood config sets `analysis.churnDays: 0`, which
   disables churn entirely (the `0 = disabled` semantic is implemented by issues 03/15; rank
   uses pagerank + entrypoints only). The guide states: freshness hashes `analysis.churnDays`
   (the config value), never observed git history; CI clone depth therefore cannot dirty a
   wiki, but rank-affecting churn requires equal-depth clones — recommending `churnDays: 0`
   for repos that want byte-reproducible CI generation.
4. `docs/guides/ci-recipes.md`: sections — GitHub Actions (the template below), GitLab CI
   equivalent snippet, pre-commit hook snippet that PROPAGATES failure:
   `archmap check || { s=$?; echo "run: archmap generate --no-llm && git add docs/wiki"; exit $s; }`,
   incremental-regen guidance, and "LLM in CI" posture: NOT recommended in v1 (endpoint
   locality); structural mode is the CI mode.
5. Template `examples/github-action-check.yml`: minimal, self-contained, comments in English,
   no repo-specific paths, same security posture as req 2 (pull_request, contents: read, full
   SHA pins), archmap invoked as `npx archmap@<exact-version>` with an EXACT-version
   placeholder and a NOTE to substitute (never `^`/`~` ranges). Runnable validation of this
   template against a packed tarball happens in issue 41's pack smoke; this issue gates it with
   `actionlint` + a pinning/permissions lint (grep-based check in tests).
6. Badge: add wiki-freshness workflow badge to README (README content itself is 42's issue;
   badge line only).

## Acceptance Criteria

- [ ] Dogfood: PR with a source change but stale committed wiki FAILS step A with the stale
      table in the step summary; regenerating + committing turns it green; a hand-tampered
      committed wiki page passes step A but FAILS step B's `git diff` gate (both demonstrated
      on real PRs in this repo — links recorded in the issue on close).
- [ ] This repo's `docs/wiki/` committed, fresh, and byte-reproducible on CI (two runs, with
      the dogfood config's `churnDays: 0`).
- [ ] Both workflow files pass `actionlint` AND the pinning/permissions lint (unpinned action
      or write permission → red).
- [ ] Guide states the churn/freshness rule (config value hashed, git history never) and
      LLM-in-CI posture; pre-commit snippet propagates non-zero exit (shell test).

## Validation

```bash
node dist/cli/index.js generate --no-llm && node dist/cli/index.js check
node dist/cli/index.js generate --no-llm && git diff --exit-code -- docs/wiki
actionlint .github/workflows/wiki-check.yml examples/github-action-check.yml
```

## Dependencies

26, 27.

## Non-goals

Marketplace Action packaging, LLM-backed CI generation, auto-commit bots (v2: wiki-diff PR bot).

## Design References

- DESIGN §3.2 (workflow 2), §16 (dogfood row), §9
