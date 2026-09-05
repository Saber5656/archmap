# 02 — CLI skeleton, error taxonomy, exit codes

## Summary

Implement the `archmap` CLI entrypoint with commander: all six subcommands registered as stubs,
global flags, centralized error handling, the full error/exit-code taxonomy from DESIGN §14, and
`--version`.

## Context

Every command issue (06, 25, 26, 32, 38, and MCP 34) plugs into this skeleton. Locking exit codes
and error shapes now prevents divergent conventions later (DESIGN §3.1, §14).

## Scope

- In: `src/cli/index.ts` (program), `src/cli/context.ts` (shared command context),
  `src/shared/errors.ts`, `src/shared/output-core.ts` (minimal envelope builder + error emitter;
  issue 04 layers success emission and the logger on top of it), a `prebuild` version-embed
  script, stub files `src/cli/{init,generate,check,ask,serve,mcp}.ts`.
- Out: real command logic; logger and human/success output rendering (issue 04).

## Detailed Requirements

1. `src/cli/index.ts`: commander program `archmap`; subcommands: `init`, `generate`, `check`,
   `ask <question>`, `serve`, `mcp`. Version: a `prebuild` npm script generates
   `src/shared/version.ts` (`export const TOOL_VERSION = "…"`) from package.json; `--version`
   prints it; `dist/` performs no runtime read of package.json (test asserts no `readFile` of it).
2. Global options (available to all subcommands, parsed before command action):
   `--config <path>`, `--repo <path>`, `--verbose`, `--quiet`, `--json`.
   `--verbose` and `--quiet` are mutually exclusive → usage error.
3. `src/cli/context.ts`: `resolveRepoRoot(opts): string` — run
   `git rev-parse --show-toplevel` via `execFile` (no shell) with cwd = `--repo` value when
   given, else process cwd; return the absolute canonical toplevel path. Non-existent `--repo`
   path, or git failure (not a repo) → `E_NOT_A_REPO` (exit 2) with hint. A `--repo` pointing
   into a subdirectory therefore still yields the true repo root. Context carries
   `{repoRoot, configPath, command, json, verbosity}`.
4. `src/shared/errors.ts`:
   - `class ArchmapError extends Error { code: ErrorCode; exitCode: number; hint?: string;
     path?: string }` (per DESIGN §14 error object). Every user-facing throw site must provide
     an actionable `hint`.
   - `ErrorCode` union and catalog table mapping code → exit code, per DESIGN §14:
     `E_INTERNAL`(1), `E_CONFIG_INVALID`(2), `E_USAGE`(2), `E_NOT_A_REPO`(2), `E_STALE`(3),
     `E_ARTIFACTS_MISSING`(4), `E_LLM_UNAVAILABLE`(5), `E_SECURITY_REMOTE_BLOCKED`(6),
     `E_SECURITY_KEY_IN_CONFIG`(6). Exported `EXIT_CODES` const object.
5. `src/shared/output-core.ts`: `buildEnvelope({ok, command, data, errors, warnings})` with the
   fixed key order of DESIGN §3.1, serialized as single-line JSON + newline;
   `emitError(ctx, ArchmapError)` — human mode: stderr `error[E_CODE]: message` + optional
   `hint: …` line (+ `path: …` when set); json mode: envelope
   `{ok:false, command, data:null, errors:[{code,message,hint?,path?}], warnings:[]}` on stdout.
   Issue 04's `emitResult` reuses `buildEnvelope` (single envelope implementation).
6. Central handler wrapping BOTH command actions and commander parsing: install commander
   `exitOverride()` and suppress its direct output; parse errors (unknown flag/subcommand,
   missing arg, `--verbose --quiet` conflict) are mapped to `ArchmapError(E_USAGE)` and routed
   through `emitError` like any action error. Unknown thrown errors → `E_INTERNAL`, stack to
   stderr only with `--verbose`. Process exits with the mapped code in all modes.
7. Command lifecycle (stubs): `--help`/`--version` resolve no repo context. Every real
   subcommand action first builds the context (repo resolution — failing `E_NOT_A_REPO` before
   any command logic), then the stub emits a `not-implemented` notice: exit 0, human = stderr
   notice line, json = success envelope with `warnings:["not-implemented"]`. (Stubs are replaced
   by later issues; the lifecycle order is permanent.)

## Acceptance Criteria

- [ ] `archmap --version` prints the package version; `archmap --help` lists all six commands.
- [ ] `archmap generate` outside a git repo exits 2 with `error[E_NOT_A_REPO]` and a hint.
- [ ] `archmap generate --json` outside a git repo emits the JSON error envelope on stdout, exit 2.
- [ ] `--verbose --quiet` together → exit 2 usage error.
- [ ] Unit tests cover: exit-code mapping for every ErrorCode, JSON envelope shape (fixed key
      order), at least one path-bearing error serialization, repo-root resolution (tempdir
      with/without `.git`; `--repo` pointing at a subdirectory returns the toplevel).
- [ ] Commander parse errors (unknown flag, unknown subcommand) produce the standard error
      output in both human and `--json` modes — no commander default output, no stack trace.
- [ ] `--version` works from `dist/` without reading package.json at runtime.
- [ ] No `child_process.exec`/shell string interpolation anywhere (`execFile` only).

## Validation

```bash
npm run test -- cli
node dist/cli/index.js --help
cd "$(mktemp -d)" && node <repo>/dist/cli/index.js check; echo "exit=$?"   # expect 2
```

## Dependencies

01.

## Non-goals

Actual command behavior; config loading (03); logger implementation (04).

## Design References

- DESIGN §3.1 (commands/flags/envelope), §14 (error taxonomy)
