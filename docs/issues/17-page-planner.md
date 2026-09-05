# 17 — Wiki page planner

## Summary

Implement the planner producing `PagePlan` (DESIGN §7.5): the fixed pages (overview, architecture,
dependencies) plus one module page per top cluster under `wiki.maxModulePages`, with stable slugs
and per-page prose budgets.

## Context

The plan is the contract between analysis and rendering/summarization: renderer (19), summarizer
(23), and manifest (20) all iterate plan pages. Slug stability is what keeps wiki diffs small
(ADR-002).

## Scope

- In: `src/wiki/plan/planner.ts`, slug utilities, PagePlan zod schema.
- Out: rendering, prose generation.

## Detailed Requirements

1. Fixed pages always planned: `index` (kind overview, sources `["*"]`), `architecture`
   (kind architecture, `["*"]`), `dependencies` (kind dependencies, sources = the manifest
   `path` values from RepoMetadata (issue 16) — may be empty; the renderer shows an empty-state
   table, and freshness additionally covers it via the aggregate inventory entry, issue 20).
2. Module pages: clusters ordered by the canonical rank comparator (score desc, id ASC); take
   top `wiki.maxModulePages`; skipped clusters → top-level PagePlan field
   `omittedClusters: [{id, files, score}]` (always present, sorted by score desc then id;
   renderer 19 lists them on the architecture page so nothing silently disappears).
3. Slugs: `modules/<slugified cluster id>`; slugify = lowercase, `/`→`-`, strip non
   `[a-z0-9-]`, collapse dashes; `(root)` → `modules/root`. Empty result after slugify (e.g.
   punctuation-only or non-ASCII directory name) → `modules/cluster-<h6>` where `<h6>` = first
   6 hex of sha256(cluster id). Collisions after slugify → colliding clusters (all but the
   ASCII-first one) get suffix `-<h6>` (hash of their own cluster id — stable even when new
   colliding clusters appear later). Slug derives ONLY from the cluster id, never from
   ordering/score.
4. `sources` per module page: the cluster's directory glob (`<clusterDir>/**`) — the freshness
   scope. NON-EXCLUSIVE clusters use explicit member file lists instead of globs: `(root)` and
   the synthetic `src` cluster (its glob `src/**` would over-match child clusters).
5. Prose budgets: per-page `proseBudgetTokens` — module pages 6000, overview 8000,
   architecture 8000, dependencies 3000 (constants; consumed by 23/24 when packing fact sheets).
6. Ordering (locked): `pages` array = fixed pages in fixed order (index, architecture,
   dependencies), then module pages in the canonical RANK order (issue 23 consumes budget in
   array order; nav display order is also this order). Each module page carries
   `score: number` (cluster score) for downstream display/decisions.
7. API: `planPages(graph, rank, metadata, config): PagePlan` — pure; golden-tested.

## Acceptance Criteria

- [ ] Fixture with 45 clusters and `maxModulePages: 40` plans exactly 40 module pages + 3 fixed;
      omitted 5 recorded with scores in `omittedClusters`.
- [ ] Slug stability: re-plan after adding an unrelated cluster keeps existing slugs identical;
      adding a NEW colliding cluster does not change the existing collided slug (hash-suffix
      stability test).
- [ ] Collision case (`src/a-b` vs `src/a_b`) yields the hash suffix deterministically; slug
      hostile matrix (`..`, `日本語`, punctuation-only, leading `/`, backslash) always yields a
      non-empty slug matching `/^modules\/[a-z0-9-]+$/` (never escapes `modules/`).
- [ ] `(root)` and synthetic `src` cluster pages use explicit file-list sources; a `src/a`
      child cluster's change does not appear in `modules/src` sources (over-match test).
- [ ] Module pages ordered by rank (not slug) in `pages`; each carries `score`.
- [ ] Golden PagePlan byte-stable on `fixture-mixed`.

## Validation

```bash
npm run test -- wiki/plan
```

## Dependencies

14, 15, 16.

## Non-goals

Custom user page templates (v2), per-page config overrides.

## Design References

- DESIGN §7.5 (schema, slug rules), §8.1 (page kinds), §9 (sources = freshness scope)
