# 10 — TypeScript/JavaScript facts extractor

## Summary

Extract `FileFacts` (DESIGN §7.2) from TS/TSX/JS syntax trees: top-level symbols with spans,
export status, signatures, first doc line, and all import/export-from specifiers.

## Context

FileFacts feed the module graph (13), wiki API tables and citations (19), and symbol-aligned
chunking (28). Accuracy of spans and export flags directly determines citation quality.

## Scope

- In: `src/core/parse/extract/ts.ts` (+ shared helpers `extract/common.ts`),
  `src/core/parse/facts-schema.ts` (the SHARED FileFacts zod schema used by issues 10/11/12 —
  this issue creates it), fixtures.
- Out: import path resolution (13), Python (11).

## Detailed Requirements

0. `facts-schema.ts` — canonical FileFacts contract (DESIGN §7.2 plus these locked extensions):
   `symbols[].exportedAs?: string`; `parseErrors?: boolean`; file-level `doc?: string` (used by
   fallback/python module docstrings); `imports[].kind: "internal" | "external" | "unknown"`
   (`unknown` = bare specifier pending resolution — issue 13 must leave no `unknown` in the
   final graph inputs); `imports[].names?: string[]`; `imports[].deferred?: boolean`;
   `hasMainGuard?: boolean`. All optional fields omitted when absent (canonical JSON).
1. Export `TS_EXTRACTOR_VERSION = "1"` (bump on behavior change; busts parse cache).
   Extractor API (pure — NO filesystem reads, no logging of source text; content arrives from
   the issue-08 `readAnalyzable` path):
   ```ts
   extractTsFacts({path, hash, language, parseLanguage, sourceText, host}): FileFacts
   ```
   (`host` = issue 09 ParserHost; the extractor calls `withTree(parseLanguage, sourceText, …)`;
   `parseLanguage` comes from `parseLanguageFor(path)` — `.tsx` files parse with the tsx
   grammar while serialized `language` remains `"typescript"`.)
2. Symbols captured (top-level and one nesting level for class members):
   - `function` declarations + arrow/function expressions assigned to top-level `const/let`;
   - `class` declarations; class `method`s (public only, including static) as `kind: "method"`,
     `name: "ClassName.methodName"`;
   - top-level `const` with literal/object/call initializer → `kind: "const"`;
   - `interface` → `interface`, `type` alias → `type`, `enum` → `enum`.
3. For each symbol: `name`, `kind`, `exported` (direct `export`, `export default`, or listed in a
   later `export { x }` / `export { x as y }` — track alias, `name` stays the local name, add
   `exportedAs` when aliased), `span` (1-based start/end line of full declaration incl.
   decorators), `signature` (source text of the header up to body start, single-spaced, max 160
   chars), `doc` (first sentence of an immediately preceding `/** … */` or `//` line, max 200
   chars, stripped of markup).
4. Imports: `import x from`, `import {a, b}`, `import * as ns`, `import "side-effect"`,
   `export … from "m"`, dynamic `import("m")` with a string literal, and `require("m")` calls →
   entries `{raw, resolved: null, kind}` with this PURELY LEXICAL rule: relative (`./`, `../`)
   or root (`/`) specifiers → `"internal"`; ALL bare specifiers (including alias-looking ones
   like `@app/x`, `@/x`, `~/x`) → `"unknown"` — issue 13 tries tsconfig paths on every
   `unknown` before classifying it external. Non-literal dynamic imports are skipped and
   counted in the `skippedDynamicImports?: number` FileFacts field (locked in facts-schema).
5. TSX: same rules via the tsx grammar; components are `function`s.
6. ERROR-containing trees: extract what is parseable; add facts field `parseErrors: true`.
7. Determinism: symbols sorted by `span.startLine`, imports by source order then raw.
8. Fixture: `tests/fixtures/extract-ts/` with a file exercising every rule above (golden FileFacts
   JSON checked in).

## Acceptance Criteria

- [ ] Golden tests: separate fixtures for `.ts`, `.tsx` (JSX component syntax), and `.js` each
      match checked-in FileFacts JSON exactly; a broken-syntax fixture extracts partially with
      `parseErrors: true`.
- [ ] Export detection covers: `export function`, `export default class`, `export const`,
      deferred `export { a as b }` (with `exportedAs`), `export * from` (recorded as import
      edge, no symbol).
- [ ] Lexical kind rule: `./x` internal, `@app/x` unknown, `react` unknown (never `external` at
      this stage).
- [ ] Spans map back to the real source lines (test slices source by span and finds the name).
- [ ] `require` + dynamic import literals captured; template-literal import skipped and counted
      in `skippedDynamicImports`.
- [ ] Purity: extractor performs no fs reads (spy) and content containing a redaction marker
      (`[REDACTED:…]`) flows through into facts unaltered (B2 compliance via issue 08 accessor).
- [ ] A 1000-line fixture extracts in < 150 ms warm (informational assert, not CI-gating).

## Validation

```bash
npm run test -- core/parse/extract/ts
```

## Dependencies

09.

## Non-goals

Type resolution, JSDoc full parsing, decorators semantics, monorepo workspace protocol
resolution (13 handles what it can; rest recorded `unresolved`).

## Design References

- DESIGN §7.2 (FileFacts contract), §8.2 (citations consume spans)
