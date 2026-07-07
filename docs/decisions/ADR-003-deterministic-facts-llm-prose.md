# ADR-003: Deterministic facts, LLM prose

- Status: accepted
- Date: 2026-07-07

## Context

Local models (8B-class) hallucinate structure and citations when asked to "document this repo".
Freshness checking also requires knowing exactly which inputs produced which output — impossible
when the LLM decides content structure. Incumbent tools are LLM-first and inherit both problems.

## Decision

1. Static analysis produces all structure: file inventory, symbols, import graph, clusters,
   entrypoints, rankings, diagrams, tables, navigation, and source citations (path + line spans
   from tree-sitter). These render even with no LLM.
2. The LLM receives compact fact sheets and returns only prose paragraphs as schema-validated
   JSON fields with hard word limits. Prose is inserted into marked slots
   (`<!-- archmap:prose:begin … -->`) inside deterministic templates.
3. The model never emits links, paths, citations, tables, or diagram content. Ask citations are
   chunk-id references validated against the retrieved set, so fabricated citations are
   structurally impossible.
4. Prose generation failures degrade per page to the no-LLM marker; they never abort a run.

## Consequences

- Small-model tolerant; reproducible; hallucination surface limited to prose wording.
- Freshness mapping (inputs → page) is exact, enabling `archmap check` and incremental regen.
- Prose quality ceiling is lower than a frontier-model free-form wiki; accepted trade-off, and the
  slot design allows richer synthesis later without changing the contract.

## Alternatives rejected

- LLM-planned wiki (model decides pages/structure): non-reproducible, unverifiable, breaks check.
- No LLM at all: loses the "explains *why*, not just *what*" value that makes wikis readable.
