# 18 — Mermaid diagram generator

## Summary

Generate deterministic Mermaid diagrams from the module graph: the system architecture diagram
(cluster level) and per-cluster dependency diagrams, with sanitized labels, node caps, and
aggregation per DESIGN §8.2.

## Context

Diagrams are the visual core of pillar P1 and must render on GitHub, in the viewer (strict
security level), and in Obsidian. Labels derive from repo content → injection surface (§13.4/B3).

## Scope

- In: `src/wiki/render/mermaid.ts`.
- Out: page assembly (19).

## Detailed Requirements

1. Node cap semantics (locked, matches DESIGN §8.2): EVERY diagram contains at most
   `wiki.maxDiagramNodes` nodes TOTAL, counting aggregate, external, and stub nodes.
2. Architecture diagram (`flowchart LR`): one node per cluster by rank order; on overflow, top
   `maxDiagramNodes - 1` clusters + ONE aggregate node labeled `… and N more`. Edges =
   cluster-level aggregated import edges; edge label = count when > 1 (`-->|12|`). Overflow
   edge rule: edges touching omitted clusters are REDIRECTED to the aggregate node with counts
   summed; aggregate↔aggregate self-edges suppressed. Entrypoint clusters get a distinct class
   (`classDef entry`); externals not shown here.
3. Cluster diagram (`flowchart TD`) per module page, within the same total cap budgeted as:
   up to 5 external-package nodes (`[( … )]` shape) + up to 2 stub nodes (one inbound
   `other clusters →`, one outbound) + member file nodes filling the remainder (by file score;
   overflow → one aggregate node counted in the cap, same edge-redirect rule). Per-cluster
   external usage is computed from `factsByPath` (resolved `external` import entries of member
   files), ranked by member-importer count desc then name ASC; edge direction: file → external.
4. Sanitization (hard requirements, tested with hostile fixtures):
   - Node ids are synthetic `n0…nN` for EVERY node type — clusters, files, aggregates,
     externals, stubs (assignment order = sorted underlying id; aggregate/stubs use fixed
     synthetic keys `(agg)`, `(in)`, `(out)` in the sort domain).
   - Labels (ALL node types incl. externals/stubs, and edge labels): strip backticks, `"`,
     `<`, `>`, `{`, `}`, `|`, `;`, `%`, newlines; collapse whitespace; max 40 chars with
     middle-ellipsis; wrapped in `["…"]` quoting form.
   - Output contains no `click`, no `callback`, no `%%` directives, no `---`/`;`-terminated
     statement injection — asserted as token-absence over the full diagram text.
5. Determinism: stable node ordering, stable edge ordering, LF, no timestamps.
6. Output API: `architectureDiagram(graph, rank, cfg): string`,
   `clusterDiagram(graph, rank, factsByPath, clusterId, cfg): string` — returns diagram body
   (without the ```mermaid fence; renderer adds fences).
7. Validity check in tests: parse generated diagrams with the `mermaid` package's parser (dev
   dependency) — every fixture diagram must parse without error.

## Acceptance Criteria

- [ ] Architecture + cluster diagrams for `fixture-mixed` match goldens and parse with mermaid.
- [ ] Hostile fixture (filenames AND external package names AND cluster names containing
      `a"];click evil[.ts`, backticks, `%%`, `;`, `callback`, unicode) produces parseable
      diagrams; token-absence assertions for `click`, `callback`, `%%` over full output.
- [ ] Node cap: 100-cluster synthetic graph → exactly `maxDiagramNodes` total nodes (29 + 1
      aggregate at default 30); omitted clusters' edges redirected to the aggregate with summed
      counts; no aggregate self-edge; cluster diagram respects the 5-external/2-stub budget
      within the same total cap.
- [ ] Byte-stable across runs.

## Validation

```bash
npm run test -- wiki/render/mermaid
```

## Dependencies

13, 14, 15 (graph, clusters, and rank scores for node capping/ordering).

## Non-goals

Sequence diagrams, click-through interactivity, theming.

## Design References

- DESIGN §8.2 (mermaid rules), §13.4 (injection), §2.4 (scale unknown)
