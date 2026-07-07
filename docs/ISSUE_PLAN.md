# archmap — Issue Plan (v1)

- Status: draft for review (PR gate)
- Date: 2026-07-07
- Canonical source: this file + `docs/issues/NN-*.md`. GitHub Issues are derived artifacts.

## 1. v1 completion statement

v1 is complete when **all 42 issues below are closed with their Validation steps passing**, which
by construction delivers: `archmap init|generate|check|ask|serve|mcp` on macOS/Linux; deep
analysis for TS/TSX/JS/Python with fallback for other languages; a deterministic, git-committed
Markdown wiki (overview, architecture, dependencies, module pages, Mermaid diagrams, mechanical
citations, navigation); a freshness manifest with a CI gate; a local-first LLM subsystem
(Ollama-default, remote opt-in, `--no-llm` first-class); hybrid FTS5(+embeddings) grounded Q&A
with validated citations; a read-only stdio MCP server with DeepWiki-compatible tools; a
loopback-only viewer; the security model of DESIGN §13 implemented and red-team tested; npm
packaging under Apache-2.0; and a dogfooding CI where archmap's own wiki stays fresh.

No v1 product behavior exists outside these issues. Newly discovered implementation unknowns
(§7) may add issues but must not silently expand scope.

## 2. Issue list in recommended execution order

Order = issue number. One issue = one coherent unit, executable by a mid-level implementation
agent without guessing.

| # | File | Title | Wave |
|---|---|---|---|
| 01 | `issues/01-project-scaffold.md` | Project scaffold: TypeScript ESM, vitest, lint, CI, license | 0 |
| 02 | `issues/02-cli-skeleton-errors.md` | CLI skeleton, error taxonomy, exit codes | 0 |
| 03 | `issues/03-config-schema-loader.md` | Config schema, loader, JSON-schema export | 0 |
| 04 | `issues/04-logger-json-output.md` | Structured logger and `--json` envelope | 0 |
| 05 | `issues/05-fs-hash-canonical-utils.md` | Hashing, canonical JSON, atomic FS utils | 0 |
| 06 | `issues/06-init-command.md` | `archmap init` scaffolding command | 0 |
| 07 | `issues/07-file-scanner.md` | File scanner and inventory | 1 |
| 08 | `issues/08-secret-filter.md` | Secret detection and exclusion/redaction policy | 1 |
| 09 | `issues/09-tree-sitter-host.md` | web-tree-sitter host, vendored grammars, parse cache | 1 |
| 10 | `issues/10-ts-extractor.md` | TypeScript/JavaScript facts extractor | 1 |
| 11 | `issues/11-py-extractor.md` | Python facts extractor | 1 |
| 12 | `issues/12-fallback-extractor.md` | Fallback extractor for other languages | 1 |
| 13 | `issues/13-import-resolver-module-graph.md` | Import resolution and module graph | 1 |
| 14 | `issues/14-clustering-entrypoints.md` | Directory clustering and entrypoint detection | 1 |
| 15 | `issues/15-ranking.md` | PageRank + git churn ranking | 1 |
| 16 | `issues/16-repo-metadata.md` | Repo metadata collector | 1 |
| 17 | `issues/17-page-planner.md` | Wiki page planner | 2 |
| 18 | `issues/18-mermaid-generator.md` | Mermaid diagram generator | 2 |
| 19 | `issues/19-markdown-renderer.md` | Markdown renderer, citations, nav | 2 |
| 20 | `issues/20-freshness-manifest.md` | Freshness manifest writer | 2 |
| 21 | `issues/21-llm-client.md` | OpenAI-compatible LLM client with security gate and budget | 2 |
| 22 | `issues/22-prompt-pack.md` | Versioned prompt pack with injection containment | 2 |
| 23 | `issues/23-module-summarizer.md` | Module prose summarizer (map step) | 2 |
| 24 | `issues/24-overview-synthesizer.md` | Overview/architecture prose synthesizer (reduce step) | 2 |
| 25 | `issues/25-generate-command.md` | `archmap generate` orchestration and incremental mode | 2 |
| 26 | `issues/26-check-command.md` | `archmap check` freshness gate | 2 |
| 27 | `issues/27-fixtures-golden-benchmarks.md` | Fixture repos, golden tests, benchmarks | 2 |
| 28 | `issues/28-chunker.md` | Symbol-aligned chunker | 3 |
| 29 | `issues/29-fts-index.md` | SQLite FTS5 chunk index | 3 |
| 30 | `issues/30-embedding-client.md` | Embeddings client with cache | 3 |
| 31 | `issues/31-vector-store-rerank.md` | Embedding store and hybrid retrieval | 3 |
| 32 | `issues/32-ask-command.md` | `archmap ask` with validated citations | 3 |
| 33 | `issues/33-ask-eval-harness.md` | Ask evaluation harness | 3 |
| 34 | `issues/34-mcp-server-core.md` | MCP stdio server core (read-only tools) | 4 |
| 35 | `issues/35-mcp-ask-tool.md` | MCP `ask_question` tool + client setup docs | 4 |
| 36 | `issues/36-viewer-server.md` | Viewer HTTP server (loopback, confined) | 4 |
| 37 | `issues/37-viewer-ui.md` | Viewer UI (bundled, client-rendered) | 4 |
| 38 | `issues/38-serve-command.md` | `archmap serve` command polish | 4 |
| 39 | `issues/39-security-redteam-pass.md` | Security red-team pass with attack fixtures | 5 |
| 40 | `issues/40-ci-recipes-dogfood.md` | CI recipes and dogfooding job | 5 |
| 41 | `issues/41-packaging-release.md` | npm packaging and release pipeline | 5 |
| 42 | `issues/42-user-docs.md` | User documentation, README, SECURITY.md | 5 |

## 3. Dependency table

`A ← B` means A depends on B. Only direct dependencies listed; transitive implied.

| Issue | Depends on |
|---|---|
| 01 | — |
| 02 | 01 |
| 03 | 02 |
| 04 | 02 |
| 05 | 01 |
| 06 | 03, 04 |
| 07 | 03, 05 |
| 08 | 07 |
| 09 | 01, 05 |
| 10 | 09 |
| 11 | 09 |
| 12 | 07, 08, 09 |
| 13 | 10, 11, 12 |
| 14 | 13 |
| 15 | 13, 14 |
| 16 | 07, 08 |
| 17 | 14, 15, 16 |
| 18 | 13, 14, 15 |
| 19 | 16, 17, 18 |
| 20 | 19 |
| 21 | 03, 04 |
| 22 | 21 |
| 23 | 08, 17, 22 |
| 24 | 23 |
| 25 | 08, 19, 20, 23, 24 |
| 26 | 20, 25 |
| 27 | 25, 26 |
| 28 | 08, 10, 11, 12, 19 |
| 29 | 25, 28 |
| 30 | 21, 28, 29 |
| 31 | 29, 30 |
| 32 | 22, 31 |
| 33 | 27, 32 |
| 34 | 20, 22, 29 |
| 35 | 32, 34 |
| 36 | 19 |
| 37 | 27, 36 |
| 38 | 26, 37 |
| 39 | 25, 26, 27, 32, 35, 37, 38 |
| 40 | 26, 27 |
| 41 | 39, 40 |
| 42 | 41 |

Parallelizable examples: {10, 11, 12, 16} after 09; {21, 22} alongside {17, 18, 19}; wave 3 and
wave 4 (36–38) can run concurrently once their deps close.

## 4. Implementation waves

| Wave | Issues | Exit criterion (wave gate) |
|---|---|---|
| 0 Foundations | 01–06 | `archmap init` runs on a sample repo; CI green; exit-code/`--json` conventions locked |
| 1 Analysis core | 07–16 | Deterministic pipeline emits FileInventory/FileFacts/ModuleGraph/RankReport on fixtures, byte-stable across runs |
| 2 Wiki + freshness | 17–27 | `generate` (with and without LLM) + `check` pass golden tests; dogfood wiki generated for archmap itself |
| 3 Ask | 28–33 | `archmap ask` answers fixture questions with valid citations; fts-only degraded mode verified |
| 4 Interfaces | 34–38 | MCP tools usable from a real agent client; viewer browses the dogfood wiki |
| 5 Hardening + release | 39–42 | Red-team fixtures pass; `npm pack` installable via `npx`; docs complete; v0.1.0 tag ready (release itself is a separate human gate) |

## 5. Coverage table (DESIGN.md § → issues)

| DESIGN section | Issues |
|---|---|
| §3 CLI contract | 02, 06, 25, 26, 32, 34, 38 |
| §4 architecture & source layout | 01, 02 |
| §5 storage layout | 05, 19, 20 |
| §6 configuration | 03, 06 |
| §7.1 FileInventory | 07 |
| §7.2 FileFacts | 09, 10, 11, 12 |
| §7.3 ModuleGraph | 13, 14 |
| §7.4 RankReport | 15 |
| §7.5 PagePlan | 17 |
| §7.6 WikiManifest | 20 |
| §7.7 Ask contracts | 28, 29, 30, 31, 32 |
| §8 wiki IA & rendering | 17, 18, 19 |
| §9 freshness model | 20, 25, 26 |
| §10 LLM subsystem | 21, 22, 23, 24 |
| §11 Ask subsystem | 28–33 |
| §12.1 MCP | 34, 35 |
| §12.2 viewer | 36, 37, 38 |
| §13 security model | 08 (13.3), 21 (13.2), 22 (13.4), 34 (13.4/13.5), 36 (13.5), 39 (13.7), 41 (13.6) |
| §14 errors/exit codes | 02 |
| §15 performance targets | 27 |
| §16 testing strategy | 01, 27, 33, 39, 40 |
| §17 packaging | 41 |

Every DESIGN section with implementable behavior is covered; §1–2 and §18 are plan-level.

## 6. Whole-product validation strategy

1. **Per-issue**: each issue's Validation section is mandatory for closing (unit tests + named
   commands).
2. **Golden determinism**: fixture repos → `generate --no-llm` byte-compared against
   `tests/golden/` on macOS + Linux CI (issue 27).
3. **LLM contracts without LLM**: mock OpenAI-compatible server asserts prompt structure
   (sentinels, budgets) and feeds canned/malformed JSON to test validation + repair + fallback
   paths (issues 21–24, 32).
4. **Real-model smoke (local only)**: `ARCHMAP_E2E_OLLAMA=1 npm run e2e:ollama` runs generate+ask
   against a live Ollama; never in CI (issue 27, 33).
5. **Security**: red-team fixtures for secrets/injection/traversal + network-egress assertion
   (issue 39) gate wave 5.
6. **Dogfood**: CI generates archmap's own wiki (`--no-llm`) and `archmap check` must pass on
   every PR (issue 40) — the freshness contract is continuously self-tested.
7. **Ask quality**: model-dependent eval rates tracked per model, informational in v1; the
   deterministic `--retrieval-only` hit-rate (≥ 0.9 on fixtures) IS CI-gated (issue 33).

## 7. Known unknowns → possible new issues

| Unknown (DESIGN §2.4) | Trigger | Likely new issue |
|---|---|---|
| Local-model prose below usefulness | Eval scores from 23/33 | Prompt v2 iteration; model recommendation matrix |
| WASM grammar size/quirks | Issue 09 | Lazy checksummed grammar download |
| FTS5 code-tokenization gaps | Issue 33 scorecard | Custom tokenizer pass |
| Mermaid unreadable at scale | Issue 27 large fixture | Diagram aggregation strategies |
| Embedding memory at 300k LOC | Issue 27/31 telemetry | Chunk cap / int8 quantization |
| Windows path/FTS behavior | Community reports | Windows CI job |

## 8. Deferred to v2 (not planned, recorded)

Remote repo cloning; watch daemon; sqlite-vec accelerator; Go/Rust/Java extractors;
DeepResearch-style multi-hop Ask; static HTML export; Obsidian wikilink mode; wiki-diff PR bot;
marketplace GitHub Action; local reranker; viewer server-side search.
