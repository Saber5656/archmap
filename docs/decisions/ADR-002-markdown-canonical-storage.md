# ADR-002: Git-committed Markdown is the canonical artifact

- Status: accepted
- Date: 2026-07-07

## Context

DeepWiki and deepwiki-open keep generated wikis inside their application (SaaS DB / server cache).
That makes the wiki invisible to git workflows: no diffs, no review, no CI contract, no offline
reading, and vendor lock-in of the artifact itself.

## Decision

1. The wiki is plain Markdown + Mermaid + two JSON files (`nav.json`, `manifest.json`) written to
   `docs/wiki/` (configurable) inside the target repo, intended to be committed.
2. Everything else (`.archmap/`: parse caches, summary caches, `index.db` with chunk text and
   embeddings) is regenerable, gitignored, and must never be committed (it may contain full code
   text, including secret-excluded file names).
3. Generated output is deterministic (sorted keys, stable slugs, LF, no timestamps except one
   informational `generatedAt`) so wiki diffs in PRs are meaningful and small.
4. Output writes are atomic (temp dir + rename swap): a crashed run never leaves a corrupt wiki.

## Consequences

- Wiki reviews happen in normal PRs; freshness is enforceable in CI (`archmap check`).
- Readable anywhere Markdown renders (GitHub, VS Code, Obsidian) with no archmap installed.
- Committed manifest adds some diff noise on regeneration; canonical serialization keeps it minimal.
- Hand edits inside generated files are detected and warned about, not merged (v1 rule).

## Alternatives rejected

- SQLite/app-internal wiki storage: kills pillar P1 (docs-as-code) entirely.
- HTML output as canonical: not diffable/reviewable; Markdown renders everywhere and HTML export
  can be added later (v2).
