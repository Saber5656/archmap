# 07 — File scanner and inventory

## Summary

Implement the deterministic file scanner producing `FileInventory` (DESIGN §7.1): walk the repo,
honor `.gitignore` + `.archmapignore` + config include/exclude + built-in defaults, detect
language, flag oversized/binary/generated files, and hash contents.

## Context

The inventory is the root input of the pipeline and of `check`'s re-scan (DESIGN §9 step 2). It
must be byte-stable across runs and platforms (ordering, hashing) per DESIGN §5.

## Scope

- In: `src/core/scan/walker.ts`, `src/core/scan/ignore.ts`, `src/core/scan/language.ts`,
  `src/core/scan/inventory.ts`, FileInventory zod schema.
- Out: secret detection (08 consumes inventory), parsing (09+).

## Detailed Requirements

1. Walk from `repoRoot` using our own recursive readdir (no globby dependency needed): skip
   `.git/`, symlinks (never followed — flag-free skip with debug log), FIFOs/sockets.
2. Ignore layering — a file is scanned only if it survives every layer, evaluated in this order:
   1. Built-in absolute denials (cannot be re-included by any later rule): `.git/`,
      `node_modules/`, `.archmap/`, and the `config.output.dir` subtree.
   2. `.gitignore` (root + nested, standard gitignore semantics via the `ignore` npm package:
      within a layer, later patterns and `!` negations apply as git does).
   3. `.archmapignore` (repo root only, same pattern syntax incl. negation).
   4. `analysis.exclude` globs (no negation support — plain deny list).
   5. `analysis.include` (when non-empty: allowlist filter applied last).
   All matching is against repo-relative POSIX paths. Files omitted by layers 1–5 do not appear
   in the inventory at all.
3. Built-in sensitive-name exclusions independent of `security.secretScan` (DESIGN §13.3):
   `.env*`, `*.pem`, `*.key`, `id_*` (any SSH-key-style name), `*.p12`, `*.pfx`. These apply to
   files that SURVIVED the ignore layers (a gitignored `.env` is simply absent; a tracked one is
   listed): the entry is kept in the inventory with flag `secret-excluded`, `size` present, and
   `hash` omitted (never read beyond stat).
4. Language detection by extension map: `.ts/.mts/.cts→typescript`, `.tsx→typescript`,
   `.js/.mjs/.cjs/.jsx→javascript`, `.py→python`, known-text extensions → `other`, everything
   else sniffed: first 8 KB containing NUL byte → `binary`.
5. Flags: `too-large` (size > `analysis.maxFileKb`·1024 — still hashed, excluded from deep parse),
   `generated` (first 3 lines match `/generated|do not edit/i` or path matches
   `*.generated.*|*_pb2.py|*.min.js`), `secret-excluded` (rule 3; issue 08 adds content-based ones).
6. Output `FileInventory` per DESIGN §7.1 with this exact zod contract: files sorted by path
   ASCII; `hash` present for every entry EXCEPT `secret-excluded` ones (schema: `hash` optional,
   required-unless-flagged refinement); binary files get `language: "binary"` and are hashed.
   `stats` definition (locked): `fileCount` = `files.length` (all entries incl. flagged);
   `totalBytes` = sum of `size` over all entries; `analyzedCount` = entries that will receive
   FileFacts = non-`binary`, non-`secret-excluded` (note: `too-large` and `generated` files DO
   count — they receive fallback facts per issue 12).
7. Concurrency: hash with a worker pool of `min(8, cpus)`; results order-independent then sorted.
8. API: `scan(config, repoRoot, logger): Promise<FileInventory>`; pure (no writes).
9. Fixture: create `tests/fixtures/scan-basic/` in THIS issue (nested `.gitignore` with
   negation, `.archmapignore`, node_modules, NUL binary, oversized file, tracked `.env`,
   `id_ecdsa`, generated file) + its golden inventory JSON. (Product-level `fixture-mixed`
   goldens are issue 27.)

## Acceptance Criteria

- [ ] On `scan-basic`: inventory matches its checked-in golden JSON byte-for-byte (canonical
      serialization), including a `!`-negated re-include, the tracked `.env` and `id_ecdsa` as
      `secret-excluded` entries without hash, and stats matching the locked definitions.
- [ ] A gitignored `.env` is absent entirely; a `!`-negation cannot re-include anything under a
      built-in absolute denial (test with `!node_modules/x`).
- [ ] Symlink pointing outside the repo is skipped and never read.
- [ ] Same inventory hash across two consecutive runs and across macOS/Linux CI.
- [ ] `analysis.include: ["src/**"]` restricts files to `src/` while stats still report
      `fileCount` of included set only.
- [ ] `output.dir` contents are absent from the inventory even when not gitignored.

## Validation

```bash
npm run test -- core/scan
```

## Dependencies

03, 05.

## Non-goals

Content-based secret scanning (08), git-tracked-only mode (scanner sees working tree; documented).

## Design References

- DESIGN §7.1 (schema), §5 (determinism), §13.3 (name-based exclusions)
