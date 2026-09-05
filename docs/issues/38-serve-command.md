# 38 — `archmap serve` command polish

## Summary

Wire viewer server + UI into `archmap serve`: artifact preflight, port selection with fallback,
`--open`, graceful shutdown, and clear terminal output.

## Context

Last-mile UX for pillar P1's reading experience (DESIGN §3.1 serve row). Keeps 36/37 pure by
owning all CLI concerns here.

## Scope

- In: `src/cli/serve.ts`.
- Out: server internals (36), UI (37).

## Detailed Requirements

1. Preflight: wiki manifest + nav must exist → else exit 4 with hint `run archmap generate`.
   Staleness warning (not failure): run the shared freshness path (25's `evaluateFreshness` +
   26's current-version derivation) inside `Promise.race` with a 2 s timer; on timeout, start
   with NO warning (serve must start fast); on stale result, warn
   `wiki may be stale (N pages) — run archmap generate` (warning token `wiki-stale`).
2. Port (locked validation): effective port must be an integer 1..65535 — invalid `--port`
   value → `E_USAGE` exit 2; invalid config `viewer.port` → `E_CONFIG_INVALID` exit 2 (03's
   schema bounds it, defense here for CLI input). Config-derived port busy → try next port up
   to +10 with notice (warning token `port-fallback`); explicit `--port` busy → fail `E_USAGE`
   (no auto-fallback when the user pinned it).
3. `--open`: open `http://127.0.0.1:<port>/` via platform opener (`open`/`xdg-open`; `start` on
   win32) using `execFile`, failure → warn only (token `open-failed`).
4. Output — human: startup banner `archmap wiki → http://127.0.0.1:4640 (Ctrl+C to stop)` +
   page count + generated date (stderr/logger; stdout stays clean). `--json`: exactly ONE
   shared envelope line via 04's `emitResult` — `{ok: true, command: "serve",
   data: {url, port, pages, generatedAt}, errors: [], warnings: [wiki-stale?, port-fallback?,
   open-failed?]}` — then the process keeps serving (documented: json mode still blocks; for
   scripts, run in background). All subsequent output goes to stderr.
5. Shutdown: SIGINT/SIGTERM → `close()` (stop accepting, ≤ 2 s drain), exit 0; second SIGINT
   during drain → immediate exit 130.
6. No watch/reload in v1: page reads hit disk per request (36 streams from fs), so a manual
   regenerate is visible on browser refresh without server restart — stated in banner docs.
7. Loopback assertion (DESIGN §13.5): serve constructs the server without any host parameter
   (36's API has none) and the emitted/opened URL is exactly `http://127.0.0.1:<port>/`; no
   `--host` flag exists.

## Acceptance Criteria

- [ ] No wiki → exit 4 with hint; stale wiki → `wiki-stale` warning then serves; evaluator
      timeout (injected slow fs) → serves with no warning.
- [ ] Default port busy (test occupies 4640) → serves on 4641 with `port-fallback`;
      `--port 4640` busy → exit 2; `--port 0` / `--port abc` / `--port 70000` → exit 2.
- [ ] `--json` emits exactly one shared-envelope line on stdout (parse + key-order test) with
      the applicable warning tokens; server still serving afterward (health probe).
- [ ] SIGINT closes cleanly (exit 0, port released — asserted by rebind); SIGTERM likewise;
      second SIGINT during drain → exit 130.
- [ ] `--open` invoked with exactly `http://127.0.0.1:<port>/` (execFile spy); failure path
      warns and serves; no host option reaches the server factory (API-shape assertion).
- [ ] Regenerate-while-serving reflects on refresh (integration test touching wiki file).
- [ ] This issue adds the viewer first-page-load row to `scripts/bench.ts` (replacing 27's
      `skipped (issue 38)` row; informational).

## Validation

```bash
npm run test -- cli/serve
node dist/cli/index.js serve --json &   # then curl /api/health
```

## Dependencies

26 (freshness path for the stale warning), 37.

## Non-goals

Live-reload websockets, non-loopback hosting, daemonization.

## Design References

- DESIGN §3.1 (serve), §12.2, §13.5 (loopback-only), §14 (exit codes)
