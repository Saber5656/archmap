# 15 — PageRank + git churn ranking

## Summary

Compute the `RankReport` (DESIGN §7.4): PageRank over the module graph, git churn per file over a
configurable window, entrypoint boost, and the combined score used to order pages, tables, and
diagram inclusion.

## Context

Rank decides what the wiki emphasizes under budget (planner 17, diagrams 18, tables 19, ask chunk
cap). Formula and tie-breaking are fixed in DESIGN §7.4 for determinism.

## Scope

- In: `src/core/rank/pagerank.ts`, `src/core/rank/churn.ts`, `src/core/rank/score.ts`.
- Out: consumption by planner/renderer.

## Detailed Requirements

1. `pagerank.ts`: standard PageRank on file nodes with edges reversed importance-wise
   (an import edge `A→B` transfers rank from A to B — being imported is what matters):
   damping 0.85, uniform init, dangling mass redistributed uniformly, max 50 iterations or
   L1 delta < 1e-8. Pure function over ModuleGraph; deterministic iteration order (sorted nodes).
2. `churn.ts`: `churnDays: 0` → churn DISABLED: no git invocation at all, every path gets
   churn 0, NO warning (deterministic CI mode; issue 40 relies on it). Otherwise: hardened git
   invocation via `execFile` (no shell) at `repoRoot`:
   `git -c core.pager=cat log --numstat --no-renames --no-ext-diff --no-textconv
   --since=<churnDays>.days --pretty=format:` with env stripped to `{PATH, HOME,
   GIT_CONFIG_NOSYSTEM: "1"}` (config-driven external diff/textconv helpers must not run on an
   untrusted repo — test proves a configured external diff is not invoked). Sum
   (additions + deletions) per path; paths not in inventory dropped; git absent, non-repo, or
   empty history → all inventory paths get churn 0 with warning token exactly
   `churn-unavailable` (distinct from the CLI's `E_NOT_A_REPO` guard, which fires earlier for
   the repo itself). Binary `-` numstat lines count 0.
3. `score.ts`: exactly DESIGN §7.4:
   `score = 0.6·norm(pagerank) + 0.25·norm(log1p(churn)) + 0.15·entrypointBoost` where `norm` is
   min-max over nodes (all-equal → 0.5), `entrypointBoost` = 1 for entrypoint paths else 0.
   Cluster score = mean of member scores weighted by per-file loc (from `locByPath`; missing →
   0-weight, all-zero cluster → unweighted mean).
4. API + output (locked): `computeRankReport({graph, locByPath: Map<string, number>, repoRoot,
   churnDays, logger}): Promise<RankReport>` — cluster membership from `graph.nodes[].cluster`.
   RankReport arrays are SORTED: nodes by `score` desc then `id` ASC; clusters by `score` desc
   then `id` ASC (the single canonical comparator; downstream 17/18/23 must not re-sort
   differently). Values rounded to 6 decimals AFTER all math (goldens depend on it).
5. Performance: ≤ 1 s for 10k nodes / 50k edges (typed arrays; no per-iteration allocation).

## Acceptance Criteria

- [ ] Hand-computable 4-node fixture: PageRank values match analytic expectation within 1e-6.
- [ ] Churn parsed from a scripted fixture repo (test creates real git history with 2 commits);
      `--since` respected; numstat binary lines handled; a repo-configured `diff.external`
      helper is NOT invoked (spy/marker file proof).
- [ ] Empty-history and git-failure cases: all churn 0 + warning token exactly
      `churn-unavailable`, no throw; `churnDays: 0` → zero churn, no git call (execFile spy),
      no warning.
- [ ] Entrypoint boost visible in final ordering; node and cluster arrays sorted by the
      canonical comparator (score desc, id ASC — tie fixture proves it).
- [ ] Full RankReport golden on `fixture-mixed` is byte-stable across two runs.
- [ ] 10k-node synthetic graph ranks in ≤ 1 s (informational assert).

## Validation

```bash
npm run test -- core/rank
```

## Dependencies

13, 14.

## Non-goals

Author/ownership analytics, time-decayed churn (v2), configurable weights (constants in v1).

## Design References

- DESIGN §7.4 (formula, tie-break), §2.4 (churn cost unknown)
