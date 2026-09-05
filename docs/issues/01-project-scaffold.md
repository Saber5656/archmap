# 01 — Project scaffold: TypeScript ESM, vitest, lint, CI, license

## Summary

Create the initial TypeScript project skeleton for archmap: strict ESM TypeScript, vitest,
eslint + prettier, GitHub Actions CI (lint + typecheck + test on macOS and Linux), Apache-2.0
license, and the source directory layout from DESIGN §4.1.

## Context

archmap is a local-first CLI tool distributed via npm (`npx archmap`). Every later issue assumes
this scaffold: strict typing, deterministic builds, and CI that runs on macOS and Linux (DESIGN
§2.1 goal 1, §16). The repository currently contains only `README.md` and `docs/`.

## Scope

- In: package.json + committed `package-lock.json`, tsconfig, eslint/prettier config, vitest
  config, `src/`+`tests/` skeleton directories with placeholder modules, CI workflow, LICENSE,
  NOTICE, .gitignore, .editorconfig.
- Out: any CLI behavior (issue 02), config schema (issue 03), README product rewrite (issue 42).

## Detailed Requirements

1. `package.json`:
   - `"name": "archmap"`, `"version": "0.1.0-dev"`, `"type": "module"`, `"license": "Apache-2.0"`,
     `"engines": { "node": ">=22" }`, `"bin": { "archmap": "./dist/cli/index.js" }`.
   - Scripts: `build` (tsc), `test` (vitest run), `test:watch`, `lint` (eslint + prettier check),
     `typecheck` (tsc --noEmit), `format`.
   - No `postinstall`/`prepare` scripts that execute code (DESIGN §13.6).
2. `tsconfig.json`: `strict: true`, `module: "nodenext"`, `moduleResolution: "nodenext"`,
   `target: "es2023"`, `outDir: "dist"`, `rootDir: "src"`, `noUncheckedIndexedAccess: true`,
   `exactOptionalPropertyTypes: true`.
3. Directory skeleton following DESIGN §4.1. Create exactly these tracked files:
   - `src/cli/index.ts` (bin entry placeholder printing nothing, exit 0) and one `index.ts`
     placeholder (empty `export {}`) in each of: `src/config`, `src/shared`, `src/core/scan`,
     `src/core/secrets`, `src/core/parse`, `src/core/graph`, `src/core/rank`,
     `src/core/metadata`, `src/wiki/plan`, `src/wiki/render`, `src/wiki/manifest`,
     `src/llm`, `src/llm/prompts`, `src/llm/synthesize`, `src/ask`, `src/mcp`, `src/viewer`.
   - `.gitkeep` in `tests/fixtures/`, `tests/golden/`, `schema/`.
   - The concrete module files named in DESIGN §4.1 (e.g. `src/llm/client.ts`,
     `src/ask/chunk.ts`) are created by their owning issues, NOT by this scaffold.
4. `package-lock.json` generated with npm on Node 22 and committed (DESIGN §13.6 lockfile
   requirement; `npm ci` in CI depends on it).
5. eslint flat config with `typescript-eslint` recommended-type-checked; prettier with default
   options + `printWidth: 100`; both wired into `npm run lint`.
6. vitest: node environment, coverage via v8 (`npm run test -- --coverage` works); one sample test
   (`tests/smoke.test.ts`) asserting `1 + 1 === 2` to prove the harness.
7. CI: `.github/workflows/ci.yml` — trigger on `pull_request` and push to `main`; matrix
   `os: [ubuntu-latest, macos-latest]`, Node 22; steps: checkout → setup-node (npm cache) →
   `npm ci` → `npm run lint` → `npm run typecheck` → `npm run test`. Pin all actions by major tag.
8. `LICENSE` = Apache-2.0 text (copyright: "archmap contributors"); `NOTICE` file naming the
   project.
9. `.gitignore`: `node_modules/`, `dist/`, `.archmap/`, coverage output. `.editorconfig`: LF,
   final newline, 2-space indent, UTF-8.

## Acceptance Criteria

- [ ] `npm ci && npm run lint && npm run typecheck && npm run test && npm run build` all succeed
      locally on Node 22.
- [ ] CI workflow passes on both OS runners for a PR.
- [ ] `git ls-files` shows every tracked file enumerated in requirement 3, plus
      `package-lock.json`, LICENSE, NOTICE; `npm ci` succeeds from a clean checkout.
- [ ] package.json has no lifecycle script that executes code on install.
- [ ] Repository has zero eslint/prettier violations.

## Validation

```bash
npm ci
npm run lint && npm run typecheck && npm run test && npm run build
node dist/cli/index.js  # may print a placeholder line; must exit 0
```

## Dependencies

None (first issue).

## Non-goals

Command parsing, config, packaging polish (`files` whitelist etc. — issue 41).

## Design References

- DESIGN §4.1 (source layout), §13.6 (supply chain), §16 (testing), §17 (packaging)
