# Wave 10 concrete review-resolution addendum

Repository: Saber5656/archmap
Pull request: #43

This addendum is the normative response to the 33 mapped human review findings on
this documentation-only pull request. It records one implementation path and one
focused verification gate per finding. The current head, base, and this file's blob
identity are pinned in the independent review manifest, not duplicated here, so a
later change cannot be mistaken for the reviewed tree. Any head or base change
invalidates this addendum and requires a fresh review.

The contracts below are design/documentation contracts. They do not claim that the
product implementation or any validation has already succeeded. A thread is
eligible for resolution only after its focused gate, the repository's full
validation, and the applicable security/privacy handoff have terminal success for
the pinned identity. No PR review bot is triggered or rerun by this addendum.

## Mandatory completion gates

The implementation owner must attach focused-test evidence and the repository's
full validation evidence to the PR before resolution. A security/privacy reviewer
must accept the path, input, output, and secret-handling changes that apply to the
thread. Missing, pending, skipped, cancelled, timed-out, stale, or unknown evidence
is a blocker. The final merge gate must re-fetch the PR head/base, required-check
policy, review decision, and unresolved-thread state immediately before any merge;
the addendum itself is not merge authorization.

## Thread contracts

### 1. Include the committed module graph in the canonical artifact set

- Thread: PRRT_kwDOTNkDkM6Oybdb
- Location: docs/decisions/ADR-002-markdown-canonical-storage.md:21
- Normative resolution: The canonical committed wiki artifact set includes
  _meta/module-graph.json. The generator writes this file deterministically from
  the sorted module graph, and the artifact manifest treats its absence as an
  error.
- Focused gate: Generate a fixture wiki twice from the same input and assert that
  both _meta/module-graph.json files are byte-identical, listed in the manifest,
  and committed by the canonical-artifact test.

### 2. Tighten the parallelism note

- Thread: PRRT_kwDOTNkDkM6Oybdh
- Location: docs/ISSUE_PLAN.md:123
- Normative resolution: The execution note states that step 36 completes before
  step 37 and step 37 completes before step 38; the three steps are serialized and
  are not presented as concurrent work.
- Focused gate: Render the Issue Plan and assert that the paragraph at line 123
  contains the ordered 36 -> 37 -> 38 dependency and that no parallel/concurrent
  wording remains for those steps.

### 3. Strip $schema before strict validation

- Thread: PRRT_kwDOTNkDkM6Oybdi
- Location: docs/issues/03-config-schema-loader.md:44
- Normative resolution: Config loading removes only the top-level $schema
  initialization key before strict schema validation. Every other unknown key
  remains a validation error and the loader preserves the original input for
  diagnostics.
- Focused gate: Feed the loader a fixture with top-level $schema and a second
  unknown key; assert that the first loads and the second produces the strict
  unknown-key error.

### 4. Use JS-compatible case-insensitive syntax

- Thread: PRRT_kwDOTNkDkM6Oybdl
- Location: docs/issues/08-secret-filter.md:26
- Normative resolution: Every JavaScript secret-filter expression uses a valid
  trailing i flag, never an inline (?i) modifier.
- Focused gate: Compile every documented regex with the project JavaScript runtime
  and run a mixed-case secret fixture through each expression.

### 5. Avoid concatenating version strings into the cache key

- Thread: PRRT_kwDOTNkDkM6Oybdn
- Location: docs/issues/09-tree-sitter-host.md:50
- Normative resolution: The parse cache key is the canonical JSON serialization of
  an object containing filePath, grammarId, and parserVersion, with object keys in
  fixed order. It is not a raw concatenation of version strings.
- Focused gate: Assert distinct keys for fixtures whose path, grammar, or parser
  version differs, and assert stable equality for the same tuple across processes.

### 6. Keep the shared imports contract aligned

- Thread: PRRT_kwDOTNkDkM6Oybdp
- Location: docs/issues/10-ts-extractor.md:27
- Normative resolution: FileFacts.imports entries contain resolved: string | null
  in the shared schema. The extractor writes null when resolution is unavailable
  and a canonical absolute module path when it succeeds; no undocumented field is
  emitted.
- Focused gate: Validate one resolved and one unresolved import fixture against the
  shared schema and assert exact serialization of the resolved field.

### 7. Clarify the tsconfig loading boundary

- Thread: PRRT_kwDOTNkDkM6Oybds
- Location: docs/issues/13-import-resolver-module-graph.md:33
- Normative resolution: tsconfig loading is a separate filesystem boundary.
  buildModuleGraph accepts a preloaded immutable config and performs pure graph
  construction without reading files; the orchestrator owns all disk reads.
- Focused gate: Invoke buildModuleGraph with a fixture config while filesystem reads
  are mocked to throw, and assert that the graph is produced without a read.

### 8. Keep src out of the merge rule

- Thread: PRRT_kwDOTNkDkM6Oybdv
- Location: docs/issues/14-clustering-entrypoints.md:25
- Normative resolution: A synthetic module or a module under src remains its own
  cluster when it is the only member; the merge rule never collapses such a node
  directly into the repository root merely because the cluster has one file.
- Focused gate: Run the clustering fixture with one src file and one synthetic
  module and assert two non-root cluster identities.

### 9. Make README selection deterministic

- Thread: PRRT_kwDOTNkDkM6Oybdx
- Location: docs/issues/16-repo-metadata.md:49
- Normative resolution: README selection precedence is README.md, then
  README.en.md, then README.rst, then README.txt; ties within a class are resolved
  by normalized path order. The selected path is recorded in repo metadata.
- Focused gate: Present all four variants in permuted directory listings and assert
  that README.md is selected every time, with lexicographic selection for ties.

### 10. Define overflow handling for external packages and stubs

- Thread: PRRT_kwDOTNkDkM6Oybdz
- Location: docs/issues/18-mermaid-generator.md:34
- Normative resolution: External packages are sorted by canonical package name and
  limited to the first five; stubs are sorted by canonical module path and limited
  to the first two. When either cap removes entries, the diagram emits one
  count-only overflow node for that class; no entry is silently discarded.
- Focused gate: Generate a fixture with six external packages and three stubs and
  assert the exact five/two retained nodes plus one external-overflow and one
  stub-overflow node.

### 11. Separate edge-label syntax from node labels

- Thread: PRRT_kwDOTNkDkM6Oybd1
- Location: docs/issues/18-mermaid-generator.md:43
- Normative resolution: Mermaid node labels use the form ID["label"], while edge
  labels use the form A -->|"label"| B. The generator escapes label quotes for
  the selected form independently.
- Focused gate: Snapshot a diagram containing one node label and one edge label and
  assert the two exact syntaxes in the generated Mermaid text.

### 12. Use one canonical inventory fingerprint everywhere

- Thread: PRRT_kwDOTNkDkM6Oybd3
- Location: docs/issues/20-freshness-manifest.md:42
- Normative resolution: Each inventory record is the UTF-8 tuple
  path + NUL + sha256 + NUL + sizeBytes + NUL + sorted comma-joined flags. Records
  are sorted by normalized path and the aggregate fingerprint is SHA-256 of the
  newline-joined records. Manifest, freshness check, and cache use this exact
  representation.
- Focused gate: Build fixtures with reordered input and reordered flags and assert
  the same aggregate hash; change any tuple member and assert a different hash.

### 13. Split the mock contract by endpoint

- Thread: PRRT_kwDOTNkDkM6Oybd5
- Location: docs/issues/27-fixtures-golden-benchmarks.md:45
- Normative resolution: The fake service exposes separate typed fixtures for chat,
  models, and embeddings. Chat returns choices[].message, models returns data[]
  model entries, and embeddings returns data[] embedding entries; one endpoint
  fixture is never reused as another endpoint's response.
- Focused gate: Run the three endpoint fixtures through their clients and assert
  their distinct response schemas and request paths.

### 14. Keep the standard JSON envelope here

- Thread: PRRT_kwDOTNkDkM6Oybd8
- Location: docs/issues/32-ask-command.md:55
- Normative resolution: The ask command always returns the envelope
  {ok, command, data, errors[], warnings[]} in JSON mode. Successful data and
  failure details are nested in that envelope; no command-specific top-level shape
  is emitted.
- Focused gate: Snapshot one successful and one failed ask invocation and validate
  both against the standard envelope schema.

### 15. Pad the table for markdownlint

- Thread: PRRT_kwDOTNkDkM6Oybd_
- Location: docs/issues/35-mcp-ask-tool.md:40
- Normative resolution: The Markdown table at this location has a blank line
  immediately before and after the table, with normal contiguous rows inside it.
- Focused gate: Run the repository Markdown lint command and assert that MD058 is
  absent for the MCP ask document.

### 16. Make the release flow fail-safe

- Thread: PRRT_kwDOTNkDkM6OybeC
- Location: docs/issues/41-packaging-release.md:45
- Normative resolution: Release execution builds and validates artifacts first,
  verifies the version tag, creates the GitHub release, and publishes only after
  those checks succeed. A failed tag or release step leaves the package
  unpublished and the recovery procedure retries the failed named step.
- Focused gate: Run the release workflow with a stubbed tag-creation failure and
  assert no publish call; run the success fixture and assert publish follows
  release creation.

### 17. Disable embeddings for --no-llm generation

- Thread: PRRT_kwDOTNkDkM6Oyces
- Location: docs/issues/25-generate-command.md:25
- Normative resolution: --no-llm sets both embeddings and index generation to
  disabled before pipeline construction, so the command performs no embedding
  provider or remote model request.
- Focused gate: Execute --no-llm with a provider that fails if called and assert
  successful local output, zero provider calls, and no index artifacts.

### 18. Wire embedding creation into the index hook

- Thread: PRRT_kwDOTNkDkM6Oycew
- Location: docs/issues/30-embedding-client.md:15
- Normative resolution: The index hook awaits ensureEmbeddings after the FTS index
  update and before reporting completion. The same hook owns the embedding error
  result and records it without leaving a falsely complete index.
- Focused gate: Run the indexing integration fixture and assert event order
  fts-updated -> embeddings-ensured -> index-complete, plus the provider call.

### 19. Use JavaScript-valid case-insensitive regexes

- Thread: PRRT_kwDOTNkDkM6Oycey
- Location: docs/issues/08-secret-filter.md:25
- Normative resolution: Secret-filter patterns are JavaScript regex literals with
  the trailing i flag and contain no unsupported inline modifier.
- Focused gate: Compile the exact patterns under Node and assert matching for
  lower-case and upper-case secret prefixes.

### 20. Preserve tracked files when applying gitignore rules

- Thread: PRRT_kwDOTNkDkM6Oyce1
- Location: docs/issues/07-file-scanner.md:28
- Normative resolution: The scanner obtains the tracked-file set from Git before
  applying ignore rules. A tracked file remains in the inventory even when ignored
  by .gitignore, and its sensitive-file flag is still evaluated.
- Focused gate: Track a .env fixture, add it to .gitignore, scan, and assert that
  the file is present and flagged.

### 21. Do not parallelize extractors before the schema exists

- Thread: PRRT_kwDOTNkDkM6Oyce3
- Location: docs/ISSUE_PLAN.md:121
- Normative resolution: Issue 10 establishes and validates the shared FileFacts
  schema before issue 11 and issue 12 extractor work begins; the Issue Plan
  dependency graph marks 10 as a prerequisite and does not call 11/12 parallel.
- Focused gate: Validate the dependency graph and assert that removing issue 10
  causes the plan check to fail for both extractor nodes.

### 22. Hash aggregate inventory from full fingerprints

- Thread: PRRT_kwDOTNkDkM6Oyce6
- Location: docs/issues/20-freshness-manifest.md:39
- Normative resolution: The aggregate inventory hash is computed from every full
  canonical record, including normalized path, content hash, byte size, and flags;
  it is never computed from content hashes alone.
- Focused gate: Change only a fixture's size or flags while retaining its content
  hash and assert that the aggregate fingerprint changes.

### 23. Keep slug ownership stable when collisions are added

- Thread: PRRT_kwDOTNkDkM6Oyce8
- Location: docs/issues/17-page-planner.md:35
- Normative resolution: The page planner persists slug ownership by source identity.
  An existing unsuffixed owner keeps its slug; a newly colliding source receives
  the next deterministic suffix and never usurps the existing owner.
- Focused gate: Plan once, add a colliding page, plan again, and assert the first
  page keeps slug and the new page receives a stable suffixed slug.

### 24. Account for the init probe fetch site

- Thread: PRRT_kwDOTNkDkM6Oyce_
- Location: docs/issues/21-llm-client.md:32
- Normative resolution: All model initialization probes and generation requests use
  the injected shared fetch transport. The request contract counts the init probe
  as a transport call and does not promise a single fetch site.
- Focused gate: Inject a fetch spy, initialize the client, and assert that both the
  health probe and generation request pass through the shared transport.

### 25. Make config loading compatible with pure resolver tests

- Thread: PRRT_kwDOTNkDkM6OycfB
- Location: docs/issues/13-import-resolver-module-graph.md:72
- Normative resolution: The resolver receives preloaded tsconfig data and has no
  filesystem dependency; config discovery and parsing occur in the caller before
  resolver invocation.
- Focused gate: Run resolver unit tests with filesystem access denied and assert
  deterministic resolution from the supplied config object.

### 26. Surface corrupt MCP search indexes explicitly

- Thread: PRRT_kwDOTNkDkM6OycfF
- Location: docs/issues/34-mcp-server-core.md:39
- Normative resolution: An unreadable or schema-invalid search index produces the
  structured error code index-corrupt, preserves the original index, and never
  deletes or silently rebuilds it during an ask request.
- Focused gate: Supply malformed and truncated index fixtures and assert the exact
  error code, preserved bytes, and absence of a delete call.

### 27. Include the file path in parse-cache keys

- Thread: PRRT_kwDOTNkDkM6OycfH
- Location: docs/issues/09-tree-sitter-host.md:48
- Normative resolution: The parse-cache key includes the normalized absolute file
  path in addition to grammar identity, parser version, and content digest.
- Focused gate: Parse identical content at two paths and assert two cache entries;
  parse the same path twice and assert a cache hit.

### 28. Resolve vendored grammars from the emitted dist depth

- Thread: PRRT_kwDOTNkDkM6OycfI
- Location: docs/issues/09-tree-sitter-host.md:58
- Normative resolution: The runtime grammar lookup derives the package root from
  the emitted module URL at dist/core/parse/host.js and resolves vendor grammars
  from package-root/vendor; it never resolves relative to process cwd.
- Focused gate: Run the compiled dist module from a different cwd and assert that
  the package-root vendor grammar loads successfully.

### 29. Add a Node shebang to the CLI bin

- Thread: PRRT_kwDOTNkDkM6OycfJ
- Location: docs/issues/01-project-scaffold.md:26
- Normative resolution: The generated CLI entry file begins with exactly
  #!/usr/bin/env node and is installed with executable mode.
- Focused gate: Build and invoke the bin directly without node preceding it, and
  assert the version output and executable mode.

### 30. Generate version.ts before typechecking

- Thread: PRRT_kwDOTNkDkM6OycfM
- Location: docs/issues/02-cli-skeleton-errors.md:26
- Normative resolution: The package validation pipeline runs version.ts generation
  before TypeScript typechecking; typecheck consumes the generated file from the
  same workspace.
- Focused gate: Delete the generated file, run the documented validation command,
  and assert generation occurs before tsc and the command exits successfully.

### 31. Avoid echoing LLM error bodies into user output

- Thread: PRRT_kwDOTNkDkM6OycfO
- Location: docs/issues/21-llm-client.md:42
- Normative resolution: User-visible LLM failures expose provider and HTTP status
  codes only. Response bodies are retained only in redacted diagnostic logs and
  are never copied to terminal, JSON, or Markdown output.
- Focused gate: Return an error body containing a sentinel secret and assert the
  sentinel is absent from every user renderer while provider/status remain present.

### 32. Hash LLM settings that change prose output

- Thread: PRRT_kwDOTNkDkM6OycfR
- Location: docs/issues/20-freshness-manifest.md:46
- Normative resolution: The freshness fingerprint includes a canonical object of
  every prose-affecting LLM setting: provider, model, temperature, maxInputTokens,
  system/instruction version, and prompt template version.
- Focused gate: Change each listed setting independently and assert a new
  fingerprint; reorder object keys and assert the fingerprint remains stable.

### 33. Exclude runtime built-ins from dependency externals

- Thread: PRRT_kwDOTNkDkM6OycfT
- Location: docs/issues/13-import-resolver-module-graph.md:60
- Normative resolution: External-package classification first removes Node runtime
  built-ins, including node:-prefixed names and the resolver's pathlib alias, from
  the external dependency set. Only remaining third-party package names are
  emitted as externals.
- Focused gate: Classify imports of node:fs, fs, os, path, and pathlib alongside a
  real package and assert only the real package is external.

## Resolution boundary

Each contract is intentionally limited to the referenced documentation behavior.
The implementation owner must update the affected design/issue text consistently,
run the focused gate and full repository validation, and obtain the required
security/privacy acceptance before resolving the mapped thread.
