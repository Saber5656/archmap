# 12 — Fallback extractor for other languages

## Summary

Produce minimal `FileFacts` for analyzable text files outside the deep-analysis languages: line
count, leading comment/doc line, and best-effort import/include references via a small regex pack,
so unsupported languages still appear in the graph and wiki instead of vanishing.

## Context

DESIGN §2.1 goal 2 promises "structural fallback for all other text files". The wiki must not
misrepresent a polyglot repo as TS/Py-only; fallback files contribute nodes (and cheap edges when
patterns allow) to the module graph.

## Scope

- In: `src/core/parse/extract/fallback.ts` with per-family regex packs.
- Out: tree-sitter parsing for these languages (v2), resolution (13).

## Detailed Requirements

1. Export `FALLBACK_EXTRACTOR_VERSION = "1"`. API mirrors issues 10/11
   (`extractFallbackFacts({path, hash, language, sourceText}): FileFacts` — pure, no fs reads;
   content arrives via the issue-08 `readAnalyzable` accessor, so `secret-excluded` files never
   reach it and `redact`-policy content is already redacted).
   Eligibility (locked): every inventory entry that gets FileFacts but cannot be deep-parsed —
   i.e. `language: "other"` entries AND deep-language files flagged `too-large` (their content
   is truncated to the first `analysis.maxFileKb` bytes for doc/import extraction; `loc` from
   full line count via the accessor content). `generated`-flagged files are processed normally.
   Never `binary`, never `secret-excluded`.
2. Facts produced: `loc`; file-level `doc` (shared schema field) = first comment line within the
   first 10 lines (comment prefixes tried: `//`, `#`, `--`, `/*`, `;`), max 200 chars;
   `symbols: []` (empty; no fake symbols); imports per regex pack.
3. Regex pack v1 (family detected by extension; unlisted extensions → no imports). Locked kind
   rule: relative-style references → `internal`; everything else → `unknown` (never `external`
   at this stage — the resolver finalizes):
   - Go (`.go`): `import "x"` and grouped import blocks → raw `x`, kind `unknown` (no go.mod
     logic in v1).
   - Rust (`.rs`): `use crate::a::b`, `mod name;` → raw path, kind `unknown`.
   - Java/Kotlin (`.java`, `.kt`): `import a.b.C;` → kind `unknown`.
   - Ruby (`.rb`): `require "x"` → `unknown`, `require_relative "x"` → `internal`.
   - Shell (`.sh`, `.bash`): `source x` / `. x` with `./`/`../` → `internal`, else `unknown`.
   - C/C++ (`.c`, `.h`, `.cpp`, `.hpp`): `#include "x"` → `internal`, `#include <x>` → `unknown`.
4. Resolver contract (issue 13): for fallback files it resolves ONLY `internal` relative paths
   (normalized join + inventory-membership check); `unknown` entries become `external` when
   bare-package-shaped or land in `unresolved` — no fabricated edges.
5. Determinism + cache participation identical to issues 10/11 (issue 09 cache with
   `grammarVersion: "none"` + `FALLBACK_EXTRACTOR_VERSION`).
6. Fixture `tests/fixtures/extract-fallback/` with one file per family + one oversized `.ts`
   file (proving too-large fallback), golden JSON.

## Acceptance Criteria

- [ ] Golden test byte-exact across all families, including the oversized `.ts` fallback case
      (truncated import scan, full loc).
- [ ] A `.go` + `.rb` + `.sh` fixture repo produces graph nodes with edges only where relative
      resolution succeeded (asserted in 13's tests; here: facts contain the raw imports).
- [ ] Kind matrix: `require_relative` internal, Go import unknown, `#include <x>` unknown —
      no `external` emitted by this extractor.
- [ ] No symbols fabricated; `doc` extraction respects the 10-line window.
- [ ] Unknown text extension (`.xyz`) yields facts with `loc` + empty imports, no crash.
- [ ] Purity: no fs reads (spy); redacted content passes through unaltered.

## Validation

```bash
npm run test -- core/parse/extract/fallback
```

## Dependencies

07 (inventory language classes), 08 (content accessor), 09 (cache host).

## Non-goals

Accurate cross-file resolution for these languages; adding grammars (v2, per ADR-004 §Consequences).

## Design References

- DESIGN §2.1 (goal 2), §7.2; ADR-004
