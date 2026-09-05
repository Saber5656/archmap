# 14 — Directory clustering and entrypoint detection

## Summary

Assign every graph node to a cluster (directory-based, DESIGN §7.3 rules), compute cluster
metrics, and detect repo entrypoints with reasons.

## Context

Clusters become wiki module pages (17) and diagram groupings (18); entrypoints anchor the
architecture page and "where to start" prose. Rules must be deterministic and honest.

## Scope

- In: `src/core/graph/cluster.ts`, `src/core/graph/entrypoints.ts`.
- Out: ranking (15), page planning (17).

## Detailed Requirements

1. Clustering (`cluster.ts`), default depth rule per DESIGN §7.3:
   - Candidate cluster roots: each direct child directory of `repoRoot` and, when `src/` exists,
     each direct child of `src/` (`src/` itself is then not a cluster).
   - Files directly under a candidate root (or deeper) belong to it; files at repo root or
     directly under `src/` go to synthetic cluster `"(root)"` / `"src"` respectively.
   - Clusters with < 2 files are merged into their parent candidate (or `(root)`).
   - Cluster `id` = repo-relative dir path (`"(root)"` for synthetic root); `label` = last path
     segment.
2. Metrics per cluster: `files`, `loc` (sum from facts), `internalEdges` (both ends inside),
   `externalEdges` (one end inside), computed from resolved edges.
3. Entrypoint detection (`entrypoints.ts`) — each `{path, reason}` with the CLOSED reason union
   and priority (strongest first; dedup by path keeps the strongest):
   `"package.json:bin"` > `"package.json:main"` (covers `main`/`module`; `exports` flattened:
   string values of the `"."` entry and its `import`/`require`/`default` conditions, max depth
   2, reason still `package.json:main`) > `"package.json:scripts"` > `"pyproject:scripts"` >
   `"python:__main__"` (any `__main__.py`) > `"python:main-guard"` (`hasMainGuard` facts field
   from issue 11) > `"convention"`.
   - npm scripts minimal grammar (locked): split script value on `&&` and `;`; tokenize each
     part on whitespace respecting single/double quotes; skip leading `ENV=val` assignments;
     accepted launchers: `node`, `tsx`, `ts-node`, `python`, `python3`; the first non-flag
     argument after the launcher that (normalized) is an inventory file → entrypoint. Anything
     else (npm indirection, other runners, glob args) is ignored — explicit non-goal.
   - `pyproject.toml [project.scripts]` values `pkg.mod:func` → resolve `pkg/mod.py` (or
     `pkg/mod/__init__.py`) via issue 13's python candidate rules; unresolvable → skipped with
     debug log.
   - Convention fallback (only when nothing stronger exists for that language):
     TS/JS: `src/index.ts`, `src/main.ts`, `index.ts`; Python: `main.py`, `app.py`, `cli.py`
     at repo root or under `src/`.
   - Manifest values are DATA, never executed; path-like values normalized repo-relative and
     confined (absolute, `..`-escaping, or non-inventory targets are dropped with debug log) —
     emitted entrypoints must be existing graph node ids.
4. Manifest parsing (this issue reads root `package.json` + `pyproject.toml` itself, as data):
   missing → skip silently; malformed → warn `manifest-unparseable`, never crash.
5. APIs (pure, return new objects; inputs immutable):
   `clusterGraph(graph, locByPath: Map<string, number>): ModuleGraph` (fills `cluster`,
   `clusters[]`; missing loc → 0) and
   `detectEntrypoints({repoRoot, graph, factsByPath, readManifest: (relPath) => string | null,
   logger}): ModuleGraph` (fills `entrypoints[]`). Deterministic ordering (clusters by id,
   entrypoints by path).

## Acceptance Criteria

- [ ] Fixture monorepo layout (`src/a`, `src/b`, `tools/`, root scripts, 1-file dir) clusters per
      rules with merge behavior verified (golden).
- [ ] Node package fixture: bin + main + `exports`-condition entry + script-referenced file all
      detected with correct closed-union reasons; strongest-reason dedup proven.
- [ ] npm script grammar negative cases: `ENV=x node src/a.ts` (detected), quoted path with
      space (detected), `npm run other`, `jest src/`, `node $VAR` (all ignored).
- [ ] Python fixture: `pyproject [project.scripts]` (incl. one unresolvable target skipped),
      `__main__.py`, main-guard file detected.
- [ ] Malicious manifest fixture: `bin` pointing at `../outside.js`, absolute path, and a shell
      snippet value — none emitted, nothing executed, `manifest-unparseable` only where JSON is
      actually broken.
- [ ] Every node has a non-empty `cluster`; cluster metrics sum consistently with edge counts
      (invariant tests).

## Validation

```bash
npm run test -- core/graph/cluster core/graph/entrypoints
```

## Dependencies

13.

## Non-goals

Graph-community clustering (v2 idea), Dockerfile/CI entrypoint mining (v2).

## Design References

- DESIGN §7.3 (clustering rules), §8.1 (architecture page consumes entrypoints)
