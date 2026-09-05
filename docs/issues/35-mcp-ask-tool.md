# 35 — MCP `ask_question` tool + client setup docs

## Summary

Add the `ask_question` tool to the MCP server, reusing the ask pipeline (31/32) with MCP-shaped
errors and timeouts, and write the agent-client setup documentation (Claude Code, Codex CLI,
Cursor).

## Context

This completes DeepWiki tool-name parity and is the highest-leverage agent feature: grounded
answers about private code, fully local. Setup docs are part of the issue because P3's value dies
without frictionless registration.

## Scope

- In: `src/mcp/tools/ask.ts`, `docs/guides/mcp-clients.md`, server registration.
- Out: ask internals (32), general MCP docs polish (42 links to the guide).

## Detailed Requirements

1. Network exception (locked; the ONLY networked MCP tool): `ask_question` reaches exactly the
   configured LLM/embeddings endpoints through the existing clients (21/30) — gate, redirect
   blocking, and redaction included; every other tool remains network-free (34's fetch-spy test
   extended to prove ask-only egress). A gate refusal performs zero network attempts.
2. Tool `ask_question` `{question: string (1..2000)}` → the canonical AnswerRecord fields
   (DESIGN §7.7: `question`, `answer`, `citations`, `insufficientEvidence`, `model`,
   `retrieval`) plus `notice` (34's fixed untrusted-content string); `answer` is sanitized via
   `sanitizeProse` and then sentinel-wrapped with 22's `fenceData` (same treatment as
   `read_wiki_contents`).
3. Precondition matrix (locked structured error codes, checked per-call — the LLM may come up
   mid-session):
   | Condition | code | hint |
   |---|---|---|
   | `ask.enabled: false` | `ask-disabled` | set ask.enabled true, re-run generate |
   | index missing/corrupt | `index-missing` | run archmap generate (with ask.enabled: true) |
   | `llm.enabled: false` | `llm-disabled` | enable llm in config |
   | config invalid (model null, key env missing) | `config-invalid` | from 21's error |
   | endpoint down/auth failed | `llm-unavailable` | includes baseUrl host |
   | non-loopback w/o opt-in | `security-remote-blocked` | allowRemoteLlm opt-in |
4. Deadline: ONE shared outer deadline per call = `llm.requestTimeoutMs + 10_000` ms covering
   retrieval + answer generation; the MCP path constructs its client with retries DISABLED
   (single attempt) and propagates an AbortSignal; deadline breach → structured `timeout`
   error, request aborted, server stays healthy.
5. Concurrency (locked): 1 active + max 4 queued; the 6th concurrent call returns `busy`.
6. `docs/guides/mcp-clients.md`: registration snippets in two tiers —
   development/dogfood (testable at this issue's time): Claude Code
   `claude mcp add archmap -- node <repo>/dist/cli/index.js mcp`, Codex CLI TOML and Cursor
   `.cursor/mcp.json` equivalents using the same command; published form (`npx archmap mcp`)
   shown with an "after first npm release (issue 41)" note. Plus: tool list, untrusted-content
   notice semantics, and a worked example session (structure → contents → ask).
7. Smoke rig (34's script) extended with an ask call behind `--with-ask` flag (mock LLM env
   var `ARCHMAP_MCP_SMOKE_LLM=mock` starts the in-process mock from 27).

## Acceptance Criteria

- [ ] Protocol test: happy path returns the canonical AnswerRecord fields + notice, with
      sanitized sentinel-wrapped `answer`; citations only from retrieved set (reuses 32
      validators — integration proven).
- [ ] Every precondition matrix row returns its exact structured code (6-case test).
- [ ] Egress: full session fetch-spy shows network only for `ask_question` and only to the
      configured loopback endpoint; gate refusal shows zero attempts.
- [ ] Timeout injection (mock latency > deadline) → `timeout` error, single attempt (no
      retries observed), subsequent calls succeed.
- [ ] Concurrency boundary: 6 parallel calls → 1 active, 4 queued, 6th gets `busy` (exact).
- [ ] Guide tested by following the dev-tier registration verbatim for Claude Code + one other
      client on the dogfood wiki (recorded as evidence in the guide's footer: date + client
      versions).

## Validation

```bash
npm run test -- mcp/ask
ARCHMAP_MCP_SMOKE_LLM=mock node scripts/mcp-smoke.mjs --with-ask
```

## Dependencies

32, 34.

## Non-goals

Multi-turn conversations, per-client custom tool naming, remote transports.

## Design References

- DESIGN §12.1 (ask row), §13.4; ADR-003 (citation validity)
