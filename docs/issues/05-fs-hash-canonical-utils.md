# 05 — Hashing, canonical JSON, atomic FS utils

## Summary

Implement the determinism toolbox every artifact depends on: sha256 content hashing, canonical
JSON serialization, atomic directory swap writes, and repo-relative POSIX path utilities.

## Context

DESIGN §5 requires byte-stable generated output (golden tests diff it) and atomic wiki writes.
All schemas (§7) use `sha256:<hex>` hashes and repo-relative POSIX paths, including on Windows.

## Scope

- In: `src/shared/hash.ts`, `src/shared/canonical-json.ts`, `src/shared/atomic-fs.ts`,
  `src/shared/paths.ts`.
- Out: any schema definitions (live with their subsystems).

## Detailed Requirements

1. `hash.ts`:
   - `hashBytes(buf: Uint8Array): string` → `"sha256:" + hex` (node:crypto).
   - `hashFile(absPath): Promise<string>` — streaming, no full-file buffering above 1 MB.
   - `hashJson(value): string` — hash of canonical serialization (see 2).
2. `canonical-json.ts`: `canonicalStringify(value, {pretty?: boolean}): string` —
   recursively sorted object keys (ASCII order), arrays preserved, numbers via `JSON.stringify`
   semantics, `undefined` properties dropped, LF line endings, trailing newline when `pretty`.
   Rejects `NaN`/`Infinity`/`bigint`/functions with a typed error. `pretty: true` = 2-space indent
   (used for committed artifacts), default compact (used for hashing).
3. `atomic-fs.ts`:
   - `writeFileAtomic(absPath, content)` — write to `absPath + ".tmp-" + pid`, fsync, rename.
   - `swapDirAtomic(finalDir, buildFn, opts?: {onWarn?: (msg: string) => void})` — creates
     `finalDir + ".tmp-" + pid`, calls `buildFn(tmpDir)` to populate, then: if `finalDir` exists
     rename it to `finalDir + ".old"`, rename tmp → final, delete `.old`. On any failure: remove
     tmp, restore `.old` if the swap half-completed, rethrow. Guarantees `finalDir` is never
     observed partially written.
   - Crash recovery on next invocation, scoped STRICTLY to names derived from `finalDir`'s
     basename (`<base>.tmp-*`, `<base>.old` — never a parent-wide glob): if `finalDir` is
     missing and `<base>.old` exists, restore `.old` → `finalDir` FIRST (it is the only intact
     previous output), then delete leftover `<base>.tmp-*`; each action reported via `onWarn`.
     Unrelated sibling `*.old`/`*.tmp-*` directories are never touched.
4. `paths.ts`:
   - `toRepoRelative(repoRoot, absPath): string` — normalized, POSIX separators, no leading `./`;
     throws typed error if outside `repoRoot` (used for confinement checks).
   - `insideRepo(repoRoot, absPath): boolean`; `joinConfined(rootAbs, ...segments): string`.
   - Confinement algorithm (all three functions): `path.resolve` both sides, then check via
     `path.relative(root, target)` — inside iff the result is `""` or neither starts with
     `".."` nor is absolute. Naive `startsWith` prefix checks are forbidden (sibling-prefix
     escape: root `/repo` vs `/repo2/x`). Symlinks are NOT resolved here — callers that serve
     files must `realpath` before confinement (JSDoc states this; issues 34/36 rely on it).
5. Typed errors are local to this module (`PathEscapeError`, `CanonicalizeError`) since the
   shared taxonomy (issue 02) may not exist yet; callers wrap them into `ArchmapError` later.
   All functions synchronous-friendly where cheap; no dependencies beyond node builtins.

## Acceptance Criteria

- [ ] Canonical JSON: `{b:1,a:{d:2,c:3}}` serializes with sorted keys; hash equal across runs and
      across object key insertion orders; property-based test with random objects.
- [ ] `swapDirAtomic` under injected failure (buildFn throws / rename fails) leaves the previous
      `finalDir` intact byte-for-byte; success replaces contents completely (removed files gone).
- [ ] Crash-state recovery: fixture state "final missing + `<base>.old` present" restores the
      old dir before cleanup; unrelated sibling `other.old` untouched; leftover `<base>.tmp-*`
      removed with `onWarn` called.
- [ ] Confinement corpus: `joinConfined("/repo", "a", "../../etc")` throws; sibling-prefix
      (`root=/repo`, `target=/repo2/x`) is OUTSIDE; root itself is inside; absolute segment
      throws; `toRepoRelative` outside-path throws; `insideRepo` matrix matches.
- [ ] Streaming `hashFile` result equals `hashBytes` of full content for a 5 MB fixture.

## Validation

```bash
npm run test -- shared/hash shared/canonical shared/atomic shared/paths
```

## Dependencies

01.

## Non-goals

Manifest/schema content (07+, 20), symlink policy for the scanner (07 defines it).

## Design References

- DESIGN §5 (storage rules, atomic swap, determinism), §7 (hash/path formats), §13.5
