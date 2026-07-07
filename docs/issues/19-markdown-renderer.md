# 19 — Markdown renderer, citations, nav

## Summary

Render all wiki pages from plan + analysis artifacts: deterministic Markdown templates per page
kind, mechanical citations, prose slots with markers, front-matter, `_meta/nav.json`, and the
localized string table.

## Context

This is the artifact users commit (ADR-002) and agents consume (34). Every deterministic section
of DESIGN §8.1 is produced here; prose slots stay empty-marked until 23/24 fill them.

## Scope

- In: `src/wiki/render/templates/{overview,architecture,dependencies,module}.ts`,
  `src/wiki/render/citations.ts`, `src/wiki/render/prose-slots.ts`, `src/wiki/render/nav.ts`,
  `src/wiki/render/strings/{en,ja}.ts`, `src/wiki/render/render.ts` (assembly).
- Out: manifest write (20), atomic output swap (25 calls it), prose content (23/24).

## Detailed Requirements

1. Templates produce the exact deterministic sections of DESIGN §8.1 per page kind. All table
   headers/fixed strings via the string table keyed by `output.language` (`en`, `ja` complete;
   unknown language → `en` with warn `language-fallback`).
2. Per-section table contract (locked; columns | row cap | sort | empty state):
   - overview stats: `Files/Analyzed/LOC/Languages` | 1 row | — | never empty.
   - overview top clusters: `Cluster/Files/LOC/Score` | 10 | rank comparator | `(no clusters)`.
   - architecture cluster table: `Cluster/Files/LOC/Fan-in/Fan-out/One-liner` | all module
     pages | rank comparator | `(no clusters)`; `plan.omittedClusters` listed beneath.
   - architecture entrypoints: `File/Reason` | 15 | path ASC | `(none detected)`.
   - dependencies declared: per ecosystem `Package/Range/Importers` | 50/ecosystem | name ASC |
     `(no manifest found)`.
   - dependencies discrepancies: declared-but-unimported + imported-but-undeclared lists |
     20 each | name ASC | `(none)`.
   - module key-files: `File/LOC/Fan-in/Fan-out/Doc` | 10 | rank comparator | never empty.
   - module exported API: `Symbol/Kind/Signature/Doc/Source` | 40 + `…N more` | exported first,
     then containing-file rank, then span order | `(no exported symbols)`.
   Input type (locked): `RenderArtifacts = {inventory, factsByPath, graph, rank, metadata, plan,
   diagrams: {architecture: string, byCluster: Map<clusterId, string>}, toolVersion}`.
3. Escaping (B3 — EVERY repo-derived string in deterministic sections: docs, signatures, names,
   dependency names, README extracts): `escapeCell(text)` escapes `|`, backslash, backticks;
   newlines → spaces; neutralizes leading markdown (`#`, `>`, `-`) at cell start; strips raw
   HTML tags. Citations and mermaid fences are the only unescaped constructs. Hostile-docstring
   fixture proves tables/headings cannot be broken out of.
4. Citations (`citations.ts`): `cite(pageRelPath, filePath, span?)` — href = POSIX relative
   path computed from the page's ACTUAL location under `config.output.dir` to the repo file
   (default output dir: page `docs/wiki/modules/x.md` → `../../../src/foo.ts`); anchor
   `#L10-L62` (single line `#L10`); text `src/ask/retrieve.ts:10-62`. Computed only from
   FileFacts spans — never from prose. Tested at default AND custom `output.dir` depths.
5. Prose slots (`prose-slots.ts`):
   - Slot registry (locked): overview: `whatIsThis`, `whereToStart`; architecture:
     `architectureNarrative` (cluster one-liners from issue 24 render as table CELLS, not
     marker slots); dependencies: `dependencyNotes`; module pages: `responsibility`, `keyFlow`.
     `ProseBySlot = Map<pageSlug, Partial<Record<slotName, string>>>` + the architecture
     one-liner map `Map<clusterId, string>`.
   - `renderSlot(slotName, proseMarkdownOrNull)` → marker pair per DESIGN §8.2 with content or
     the disabled-notice blockquote.
   - `sanitizeProse(md, ctx: {inventoryPaths: Set<string>, pageRelPath: string})`: strip raw
     HTML; percent-decode + normalize each link target ONCE before checking; absolute
     (`http…`), scheme-relative (`//`), root-absolute (`/x`), traversal-escaping, and
     non-inventory targets render as plain code text; images stripped; headings demoted to
     h4+. §13.4 whitelist gate — also reused by issues 32/35 for answer markdown.
   - `extractSlots(existingFileContent)` → map slot → current prose (used by incremental regen
     and by 20/26 renderHash normalization).
6. Front-matter (locked — supersedes the `inputsHash` duplication idea; the manifest is the
   single hash source): `slug` and `generatedBy: archmap@<toolVersion>` only.
7. `nav.ts`: `_meta/nav.json` = `{meta: {repoName, toolVersion}, pages: [ordered tree]}` —
   `repoName` from RepoMetadata (first manifest `name`, else repo directory basename); tree =
   Overview, Architecture, Dependencies, Modules (children = module pages in plan order = rank
   order). Titles localized.
   Also emit `_meta/module-graph.json` = the final ModuleGraph in canonical JSON (committed
   artifact; MCP `get_module_graph` (34) serves it without re-analysis).
8. `render.ts`: `renderWiki(plan, artifacts, proseBySlot, config): RenderedWiki`
   (`Map<relPath, content>` — pure; no fs). Byte-determinism given equal inputs.

## Acceptance Criteria

- [ ] Golden wiki for `fixture-mixed` (no-LLM mode) matches `tests/golden/` byte-for-byte,
      including nav.json, both in `en` and `ja` string tables.
- [ ] Citation links resolve: test walks every generated link, resolves it from the page's real
      path, and asserts the target exists and anchors match span lines — at default AND custom
      `output.dir` depth.
- [ ] `sanitizeProse` matrix: HTML, absolute link, scheme-relative, root-absolute,
      percent-encoded traversal (`%2e%2e/`), image, deep heading, non-inventory relative link —
      all neutralized per rules.
- [ ] `escapeCell` hostile fixture: docstring containing `| pipes`, newlines, backticks, `<img`,
      leading `#` — table structure intact in rendered page.
- [ ] Slot extraction round-trips: render → extract → re-render is idempotent; every registry
      slot appears exactly once per owning page.
- [ ] Discrepancy tables correct on a fixture with unused + undeclared deps; every locked
      empty-state string appears on an empty-repo fixture.

## Validation

```bash
npm run test -- wiki/render
```

## Dependencies

16, 17, 18.

## Non-goals

Writing files to disk (25), manifest content (20), HTML export.

## Design References

- DESIGN §8.1–8.2 (all rules), §13.4 (prose whitelist); ADR-002, ADR-003
