# ADR-004: web-tree-sitter (WASM) with vendored grammars

- Status: accepted
- Date: 2026-07-07

## Context

Symbol/import extraction needs real parsers. Options: native `tree-sitter` Node bindings
(node-gyp, ABI breakage across Node versions, painful `npx` installs), `web-tree-sitter` (WASM,
portable, ~2-4x slower), per-language compilers (tsc/Python ast — heavyweight, uneven), or
regex-only (inaccurate).

## Decision

1. Use `web-tree-sitter` at an exactly pinned version; load vendored `.wasm` grammar files for
   `typescript`, `tsx`, `javascript`, `python` shipped inside the npm package (`files` whitelist).
2. Grammar `.wasm` binaries are built in a documented, repeatable step (`tree-sitter build --wasm`
   from pinned grammar repo tags) and checked into the repo with recorded source tag + sha256.
3. Parse results are cached in `.archmap/cache/parse/` keyed by `(fileHash, grammarVersion)`, so
   WASM's speed penalty applies only to changed files.
4. Non-vendored languages use the fallback extractor (regex imports + file facts) — adding a
   language later means vendoring one more grammar + one extractor module.

## Consequences

- `npx archmap` works with zero compile step on macOS/Linux/Windows; no node-gyp support burden.
- ~MBs of wasm in the package; acceptable (documented in package size budget, issue 41).
- Slower cold parse than native; mitigated by cache; performance targets in DESIGN §15 account for it.

## Alternatives rejected

- Native bindings: install fragility contradicts the "npx and go" distribution goal.
- tsc/ast-based per-language analysis: 10x scope for marginal v1 gain; revisit for type-aware
  features in v2.
