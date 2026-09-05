# 41 — npm packaging and release pipeline

## Summary

Make `npx archmap` real: package `files` whitelist (dist, wasm grammars, viewer assets, schema),
pack smoke tests, the manually-triggered release workflow with npm provenance, dependency audit
gates, and the pre-release security checklist.

## Context

DESIGN §17 + §13.6. Distribution is where supply-chain promises become real; the release itself
remains a human gate (merge ≠ release per repo policy).

## Scope

- In: package.json finalization, `.github/workflows/release.yml`, `scripts/pack-smoke.sh`,
  `docs/RELEASING.md` (checklist), Dependabot config, CHANGELOG.md bootstrap.
- Out: user docs (42), marketing/registry listings.

## Detailed Requirements

1. package.json: `files: ["dist", "vendor/grammars", "dist/viewer-assets", "schema",
   "README.md", "LICENSE", "NOTICE"]`; `exports` map for CLI-only surface (no public API export
   in v1 — `"."` maps to a stub that throws with a helpful message; bin is the product);
   `engines.node >= 22`; `publishConfig: {access: "public", provenance: true}`; every direct
   runtime dependency justified in RELEASING.md's dependency table (count gate in req 4).
2. Pack smoke (`pack-smoke.sh`, run in CI on both OS) — exact ordered contract:
   1. `npm pack` → note tarball path; 2. `mkdir tmp && cd tmp && npm init -y && npm i
   <tarball>`; 3. copy `fixture-ts` into a fresh temp git repo; 4. run there, asserting exit
   codes: `npx --no-install archmap init --yes` (0), `npx --no-install archmap generate
   --no-llm --json` (0), `npx --no-install archmap check --json` (0); 5. start
   `npx --no-install archmap serve --json &`, poll `/api/health` until ok (≤ 10 s), SIGINT and
   assert exit 0; 6. run `scripts/mcp-smoke.mjs` pointed at the INSTALLED bin; 7. execute the
   command sequence from `examples/github-action-check.yml` (issue 40) against the tarball
   install (runnable-template proof deferred from 40); 8. assert wasm + viewer assets loaded
   from the installed location (issue 09/37 path rules).
   Package size gate: tarball ≤ 15 MB (fails loudly with breakdown by folder).
3. `release.yml`: `workflow_dispatch` with inputs `version` (string) and `dry_run` (boolean,
   default true) → guards: ref is `main`, `version` equals package.json `version` → the
   workflow runs the FULL gate suite itself (`npm ci && npm run build && npm test && npm run
   test:security && bash scripts/pack-smoke.sh`) — this self-contained suite IS the release
   gate (no external check-run querying) → `dry_run: true` → `npm publish --dry-run
   --provenance`, NO tag, NO GitHub Release; `dry_run: false` → `npm publish --provenance`
   (OIDC trusted publishing implemented; NPM_TOKEN fallback documented) + `git tag v<version>`
   + GitHub Release with CHANGELOG extract.
   Version contract: this issue KEEPS package.json at `0.1.0-dev` and validates the workflow
   with a real `dry_run: true` dispatch (`version: 0.1.0-dev`); the human release PR
   (RELEASING.md step 1) later bumps version + CHANGELOG to `0.1.0`. NO auto-version-bump.
4. Dependency hygiene: `dependabot.yml` (npm weekly, grouped minor/patch); CI `npm audit
   --omit=dev --audit-level=high` gate; `lockfile-lint` step (registry allowlist).
   B5 package-security tests (CI): package.json has NO `preinstall`/`install`/`postinstall`/
   `prepare` scripts that execute code on install; `better-sqlite3` and `web-tree-sitter`
   pinned EXACT (no range prefix); direct `dependencies` count ≤ 12 (hard gate over
   package.json `dependencies` only, devDependencies excluded) — exceeding requires raising
   the constant in the same PR as a documented exception row in RELEASING.md's dependency
   table.
5. `docs/RELEASING.md` checklist: version+CHANGELOG PR → CI+security suites green → dogfood wiki
   fresh → **repo history secret scan** (`gitleaks detect` command given; per repo-owner policy
   run before any public release) → dispatch release → verify provenance badge on npm → smoke
   `npx archmap@<version>` on a clean machine.
6. CHANGELOG.md: keep-a-changelog format, `0.1.0` section seeded from ISSUE_PLAN waves.

## Acceptance Criteria

- [ ] Pack smoke green on macOS + Linux CI following the exact 8-step contract; tarball size
      within gate and reported in job summary.
- [ ] `npx` flow from tarball works in a repo whose cwd ≠ package dir (wasm/assets path proof);
      40's example-workflow command sequence runs green against the tarball.
- [ ] Release workflow: a real `dry_run: true` dispatch on main executes ALL gates and
      `npm publish --dry-run --provenance` without tagging; the `dry_run: false` branch is
      code-reviewed as publish-capable (guards + publish + tag + release steps present).
- [ ] B5 tests: lifecycle-script absence, exact pins for the two native/WASM deps, direct-deps
      count gate — each red when violated (mutation-checked once, documented).
- [ ] `npm audit` gate red when a known-vuln dep is injected in a test branch (documented trial),
      green on main.
- [ ] RELEASING.md checklist complete incl. gitleaks command and the version-bump step;
      dependency table lists every runtime dep with justification.

## Validation

```bash
bash scripts/pack-smoke.sh
npm publish --dry-run
```

## Dependencies

39, 40.

## Non-goals

Actual first publish (human decision), Homebrew/binaries (v2), API surface export.

## Design References

- DESIGN §17, §13.6 (B5); repo policy: merge ≠ release
