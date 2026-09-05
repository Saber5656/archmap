# 11 — Python facts extractor

## Summary

Extract `FileFacts` from Python syntax trees: module-level functions/classes/constants, methods,
docstring first lines, `__all__`, and all import statements.

## Context

Python is the second deep-analysis language of v1 (DESIGN §2.1). Same downstream consumers as
issue 10.

## Scope

- In: `src/core/parse/extract/py.ts`, fixtures.
- Out: module path resolution (13).

## Detailed Requirements

1. Export `PY_EXTRACTOR_VERSION = "1"`. API mirrors issue 10 exactly:
   `extractPyFacts({path, hash, language, parseLanguage: "python", sourceText, host}): FileFacts`
   — pure, no fs reads, no logging of source text; uses the shared facts schema from issue 10
   (`src/core/parse/facts-schema.ts`).
2. Symbols (module level + ONE level of class nesting only — deeper nesting is ignored):
   module-level `def` → `function` (async included), module-level `class` → `class`, `def`
   directly inside a module-level class (excluding names starting `_` except `__init__`) →
   `method` named `Class.method`, module-level `NAME = literal` where NAME is UPPER_SNAKE →
   `const`. Classes nested inside classes/functions are NOT emitted (locked simplification).
3. `exported` semantics: if `__all__` is a literal list → symbols listed are `exported: true`,
   others false; if no `__all__` → all non-underscore-prefixed module-level symbols
   `exported: true` (Python convention). Record `hasDunderAll: boolean` in facts extras.
4. `signature` per kind (locked): `function`/`method` = `def` header through `:`; `class` =
   `class` header through `:`; `const` = `NAME = <value preview>` (value source text, single
   line). All max 160 chars, decorators excluded from signature but included in span.
   `doc`: first line of the docstring (function/class), max 200 chars.
5. Imports use the shared schema fields `{raw, names?, deferred?, kind}`:
   `import a.b` → `{raw: "a.b"}`; `import a as x` → `{raw: "a"}`;
   `from a.b import c, d as e` → `{raw: "a.b", names: ["c", "d"]}`;
   `from . import x` → `{raw: ".", names: ["x"]}`; `from ..pkg import y` →
   `{raw: "..pkg", names: ["y"]}` (relative dots preserved in `raw`); star imports →
   `names: ["*"]`. Conditional/inside-function imports included with `deferred: true`.
6. `kind` is PURELY LEXICAL: leading-dot relative → `internal`; every absolute import →
   `unknown`. First-party-root detection and final internal/external classification happen ONLY
   in issue 13 (which must leave no `unknown`).
7. Module docstring first line captured as the file-level `doc` field (shared schema).
8. `hasMainGuard: true` when the module contains a top-level `if __name__ == "__main__":`
   (or `'__main__'`) statement — consumed by issue 14 entrypoint detection; fixture asserts it.
9. Determinism and error handling identical to issue 10 (sorted, `parseErrors: true`).
10. Fixture `tests/fixtures/extract-py/` exercising all rules, golden JSON.

## Acceptance Criteria

- [ ] Golden test passes byte-exact.
- [ ] `__all__` fixture flips export flags correctly; no-`__all__` fixture uses underscore rule.
- [ ] Relative import depth preserved (`from ..a.b import c` → raw `..a.b`, names `["c"]`);
      `from . import x` → raw `.`, names `["x"]`; star import → names `["*"]`.
- [ ] Every absolute import is `unknown`; every relative import is `internal` (kind matrix).
- [ ] Async defs, decorated defs covered; a class nested inside a class emits nothing (locked
      simplification proven); `hasMainGuard` fixture asserts true/false cases.
- [ ] Signature per kind matches the locked formats (function/class/const fixtures).

## Validation

```bash
npm run test -- core/parse/extract/py
```

## Dependencies

09.

## Non-goals

Type stubs, namespace packages beyond first-party root detection (13), dynamic `__all__`.

## Design References

- DESIGN §7.2; §8.2
