# 13 — Import resolution and module graph

## Summary

Resolve raw import specifiers from FileFacts to repo files where possible and build the
`ModuleGraph` (DESIGN §7.3): nodes, import edges, external dependency aggregation, and an honest
`unresolved` list.

## Context

The graph is the backbone of clustering (14), ranking (15), diagrams (18), and
`get_module_graph` (34). Resolution must be deterministic and must never fabricate edges.

## Scope

- In: `src/core/graph/resolve-ts.ts`, `resolve-py.ts`, `resolve-fallback.ts`, `build.ts`,
  ModuleGraph zod schema.
- Out: clustering/entrypoints (14), rank metrics (15).

## Detailed Requirements

1. TS/JS resolution (`resolve-ts.ts`) — input kinds from issue 10 are `internal` (relative) and
   `unknown` (all bare specifiers):
   - `internal` relative specifiers: try exact, then `.ts/.tsx/.js/.jsx/.mjs/.cjs` extensions,
     then `/index.<ext>`; specifier with explicit `.js` also tries `.ts/.tsx` (NodeNext
     convention).
   - `unknown` bare specifiers: FIRST try tsconfig `paths`/`baseUrl` (read `tsconfig.json` at
     repo root plus `extends` chain, max depth 3; longest-prefix alias mapping, then the
     relative rules on the mapped result). Only specifiers that match no alias are then
     classified `external`. Workspace/monorepo package resolution is OUT of scope for v1:
     bare specifiers that match no tsconfig alias become `external` even if they name a
     workspace package (recorded in `stats` as a note when root package.json has `workspaces`).
   - Unresolvable relative or alias-mapped specifiers → `unresolved`.
2. Python resolution (`resolve-py.ts`) — input kinds from issue 11 are `internal` (relative) and
   `unknown` (absolute). Deterministic candidate order (locked):
   - `import M` (absolute, `M = a.b`): if top segment ∈ first-party roots →
     `a/b.py` then `a/b/__init__.py` (relative to the detected root's parent); hit → edge; miss
     → `unresolved`. Top segment not first-party → `external`.
   - `from M import n1, n2` (absolute): resolve `M` as above to its directory/module; then for
     EACH name `n`: try `<M>/<n>.py`, then `<M>/<n>/__init__.py` (submodule import → edge per
     resolved name); if no name resolves as a submodule, edge to `M`'s own module file
     (`<M>.py` or `<M>/__init__.py`) — the names are attributes.
   - Relative `from .[..]P import n`: anchor at the importing file's package, ascend one level
     per extra dot, then apply the same per-name candidate order against `P` (empty `P` = the
     anchor package itself: try `<anchor>/<n>.py`, `<anchor>/<n>/__init__.py`, else
     `<anchor>/__init__.py`).
   - First-party roots: directories containing `__init__.py` directly under repo root or under
     `src/`; plus `pyproject.toml [project] name` normalized (dash→underscore).
3. Fallback resolution: `internal` relative specifiers only — normalized join + inventory
   membership check (per issue 12). Fallback `unknown` entries: matching
   `/^[A-Za-z0-9_@][A-Za-z0-9_@.:-]*$/` (package-shaped, incl. dotted `a.b.C` and
   `crate::a::b`) → `external`; everything else → `unresolved`.
4. `build.ts`: assemble the graph per DESIGN §7.3 with the PRE-CLUSTERING shape (locked):
   - `nodes`: one per analyzable file (deep + fallback), `fanIn`/`fanOut` from resolved edges
     only; `cluster: ""` (issue 14 fills it; the zod schema allows empty string until then).
   - `edges`: `{from, to, kind: "import"}`, deduplicated, self-edges dropped, sorted by
     `(from, to)`.
   - `clusters: []`, `entrypoints: []` (issue 14 populates).
   - `externals`: aggregated by top-level package name (`@scope/pkg` kept whole, `pkg/sub` →
     `pkg`; Python: top module), with `importers` counts; no `unknown` kind survives.
   - `unresolved`: `{from, raw}` sorted.
   - `stats`: `{resolutionRate}` where
     `resolutionRate = resolvedInternal / (resolvedInternal + unresolved.length)` over import
     entries that were candidates for internal resolution (externals excluded from both terms);
     `1` when the denominator is 0.
5. Path-confinement security requirement (repo content is untrusted, DESIGN §13): all candidate
   paths are normalized repo-relative POSIX; candidates that escape `repoRoot` after
   normalization (specifiers or tsconfig alias targets containing `..`/absolute paths) are
   NEVER stat'd or read — they go straight to `unresolved`; tsconfig `extends` targets outside
   the repo are ignored with a warn. Resolution performs no filesystem reads at all: existence
   checks are inventory-membership lookups (traversal-safe by construction; test proves no fs
   access with an fs spy).
6. Determinism: byte-stable output on fixtures (canonical JSON golden).
7. API: `buildModuleGraph(inventory, factsByPath, repoRoot, tsconfig?): ModuleGraph` — pure.

## Acceptance Criteria

- [ ] TS fixture with tsconfig `paths` alias (`@app/*` — arriving as `unknown` from the
      extractor), NodeNext `.js`-suffixed relative import, `index.ts` folder import, and a bare
      external — all resolved/classified correctly (golden).
- [ ] Py fixture with `src/pkg/__init__.py`, `from ..a import b` (b as submodule AND b as
      attribute variants), `from . import x`, absolute first-party and stdlib imports — correct
      edges per the locked candidate order; stdlib lands in `externals`.
- [ ] No edge exists whose target file is not in the inventory (invariant test); no `unknown`
      kind survives into the graph.
- [ ] Traversal fixtures: specifier `../../../../etc/passwd`, tsconfig alias mapping outside the
      repo, absolute-path specifier — all land in `unresolved`, zero fs reads (spy).
- [ ] `unresolved` populated (not silently dropped) for an alias without tsconfig mapping.
- [ ] `resolutionRate` matches the locked formula on fixtures; 100% for the deep-language
      fixtures.

## Validation

```bash
npm run test -- core/graph
```

## Dependencies

10, 11, 12.

## Non-goals

Multi-tsconfig workspaces, Go/Rust/Java resolution, dependency version reading (16 handles
manifests).

## Design References

- DESIGN §7.3 (schema, clustering note), §2.4 (honesty about unresolved)
