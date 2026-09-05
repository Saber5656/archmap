# 39 — Security red-team pass with attack fixtures

## Summary

Execute DESIGN §13.7 as automated tests: secret-canary leak sweep across every artifact, prompt
injection end-to-end on `fixture-inject`, traversal corpora against viewer + MCP, and the
network-egress assertion — the gate for wave 5.

## Context

Individual issues implemented their local mitigations; this issue proves the composed system
holds the threat-model promises (B1–B4) and pins them against regression. OSS release (41) is
blocked on this.

## Scope

- In: `tests/security/*.test.ts`, fixture finalization (`fixture-secrets`, `fixture-inject`
  extensions), `tests/helpers/egress-guard.ts`, SECURITY acceptance summary
  `docs/security/REDTEAM.md` (what is tested, how, result matrix).
- Out: new mitigations (any failure here reopens the owning issue; fixes land there).

## Detailed Requirements

1. Canary sweep (B2): run full `generate` (mock LLM with embeddings enabled, both secret
   policies) + ask-index build on `fixture-secrets`; assert canary strings (imported from
   `tests/helpers/canaries.ts`, issue 08) appear in NONE of — wiki files, nav/manifest,
   `.archmap/cache/**`, `index.db` (dump via SELECT), mock-LLM captured request bodies for
   BOTH `/v1/chat/completions` AND `/v1/embeddings`, process stdout/stderr captures.
   `redact` policy: `[REDACTED:` markers present where files were kept.
2. Injection E2E (B3) — string-level, testable expectations (sanitizers are structural
   whitelists, NOT semantic instruction detectors; echoed instruction TEXT as inert prose is
   acceptable): `fixture-inject` payloads (instruction-override text, fake ARCHMAP_DATA
   sentinels, mermaid breakout labels, HTML/script in docstrings, malicious relative/absolute
   links in README):
   - generate with SCRIPTED mock LLM that ECHOES injected payloads into prose fields → final
     wiki contains NO active content from them: no HTML elements, no event handlers, no
     absolute/scheme links, no image syntax, no live `click`/`%%` in mermaid output (token
     assertions); all mermaid blocks still parse; pages render.
   - captured prompts: every payload occurrence sits INSIDE matched sentinel fences (fence
     integrity despite fake sentinels in content).
   - MCP `read_wiki_contents` + `search_wiki` on the poisoned page → sentinels/notice intact.
3. Traversal (B4): shared corpus module (`../`, `..%2f`, `%2e%2e%5c`, double-encode, null byte,
   absolute, UNC-style, symlink-in-root) run against BOTH viewer routes and MCP slug/cluster
   inputs; assert 404/structured-error + zero out-of-root fs reads (fs spy).
4. Egress guard (B1): `egress-guard.ts` monkey-patches `net.Socket.connect`/`fetch` dispatcher
   INSIDE the archmap process only (the test's own HTTP client stays outside the guard;
   loopback inbound connections to the viewer are not egress): full `generate --no-llm`,
   `check`, `mcp` session, and a `serve`+browse session → zero non-allowlisted outbound
   attempts from archmap; `generate` with mock LLM → only the mock's loopback address observed.
5. Config attack surface: config files attempting `output.dir: "../x"`, key-shaped values, and
   `ARCHMAP_LLM_BASE_URL=http://evil.example` env with `allowRemoteLlm=false` → all blocked with
   exit 6/2 (matrix reusing 03/21 units at the CLI level).
6. `docs/security/REDTEAM.md`: table mapping §13 boundary → tests → status; instructions to
   re-run locally (`npm run test:security`); explicit "not covered" list (e.g., malicious
   dependency at build time, OS-level attacker) for honesty.
7. CI: `npm run test:security` job added; failing = red PR.

## Acceptance Criteria

- [ ] All four suites green in CI on both OS runners; each suite fails when its mitigation is
      manually broken (mutation check documented for one example per suite — proves the tests
      actually bite).
- [ ] REDTEAM.md matrix: B1–B4 rows each have ≥ 1 test reference; the B5 row explicitly reads
      `deferred to issue 41 (supply-chain gates)`.
- [ ] Canary list is single-sourced from 08 (import, not copy); embeddings request bodies are
      part of the sweep.
- [ ] Egress guard catches an intentionally added `fetch("https://example.com")` (self-test).

## Validation

```bash
npm run test:security
```

## Dependencies

25, 26, 27, 32, 35, 37, 38.

## Non-goals

Pen-testing the host OS, dependency audit automation (41), fuzzing (v2 candidate).

## Design References

- DESIGN §13 (entire), §16 (security row)
