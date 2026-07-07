# 06 — `archmap init` scaffolding command

## Summary

Implement `archmap init`: create `archmap.config.json` and `.archmapignore`, append `.archmap/` to
`.gitignore`, detect repo languages to prefill `analysis.languages`, and probe a local Ollama to
propose `llm.model`.

## Context

`npx archmap init` is the zero-to-configured path (DESIGN §3.1, §3.2 workflow 1). The model field
is required-when-enabled but has no hardcoded default (models age fast), so init fills it by
probing the local runtime (ADR-001 keeps this loopback-only).

## Scope

- In: `src/cli/init.ts`, `src/core/init/detect.ts`, `src/core/init/ollama-probe.ts`, template for
  `.archmapignore`.
- Out: scanning internals (07 — detection here is a cheap extension census, not the scanner),
  generate pipeline.

## Detailed Requirements

1. Behavior when `archmap.config.json` already exists: exit 2 `E_USAGE` with hint
   `config exists; edit it directly or delete it to re-init` (never overwrite).
2. Language detection: walk tracked files via `git ls-files -z` (execFile); count extensions;
   include `"typescript"` if any `.ts/.tsx`, `"javascript"` if any `.js/.jsx/.mjs/.cjs`,
   `"python"` if any `.py`; result → `analysis.languages` (order fixed: typescript, javascript,
   python). Empty intersection → keep schema default and warn `no-supported-languages`.
3. Ollama probe (`ollama-probe.ts`): `GET http://localhost:11434/api/tags`, timeout 1500 ms,
   fetch with `redirect: "error"` (a redirect must abort the probe — the only permitted network
   destination is the loopback URL itself, DESIGN §13.2).
   - Expected response shape: HTTP 200 with JSON `{models: [{name: string, …}]}` (Ollama
     `/api/tags`). Non-200, malformed JSON, or missing `models` array → treated as probe
     failure (below) with warning `ollama-probe-failed`.
   - Reachable with usable models: choose the first available model matching preference order
     `["qwen3", "llama3", "gemma3", "deepseek-r1"]` by name prefix (pick the largest tag within
     the first matching family); set `llm.model` to the exact tag string; also verify
     `embeddings.model` (`nomic-embed-text*`) is present, else warn `embedding-model-missing`
     (human hint shows the `ollama pull nomic-embed-text` command).
   - Reachable but NO preferred chat-family model present (e.g. only an embedding model) →
     `llm.enabled: false` + warning `llm-model-missing` (human hint lists detected models and
     a pull suggestion). Never write a made-up model name.
   - Unreachable / probe failure: set `llm.enabled: false` in the written config and warn
     `ollama-not-found` (hint: install/start Ollama or edit `llm.baseUrl`; generate still works
     structurally until enabled).
4. Write `archmap.config.json` (canonical pretty JSON, `$schema` pointing to the repo-published
   schema URL per DESIGN §6) containing ONLY keys that differ from defaults plus `$schema`,
   `llm.model`, and `analysis.languages` — minimal diff-friendly config.
5. Write `.archmapignore` template (comments + common entries: `dist/`, `build/`, `coverage/`,
   `*.min.*`, `vendor/`). If `.archmapignore` already exists: leave it untouched and report it
   as `unchanged` (never overwrite user rules).
6. `.gitignore`: append `.archmap/` if not already present (create file if missing; preserve
   existing content; exactly one trailing newline).
7. `--yes`: no prompts (v1 has no interactive prompts anyway — flag reserved and accepted).
   Idempotency: running init twice → second run exits 2 without modifying anything.
8. Output — human: summary table (config path, languages, model or disabled, files
   written/unchanged, hint lines for warnings). `--json` `data` payload (exact schema):
   `{configPath: string, languages: string[], llm: {enabled: boolean, model: string|null},
   files: [{path: string, action: "created"|"appended"|"unchanged"}]}`;
   `warnings` = the kebab-case tokens defined above (hints are human-mode only, per issue 04).

## Acceptance Criteria

- [ ] In a fixture repo with `.ts` + `.py` files and a mocked Ollama `/api/tags` (msw or local
      http server in tests) returning `qwen3:8b` and `nomic-embed-text`, init writes config with
      those values and both ignore files; second run exits 2 and changes nothing (mtime/content).
- [ ] With Ollama unreachable, config has `llm.enabled: false`, warning emitted, exit 0.
- [ ] Probe edge matrix: 200-but-no-chat-model → `llm-model-missing` + enabled:false;
      malformed JSON / non-200 → `ollama-probe-failed` + enabled:false; 302 redirect to a
      non-loopback URL → probe aborts, no outbound remote request (request capture).
- [ ] Pre-existing `.archmapignore` untouched and reported `unchanged`.
- [ ] Written config (including `$schema`) passes `loadConfig` round-trip; `.gitignore` gains
      `.archmap/` exactly once across repeated manual appends.
- [ ] No network call other than the loopback probe (test asserts no other fetch).

## Validation

```bash
npm run test -- cli/init
npm run build
set -o pipefail
tmp=$(mktemp -d) && cp -R tests/fixtures/fixture-mixed/. "$tmp" && git -C "$tmp" init -q
(cd "$tmp" && node "$OLDPWD/dist/cli/index.js" init --json | jq -e '.ok == true')
```

## Dependencies

03, 04.

## Non-goals

Interactive prompt UX; probing non-Ollama runtimes (config is hand-editable for those).

## Design References

- DESIGN §3.1 (init row), §6 (config), §13.2 (loopback-only probe); ADR-001
