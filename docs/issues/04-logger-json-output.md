# 04 — Structured logger and `--json` envelope

## Summary

Implement the shared logger (levels, secret redaction, verbosity flags) and the single helper that
emits the DESIGN §3.1 JSON envelope for all commands.

## Context

Commands must never hand-roll output: human mode writes progress to stderr and results to stdout;
`--json` mode writes exactly one envelope object to stdout. Logs must never contain file contents
or secret values (DESIGN §10 security gate, §13.3).

## Scope

- In: `src/shared/logger.ts`, `src/shared/output.ts` (success emission layered on issue 02's
  `output-core.ts` — the envelope builder and `emitError` already exist there), redaction
  helpers, stdout-guard installation in `src/cli/index.ts` (json mode).
- Out: error taxonomy and `buildEnvelope`/`emitError` (02), any command logic.

## Detailed Requirements

1. `logger.ts`: `createLogger({verbosity: "quiet"|"normal"|"verbose"})` with methods
   `debug/info/warn/error(msg, fields?)`. All logger output goes to **stderr** (stdout is reserved
   for results). `debug` only at verbose; `info` suppressed at quiet.
2. Formatting: `HH:MM:SS level message key=value…` in human mode; fields with value length > 200
   chars are truncated with `…(+N chars)`.
3. Redaction: `registerSecretValue(v: string)` — logger replaces registered values with
   `[REDACTED]` in messages and fields. Rules: values matched as exact literals (no regex
   interpretation — escape before matching), redaction applied BEFORE truncation (no leaked
   prefixes), empty/1-char registrations ignored, multiline values matched per whole value.
   Field names matching `/key|token|secret|password/i` are always redacted regardless of
   registration.
4. `LogFields` type: `Record<string, string | number | boolean>` — object/array values are
   rejected at runtime with a thrown TypeError in dev (test) to block payload dumping; module
   docs state the DESIGN §10 invariant (never log file contents, prompts, or responses) and
   name the enforcing tests (issue 21 logger hook, issue 39 canary sweep).
5. `output.ts`: `emitResult(ctx: OutputContext, {data, warnings, render})` where
   `OutputContext = {command: string, json: boolean, verbosity}` (provided by 02's CLI context):
   - `--json`: writes 02's `buildEnvelope({ok:true, command, data, errors:[], warnings})`
     (single line + newline, fixed key order) to stdout; nothing else may write to stdout in
     json mode — this issue installs the process-level stdout guard in the CLI entry (any other
     stdout write throws in tests).
   - human: calls the command-provided `render(data)` callback to pretty-print to stdout.
   - `warnings` is `string[]` of stable kebab-case tokens (e.g. `budget-exhausted`); hint text
     for humans goes through the logger, never into the warnings array.
6. Progress helper `logger.progress(label, current, total)` → single rewritten stderr line on TTY,
   plain lines when not a TTY (CI-safe).

## Acceptance Criteria

- [ ] Unit tests: level filtering, quiet/verbose behavior, truncation; redaction of registered
      values (long secret, regex-metacharacter value, multiline value, value inside a field) —
      all before truncation; sensitive field names always redacted.
- [ ] `LogFields` object value throws (runtime guard test).
- [ ] JSON mode: exactly one line on stdout, parseable, key order matches 02's envelope; logger
      writes only to stderr; stdout guard throws on a stray `console.log` (test).
- [ ] Non-TTY progress produces plain sequential lines.

## Validation

```bash
npm run test -- shared/logger shared/output
```

## Dependencies

02.

## Non-goals

Log files on disk (not in v1 — DESIGN §5 storage table contains no log directory), telemetry
(never).

## Design References

- DESIGN §3.1 (envelope), §10 (no payload logging), §13.3 (redaction)
