# 09 — web-tree-sitter host, vendored grammars, parse cache

## Summary

Implement the parsing host: initialize `web-tree-sitter`, load vendored `.wasm` grammars for
typescript/tsx/javascript/python, expose a `parseFile` API, and cache parse-derived facts keyed by
`(fileHash, grammarVersion)` per ADR-004.

## Context

Extractors (10, 11) need syntax trees without native-binding install pain. Grammars are vendored
binaries with recorded provenance; the cache neutralizes WASM's speed penalty on warm runs.

## Scope

- In: `src/core/parse/host.ts`, `src/core/parse/cache.ts`, `vendor/grammars/` (wasm files +
  `PROVENANCE.md`), `scripts/build-grammars.md` (documented repeatable build steps).
- Out: language-specific extraction logic (10–12).

## Detailed Requirements

1. Vendor grammars: `tree-sitter-typescript` (emits `typescript.wasm` + `tsx.wasm`),
   `tree-sitter-javascript`, `tree-sitter-python`, built with `tree-sitter build --wasm` from
   pinned upstream release tags. Commit binaries under `vendor/grammars/` with
   `PROVENANCE.md` recording: upstream repo, tag, commit sha, build command, tree-sitter CLI
   version, sha256 of each wasm. A unit test asserts recorded sha256 == actual file hash.
2. `host.ts` — parse dialect is EXPLICIT and distinct from the serialized FileFacts `language`:
   - `type ParseLanguage = "typescript" | "tsx" | "javascript" | "python"`; the caller derives it
     from the file extension (`.tsx` → `tsx`, `.ts/.mts/.cts` → `typescript`, `.js/.jsx/.mjs/
     .cjs` → `javascript`, `.py` → `python`) — helper `parseLanguageFor(path)` exported here.
   - `initParsers(langs: ParseLanguage[], opts: {maxBytes: number}): Promise<ParserHost>` —
     `Parser.init()` once, load only requested grammars; `GRAMMAR_VERSIONS` const map covering
     all four dialects (`{typescript: "<tag>", tsx: "<tag>", …}`) exported.
   - `parserHost.withTree(parseLanguage, sourceText, fn: (tree, hasErrors) => T): T` — parses,
     invokes `fn`, and calls `tree.delete()` in `finally` (wasm heap hygiene; the tree never
     escapes the callback). Text larger than `opts.maxBytes` → typed refusal error (callers
     pre-filter via inventory flags; this is defense in depth). Parse errors never throw —
     tree-sitter ERROR nodes are surfaced via `hasErrors`.
   - Concurrency is the CALLER's concern (pipeline, issue 25); the host is synchronous per call.
3. `cache.ts`: cache stores **extracted FileFacts JSON** (not trees). Concrete API:
   ```ts
   createParseCache({cacheDir: string, logger}): ParseCache   // cacheDir = <repoRoot>/.archmap/cache/parse
   parseCache.getOrCompute(
     file: {path: string, hash: string},
     versions: {grammarVersion: string, extractorVersion: string},
     compute: () => Promise<FileFacts>): Promise<FileFacts>
   ```
   Key = `sha256(file.hash + versions.grammarVersion + versions.extractorVersion)`; storage
   `<cacheDir>/<first2>/<key>.json` (canonical JSON); corrupted/unparseable cache entry →
   recompute + overwrite + debug log (never crash). Fallback extractor (issue 12) passes
   `grammarVersion: "none"`.
4. Cache maintenance: `pruneParseCache(cacheDir, keepKeys: Set<string>)` deletes entries not in
   the current run's keyset when cache dir exceeds 200 MB (called by generate, issue 25).
5. Dependency pinning: `web-tree-sitter` added to package.json at an EXACT version (no
   caret/range — DESIGN §13.6 rule for native/WASM payloads); package.json/lockfile changes are
   in scope for this issue; a test asserts the manifest entry has no range prefix.
6. Node/npm packaging note: wasm files must load from the package install dir via
   `new URL("../vendor/grammars/…", import.meta.url)` — no cwd assumptions (verified by a test
   spawning from another cwd, and again in issue 41's pack test).

## Acceptance Criteria

- [ ] Parses a TS, TSX (JSX syntax — proves the tsx grammar is selected), JS, and PY fixture
      each with zero ERROR nodes via `withTree`; a broken file reports `hasErrors=true` without
      throwing; over-`maxBytes` input raises the typed refusal.
- [ ] `parseLanguageFor` maps all listed extensions correctly (unit matrix).
- [ ] `PROVENANCE.md` sha256 entries match vendored files (test); `web-tree-sitter` pinned exact
      in package.json (test).
- [ ] Cache: second `getOrCompute` with same inputs does not invoke `compute` (spy); changed
      `extractorVersion` or `grammarVersion` busts it; corrupted JSON recovers by recompute.
- [ ] `pruneParseCache`: over-threshold cache keeps `keepKeys` entries, deletes others, ignores
      foreign files; under-threshold cache is untouched.
- [ ] Works when invoked from a different cwd than the package root.

## Validation

```bash
npm run test -- core/parse/host core/parse/cache
```

## Dependencies

01, 05.

## Non-goals

Symbol extraction (10–12); grammar lazy-download (known unknown, only if size becomes a problem).

## Design References

- DESIGN §4.1, §7.2 (facts are what's cached); ADR-004
