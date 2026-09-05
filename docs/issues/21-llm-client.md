# 21 — OpenAI-compatible LLM client with security gate and budget

## Summary

Implement the single LLM chat client per DESIGN §10: OpenAI-compatible `chat/completions` over
fetch, loopback security gate (ADR-001), env-indirected API key, retries, timeouts, token
estimation, and per-run budget accounting.

## Context

This is trust boundary B1 — the only place archmap talks to a network. Everything LLM-flavored
(23, 24, 30, 32) goes through this client; the gate and budget are non-bypassable by construction
(module owns the only fetch call).

## Scope

- In: `src/llm/transport.ts` (shared OpenAI-compatible request helper — the only fetch call
  site), `src/llm/client.ts`, `src/llm/budget.ts`, `src/llm/security-gate.ts`.
- Out: prompts (22), the embeddings endpoint call itself (30 — but it MUST go through this
  issue's transport).

## Detailed Requirements

1. `security-gate.ts`: `assertEndpointAllowed(baseUrl, security)`:
   - Parse URL; allowed hosts when `allowRemoteLlm=false`: `localhost`, `127.0.0.1`, `::1`
     (exact host match; any port). Everything else → `E_SECURITY_REMOTE_BLOCKED` (exit 6) with
     hint naming the host and the config key to opt in.
   - When `allowRemoteLlm=true` and host non-loopback: log ONE `warn` per process naming host.
   - Non-http(s) schemes rejected. Called at client construction AND before every request
     (defense in depth).
2. Shared transport (`transport.ts`, exported — the ONLY fetch call site in the codebase;
   issue 30 MUST use it for `/embeddings`): `openAiRequest({baseUrl, path, body, apiKeyEnv,
   timeoutMs, logger})`:
   - Gate check (req 1) before every request; `redirect: "error"` on EVERY fetch — a loopback
     endpoint redirecting anywhere (even loopback) aborts the request (B1: only the configured
     destination is ever contacted).
   - API key: `apiKeyEnv` → `process.env[name]`; missing named var → `E_CONFIG_INVALID` with
     hint; key registered with logger redaction (issue 04) and sent as `Authorization: Bearer`;
     no header when null (Ollama).
   - Timeout via AbortController; retries: 2 on network error/5xx/429 with jittered backoff
     (500·2^n ms ±25%); 4xx (non-429) fail fast `E_LLM_UNAVAILABLE` including server error body
     first 200 chars (sanitized to single line).
3. `client.ts`: `createLlmClient(config.llm, security, logger, budget?)` →
   `complete({system, user, label}): Promise<{text, usage}>`:
   - Constructor validation: when `llm.enabled` and (`llm.model` is null or empty) →
     `E_CONFIG_INVALID` with hint `set llm.model — run archmap init to detect one` (the late
     check promised by issue 03).
   - `POST {baseUrl}/chat/completions` body `{model, messages, temperature,
     response_format: {type: "json_object"}}` (tolerate servers ignoring response_format).
   - Budget enforcement INSIDE `complete()` when a budget is attached (generate always attaches
     one; `ask` runs budget-less): `tryConsume(estimateTokens(system + user))` before the
     request — refusal throws typed `BudgetExhaustedError` that callers map to their
     degradation path. Server-reported `usage` accumulated into the budget's usage totals.
   - `label` is a short caller tag (e.g. page slug) used ONLY for metadata logging.
   - `preflight()`: `GET {baseUrl}/models`; statuses 404/405/501 → fallback harmless completion
     `{model, messages: [{role: "user", content: "ping"}], max_tokens: 1, temperature: 0}`;
     auth/security statuses (401/403) fail fast, NO fallback. Failure → `E_LLM_UNAVAILABLE`
     exit 5 with actionable hint (is Ollama running? `ollama serve`).
   - Concurrency limiter: at most `config.llm.concurrency` in-flight completions (semaphore;
     shared client instance per run).
4. `budget.ts`: `createBudget(maxInputTokensPerRun)`:
   - `estimateTokens(text) = ceil(text.length / 3.6)` (documented heuristic const).
   - `tryConsume(tokens): boolean` — false once cap would be exceeded; thereafter
     `budget.exhausted = true`; caller-visible remaining count; synchronous check-and-set.
   - Usage totals (`promptTokens`, `completionTokens`, `calls`) exposed for the run report.
5. Loopback rule (locked): compare normalized `URL.hostname` (IPv6 brackets stripped) against
   `localhost`, `127.0.0.1`, `::1`; any port allowed.
6. Never log prompt/response bodies at any verbosity (metadata only: sizes, duration, model,
   label). Enforced by unit test hooking the logger.

## Acceptance Criteria

- [ ] Gate: `http://192.168.1.10:11434/v1` blocked (exit-6 error) by default; allowed with
      `allowRemoteLlm=true` + single warning; `https://api.openai.com/v1` same behavior;
      `file://` rejected always. Loopback matrix: `localhost:8080`, `127.0.0.1:11434`,
      `http://[::1]:11434` all allowed.
- [ ] Redirect containment: mock loopback endpoint 302-redirecting to a remote URL → request
      aborts, zero remote connection attempts (request capture).
- [ ] Mock-server tests: success path, 500→retry→success, 429 backoff, timeout abort, 401
      fail-fast with sanitized body (no fallback ping on 401); /models ok; /models 404 →
      fallback ping succeeds with the exact locked payload.
- [ ] `createLlmClient` with `model: null` → E_CONFIG_INVALID with the locked hint.
- [ ] Missing `OPENAI_API_KEY` named in apiKeyEnv → E_CONFIG_INVALID; key value never appears in
      captured logs (redaction test).
- [ ] Budget: tryConsume crossing the cap flips exhausted exactly at the boundary; estimator
      matches formula; `complete()` with attached budget consumes before fetch and throws
      `BudgetExhaustedError` when exhausted (no request sent — capture proof); usage totals
      accumulate.
- [ ] Concurrency: 10 parallel complete() with limit 2 → max 2 concurrent observed by mock.

## Validation

```bash
npm run test -- llm/client llm/budget llm/security-gate
```

## Dependencies

03, 04.

## Non-goals

Streaming, prompt construction (22), embeddings (30), provider-specific auth schemes.

## Design References

- DESIGN §10 (client contract), §13.2 (B1 egress policy), §14 (exit 5/6); ADR-001
