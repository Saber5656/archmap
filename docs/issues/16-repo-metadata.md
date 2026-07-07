# 16 — Repo metadata collector

## Summary

Collect repo-level metadata for wiki pages: package manifests (name, version, deps), README
ingest, license detection, and safe git info.

## Context

The overview page (§8.1) needs the repo's self-description; the dependencies page needs declared
dependency tables joined with the graph's observed `externals` (13).

## Scope

- In: `src/core/metadata/manifests.ts`, `readme.ts`, `git-info.ts`, `license.ts`,
  `src/core/metadata/schema.ts` (RepoMetadata zod schema).
- Out: rendering (19), external-usage join (19 does the join).

## Detailed Requirements

1. RepoMetadata contract (locked; canonical JSON; optional = omitted):
   ```ts
   collectRepoMetadata({repoRoot, inventory, readAnalyzable, logger}): Promise<RepoMetadata>
   interface RepoMetadata {
     schemaVersion: 1
     manifests: Array<{ path: string;            // repo-relative POSIX — sort key
       ecosystem: "npm" | "python" | "python-requirements" | "go" | "cargo";
       name?: string; version?: string; description?: string;
       deps: Array<{name: string, range: string}>;   // sorted by name
       workspaces?: string[] }>                      // npm only, raw globs
     readme: { path: string; h1?: string; firstParagraph?: string;
               h2Sections: string[] } | null
     license: { spdxId: string | "unknown"; sourcePath: string } | null
     git: { branch: string | null; shortSha: string | null; commitCount: number | null;
            lastCommitIso: string | null; remoteName: string | null }
     warnings: string[]
   }
   ```
2. `manifests.ts` — discovery scope is REPO ROOT ONLY in v1 (`package.json`, `pyproject.toml`,
   `requirements.txt`, `go.mod`, `Cargo.toml` at `repoRoot`; monorepo sub-manifests are a v2
   item). Content read through `readAnalyzable` (a manifest excluded by the secret filter is
   skipped with warn). Parsers: package.json (name, version, description, deps/devDeps
   name+range, workspaces globs), pyproject ([project] name/version/description/dependencies;
   TOML via a small pinned parser dep), requirements.txt (name + specifier per line, comments
   skipped), go.mod (module path + require lines, regex), Cargo.toml ([package] +
   [dependencies] names). Malformed → warn `manifest-unparseable`, skip.
3. `readme.ts`: locate `README.md` (case-insensitive, repo root) in the inventory; content via
   `readAnalyzable`; extract: first H1 text, first paragraph (≤ 500 chars, cut on a word
   boundary, markdown stripped), H2 titles. Full README text is NOT stored.
4. `git-info.ts` via `execFile` (no shell): branch (`null` when detached), HEAD short sha,
   commit count, latest commit ISO date. `remoteName` selection (locked): upstream remote of
   the current branch → else `origin` when it exists → else first remote by ASCII → else null.
   Repo with no commits → all git fields null except remoteName. NEVER remote URLs (may embed
   credentials; DESIGN §13).
5. `license.ts` precedence (locked): a `LICENSE*` file exists → sniff text against Apache-2.0,
   MIT, BSD-3-Clause, GPL-3.0, MPL-2.0 → `{spdxId, sourcePath}` or
   `{spdxId: "unknown", sourcePath}`; no LICENSE file → `package.json.license` value
   (`sourcePath: "package.json"`); neither → `license: null`.
6. Empty repo yields valid RepoMetadata (`manifests: []`, nulls); never throw on absence.
   Determinism: manifests sorted by path; deps by name; h2Sections in document order.

## Acceptance Criteria

- [ ] Fixture with package.json (with workspaces) + pyproject + requirements + go.mod yields
      all manifests with `path`/`ecosystem`/`workspaces` per the locked schema (golden JSON,
      byte-stable ordering).
- [ ] README extract: H1, first paragraph truncation at 500 chars on word boundary, H2 list.
- [ ] git matrix: normal repo (upstream remote wins), origin fallback, multi-remote ASCII
      fallback, detached HEAD (branch null), zero-commit repo (nulls) — remote URL string
      absent from output in all cases (grep test).
- [ ] License matrix: each of the 5 sniffed licenses, unrecognized LICENSE file → `unknown`
      with sourcePath, package.json fallback, and no-license → null.
- [ ] Empty repo → valid metadata, no throw; malformed pyproject → `manifest-unparseable`
      warning; secret-shaped value in package.json description handled per the secret filter
      (canary absent from RepoMetadata under `exclude`/`redact` — integration test with 08).

## Validation

```bash
npm run test -- core/metadata
```

## Dependencies

07, 08.

## Non-goals

Dependency vulnerability lookups (never in v1 — offline posture), full README semantic parsing.

## Design References

- DESIGN §8.1 (overview/dependencies pages), §13 (no remote URLs in artifacts)
