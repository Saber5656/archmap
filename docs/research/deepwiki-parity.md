# Research: DeepWiki parity and archmap differentiation

- Status: accepted (informs DESIGN.md v1 scope)
- Date: 2026-07-07
- Method: web research (sources at bottom), verified 2026-07-07

## 1. Reference products

### 1.1 DeepWiki / Devin Wiki (Cognition, SaaS)

| Aspect | Fact |
|---|---|
| Launch | 2025-04 (public DeepWiki), part of Devin product family |
| Input | Public GitHub repos via `github.com` → `deepwiki.com` URL swap; private repos require a Devin account |
| Output | Hierarchical wiki pages, Mermaid architecture/data-flow diagrams, source-linked citations |
| Q&A | "Ask" with fast mode (precomputed code graph) and deeper research mode |
| Freshness | Re-index on provider side; user has no local control |
| MCP | Official remote MCP at `https://mcp.deepwiki.com/` (`sse` and `mcp` endpoints), no auth, **public repos only**. Tools: `read_wiki_structure`, `read_wiki_contents`, `ask_question` |
| Hosting | Cloud only. Code is processed on Cognition infrastructure |

### 1.2 deepwiki-open (AsyncFuncAI, OSS, MIT)

| Aspect | Fact |
|---|---|
| Form | Self-hosted web app: Python backend (~52%) + Next.js/TypeScript frontend (~46%), Docker Compose deployment |
| Input | GitHub / GitLab / Bitbucket repos |
| Output | Interactive wiki in the web app; Mermaid diagrams; RAG Q&A ("Ask") + multi-turn "DeepResearch" |
| Artifact model | Wiki lives in the app (server cache/state). No evidence of git-committed Markdown as the canonical artifact |
| Providers | OpenAI, Google Gemini, OpenRouter, LiteLLM; Ollama supported via a dedicated Dockerfile (local is an option, not the default posture) |
| Freshness | No staleness/rot detection of generated wikis found |
| MCP | No MCP server found |
| License | MIT |

### 1.3 Adjacent tools (for positioning only)

| Tool | What it does | Relevance |
|---|---|---|
| Swimm (commercial) | Human-authored docs coupled to code, with auto-sync/staleness detection in CI | Proves demand for "docs that do not rot"; not auto-generated, not local-first |
| regenrek/deepwiki-mcp (OSS) | MCP that fetches deepwiki.com content | Confirms agent demand for wiki-as-context; still public-repo/cloud only |
| RepoAgent and similar LLM doc generators | One-shot doc generation | No freshness contract, no agent interface |

## 2. Gap analysis

| Capability | DeepWiki SaaS | deepwiki-open | Gap for a new tool |
|---|---|---|---|
| Wiki pages + diagrams + citations | Yes | Yes | Parity required (table stakes) |
| Grounded Q&A | Yes | Yes | Parity required |
| Works on local/private repos without cloud | No | Partially (self-host, but web-app + cloud LLM posture) | **Open** |
| Code never leaves the machine by default | No | No (Ollama optional) | **Open** |
| Wiki as git-committed Markdown (docs-as-code) | No | No | **Open** |
| Staleness detection / CI freshness gate | No | No | **Open** |
| MCP for private/local repos | Devin account required | No | **Open** |
| Usable with zero LLM (deterministic mode) | No | No | **Open** |

## 3. archmap strategy (decision)

archmap does not compete as "a cheaper DeepWiki clone" (deepwiki-open already owns that spot).
It occupies the intersection the incumbents structurally cannot serve:

1. **Git-native docs-as-code.** The wiki is plain Markdown committed into the target repo
   (`docs/wiki/` by default). `archmap check` is a CI gate that fails when the wiki no longer
   matches the code (hash-mapped freshness manifest). Regeneration is incremental. This is the
   "腐らないアーキテクチャ図" (architecture docs that never rot) thesis of this repository.
2. **Local-first privacy.** Default LLM endpoint is a loopback Ollama; any non-loopback endpoint
   requires an explicit `security.allowRemoteLlm: true` opt-in. A no-LLM degraded mode still
   produces the structural wiki (diagrams, module index, dependency graph, navigation). No
   telemetry, ever.
3. **Agent-native MCP.** A local stdio MCP server exposes the generated wiki to coding agents
   (Claude Code, Codex, Cursor) using DeepWiki-compatible tool names (`read_wiki_structure`,
   `read_wiki_contents`, `ask_question`) plus archmap extras (`search_wiki`, `get_module_graph`).
   This gives agents senior-engineer onboarding context for private repos, fully offline.

Architectural consequence: because local models are weaker, **all facts are extracted
deterministically** (tree-sitter symbols, import graph, entry points, rankings) and the LLM only
writes prose over verified facts; citations are generated mechanically, never by the LLM. This
makes output reproducible, cheap, small-model-tolerant, and hallucination-resistant — a technical
moat rather than a feature checkbox.

## 4. Facts that shaped technical decisions

| Fact (verified 2026-07) | Decision |
|---|---|
| `sqlite-vec` has platform issues in 2026 (Windows DLL vs better-sqlite3 12.x SQLite version; `node:sqlite` built with `OMIT_LOAD_EXTENSION`) and slow release cadence | v1 avoids native SQLite extensions: FTS5 (bundled in better-sqlite3) for recall + in-process cosine re-rank over candidate embeddings. sqlite-vec deferred to v2 as optional accelerator. See ADR-005 |
| `web-tree-sitter` (WASM) runs in Node without node-gyp; grammars compile to `.wasm` via `tree-sitter build --wasm` (wasi-sdk auto-download since v0.26.1); slower than native bindings but portable | v1 uses web-tree-sitter with vendored `.wasm` grammars for TS/TSX/JS/Python. See ADR-004 |
| DeepWiki MCP tool names are becoming a de-facto convention (`read_wiki_structure`, `read_wiki_contents`, `ask_question`) | archmap MCP mirrors these names for drop-in familiarity |
| deepwiki-open is MIT-licensed Python+Next.js | No code reuse planned (different architecture); license-compatible if small utilities are ever ported, with attribution |

## 5. Risks of this strategy

| Risk | Mitigation |
|---|---|
| Local-model prose quality below usefulness threshold | Deterministic layer carries the core value; prose is additive. Eval harness (issue 33) measures quality per model; docs recommend minimum model sizes |
| deepwiki-open adds Markdown export / freshness later | Our moat is the CI freshness contract + MCP + deterministic reproducibility as a system, not one feature |
| Niche too small (docs-as-code + local LLM overlap) | Primary user is the author (personal leverage guaranteed); OSS upside is optional |

## Sources

- https://docs.devin.ai/work-with-devin/deepwiki
- https://docs.devin.ai/work-with-devin/deepwiki-mcp
- https://cognition.com/blog/deepwiki-mcp-server
- https://github.com/AsyncFuncAI/deepwiki-open
- https://asyncfunc.mintlify.app/getting-started/introduction
- https://github.com/asg017/sqlite-vec (+ 2026 platform issue reports)
- https://github.com/tree-sitter/tree-sitter/tree/master/lib/binding_web
- https://www.npmjs.com/package/web-tree-sitter
- https://github.com/regenrek/deepwiki-mcp
