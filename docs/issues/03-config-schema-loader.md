# 03 — Config schema, loader, JSON-schema export

## Summary

Implement the `archmap.config.json` zod schema exactly as DESIGN §6, the loader with precedence
(CLI > env > file > defaults), strict unknown-key rejection with did-you-mean hints,
key-in-config secret detection, and JSON-schema export to `schema/archmap.schema.json`.
(Runtime loopback enforcement for `baseUrl`s is issue 21's `assertEndpointAllowed`; this issue
only validates URL shape.)

## Context

Config is consumed by every subsystem; the security posture (ADR-001) is enforced first here.
DESIGN §6 is the single source of truth for keys, defaults, and hard rules.

## Scope

- In: `src/config/schema.ts`, `src/config/load.ts`, `src/config/env.ts`,
  `scripts/export-json-schema.ts` (build-time), `schema/archmap.schema.json` (generated, committed).
- Out: consuming config in commands; init scaffolding (06); the network loopback check itself at
  request time (issue 21 — but the *shape* validation lives here).

## Detailed Requirements

1. `schema.ts`: zod object mirroring DESIGN §6 exactly — keys, types, defaults:
   `output.dir="docs/wiki"`, `output.language="en"`, `analysis.include=[]`, `analysis.exclude=[]`,
   `analysis.languages=["typescript","javascript","python"]`, `analysis.maxFileKb=512`,
   `analysis.churnDays=180` (integer ≥ 0; `0` = churn disabled, semantic implemented by issue
   15), `llm.enabled=true`, `llm.baseUrl="http://localhost:11434/v1"`,
   `llm.model=null`, `llm.apiKeyEnv=null`, `llm.temperature=0.2`,
   `llm.maxInputTokensPerRun=2000000`, `llm.requestTimeoutMs=120000`, `llm.concurrency=2`,
   `embeddings.{enabled=true,model="nomic-embed-text",baseUrl=null}`,
   `ask.{enabled=true,topKChunks=12,ftsCandidates=500}`,
   `security.{allowRemoteLlm=false,secretScan=true,secretPolicy="exclude"}`,
   `viewer.port=4640`, `wiki.{maxModulePages=40,maxDiagramNodes=30}`.
   Export TypeScript type `ArchmapConfig = z.infer<…>`.
   Additionally accept an optional top-level `$schema: string` (the file `init` writes carries
   it); it is stripped from the returned `ArchmapConfig`.
   `llm.model: null` is VALID at load time even with `llm.enabled: true` — the "model required
   when LLM is actually used" check is enforced by issue 21's `createLlmClient` (raises
   `E_CONFIG_INVALID` with hint `set llm.model — run archmap init to detect one`); documented
   here so 03/06/21 share one contract.
2. Strictness: `.strict()` at every object level. On unknown key, error message must include the
   nearest valid key by Levenshtein distance ≤ 3 (`did you mean "llm.baseUrl"?`).
3. Validations:
   - `llm.apiKeyEnv` must match `/^[A-Z][A-Z0-9_]*$/` when set (an env var *name*); violation →
     `E_CONFIG_INVALID` (exit 2) with path + hint.
   - Any config string value matching secret shapes (`sk-[A-Za-z0-9]{20,}`, `AKIA[0-9A-Z]{16}`,
     PEM header, `ghp_[A-Za-z0-9]{36}`) → `E_SECURITY_KEY_IN_CONFIG` (exit 6) naming the
     offending path, value never echoed. This scan runs on EACH source separately (file values,
     `ARCHMAP_*` env override values, CLI override values) BEFORE precedence merging, so a
     committed secret is caught even when overridden. The process env value referenced by
     `apiKeyEnv` is never scanned or serialized.
   - `llm.baseUrl`/`embeddings.baseUrl` must parse as http(s) URLs (shape only; loopback
     enforcement is issue 21).
4. `load.ts`: `loadConfig({repoRoot, configPath?, cliOverrides?}): ArchmapConfig`.
   - File: `configPath` or `<repoRoot>/archmap.config.json`; missing file → defaults (valid);
     malformed JSON → `E_CONFIG_INVALID` with line/column.
   - Env overrides (`env.ts`): `ARCHMAP_<SECTION>_<KEY>` upper-snake maps to config path
     (e.g. `ARCHMAP_LLM_BASE_URL` → `llm.baseUrl`, `ARCHMAP_OUTPUT_DIR` → `output.dir`).
     Value grammar: booleans `"true"/"false"`; numbers via `Number` (NaN →
     `E_CONFIG_INVALID`); string-array keys (`analysis.include/exclude/languages`) split on
     `,` with trimming (empty string → `[]`); nullable strings accept the literal `"null"`;
     empty value for non-array keys → `E_CONFIG_INVALID`. Unknown `ARCHMAP_*` names →
     `E_CONFIG_INVALID` with the same did-you-mean hint mechanism. Document the full mapping
     table in module JSDoc.
   - CLI overrides passed as a partial object by commands (e.g. `--no-llm` → `llm.enabled=false`).
   - Precedence: defaults < file < env < CLI. Merged result validated once against the schema
     (in addition to the per-source secret scan above).
5. `output.dir` must resolve inside `repoRoot` after normalization (no `..` escape) →
   `E_CONFIG_INVALID` otherwise (DESIGN §5 confinement). Implemented INLINE here with the same
   algorithm issue 05 later centralizes (`path.resolve` + `path.relative` boundary check — no
   `startsWith`); this keeps 03 independent of 05 in the execution order.
6. JSON-schema export: build script converts the zod schema (via `zod-to-json-schema`) to
   `schema/archmap.schema.json`; `npm run build` regenerates it; CI fails if the committed file
   is out of date (compare in test).

## Acceptance Criteria

- [ ] All defaults load with no config file; result equals the DESIGN §6 table; a config file
      containing `$schema` round-trips (accepted, stripped from the result).
- [ ] `llm.model: null` with `llm.enabled: true` loads successfully (21 owns the late check).
- [ ] Unknown key `llm.baseURL` errors with hint `did you mean "llm.baseUrl"?`; unknown
      `ARCHMAP_LLM_BASEURL` env var errors likewise.
- [ ] `ARCHMAP_LLM_BASE_URL=http://x` overrides file value; CLI override beats env (test proves
      ordering); array override `ARCHMAP_ANALYSIS_LANGUAGES=typescript,python` works; nullable
      override `ARCHMAP_EMBEDDINGS_BASE_URL=null` yields null.
- [ ] Config containing `"apiKeyEnv": "sk-abc…"`-style literal or an AWS key anywhere exits 6
      without printing the secret value — including when an env/CLI override replaces the
      offending path (per-source scan proof). Malformed `apiKeyEnv` name (`"my key"`) exits 2.
- [ ] `output.dir: "../../etc"` is rejected; `llm.baseUrl: "not a url"` is rejected (shape).
- [ ] `schema/archmap.schema.json` is committed, regenerated by build, and drift-tested.

## Validation

```bash
npm run test -- config
npm run build && git diff --exit-code schema/archmap.schema.json
```

## Dependencies

02.

## Non-goals

Runtime loopback enforcement (21), init file writing (06).

## Design References

- DESIGN §6 (full contract), §5 (confinement), §13.3, §14; ADR-001
