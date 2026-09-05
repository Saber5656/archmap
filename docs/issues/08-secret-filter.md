# 08 — Secret detection and exclusion/redaction policy

## Summary

Implement content-based secret detection over inventoried text files with the two policies from
DESIGN §13.3 (`exclude` | `redact`), producing a report and updating file flags so no downstream
sink (prompts, chunks, index, wiki) ever receives secret material.

## Context

archmap sends code to an LLM (loopback by default, possibly remote by opt-in) and persists chunk
text into `.archmap/index.db`. B2 in the threat model requires a pre-sink filter. Name-based
exclusions already happen in the scanner (07); this issue adds content rules.

## Scope

- In: `src/core/secrets/rules.ts`, `src/core/secrets/scan.ts`, `src/core/secrets/report.ts`.
- Out: consuming the filter in generate/chunker (25, 28 wire it); log redaction (04).

## Detailed Requirements

1. Rulepack (`rules.ts`) — each rule `{id, description, detect(content, path): Finding[]}` with
   these locked contracts (all regexes given here ARE the spec; tune only with fixture proof):
   - `aws-access-key`: `/\bAKIA[0-9A-Z]{16}\b/`.
   - `aws-secret-key`: `/(?i)\baws.{0,30}?(secret|sk).{0,10}?[=:]\s*['"][A-Za-z0-9/+=]{40}['"]/`
     (40-char base64-ish within one line of an `aws…secret` keyword).
   - `github-token`: `/\bgh[pousr]_[A-Za-z0-9]{36,}\b/`.
   - `slack-token`: `/\bxox[baprs]-[A-Za-z0-9-]{10,}\b/`.
   - `openai-key`: `/\bsk-[A-Za-z0-9_-]{20,}\b/`.
   - `google-api-key`: `/\bAIza[0-9A-Za-z_-]{35}\b/`.
   - `private-key-pem`: `/-----BEGIN (RSA |EC |OPENSSH |DSA )?PRIVATE KEY-----/`.
   - `jwt`: three dot-separated base64url segments where the FIRST segment base64url-decodes to
     JSON containing an `"alg"` key; decode failure → no finding.
   - `env-assignment`: `/(?i)^\s*(export\s+)?[A-Z0-9_]*?(API_?KEY|SECRET|TOKEN|PASSWORD)[A-Z0-9_]*\s*[:=]\s*['"]?[^\s'"]{8,}/m`,
     applied ONLY to non-code contexts: files with extension `.env*`, `.ini`, `.cfg`, `.conf`,
     `.properties`, `.toml`, `.yaml`, `.yml`, `.json`, or inventory `language: "other"`.
   - `generic-high-entropy`: candidate = maximal token of charset `[A-Za-z0-9+/=_-]` with
     length ≥ 32; finding when Shannon entropy > 4.5 bits/char, EXCLUDING pure-hex tokens of
     length 40 or 64 (git/sha256 digests) and excluding lockfiles entirely.
2. False-positive suppressions: a match is skipped when the same or previous line contains
   `archmap:allow-secret`; lockfiles (`package-lock.json`, `*.lock`) are exempt from
   `generic-high-entropy` only.
3. Scan eligibility (locked): inventory entries with a `hash` (i.e. not name-based
   `secret-excluded` from issue 07 — those are NEVER read) and `language != "binary"`;
   `too-large` and `generated` files ARE scanned (they feed fallback facts/chunks).
   `security.secretScan: false` → content rules are skipped but the accessor below still exists
   and name-based exclusions still hold (callers never branch on the setting).
4. `scan.ts` public API (the single content gateway for ALL downstream consumers —
   extractors 10–12, metadata 16, chunker 28, pipeline 25):
   ```ts
   filterSecrets(inventory, fileReader: (path: string) => Promise<string>,
                 policy: "exclude" | "redact"): Promise<SecretFilterResult>
   interface SecretFilterResult {
     findings: Finding[]                 // {path, ruleId, line, column, length} — never the text
     updatedInventory: FileInventory     // exclude policy: offending files gain flag "secret-excluded"
     readAnalyzable(path: string): Promise<string>
   }
   ```
   `readAnalyzable` behavior: clean file → raw content; `redact` policy file with findings →
   content with matched spans replaced by `[REDACTED:<ruleId>]` (line count preserved);
   `secret-excluded` file (either name-based or exclude-policy) → throws typed
   `SecretAccessError`; path not in inventory → throws typed `UnknownPathError`.
   Flag semantics (aligned with DESIGN §13.3): `secret-excluded` is set ONLY by name-based rules
   (07) and by the `exclude` policy; the `redact` policy sets NO flag — findings/report only,
   and downstream consumers transparently receive redacted content via the accessor.
5. Report (`report.ts`): human table (stable ordering: path, ruleId, line) + JSON section
   `{scannedFiles: number, findings: [{path, ruleId, line}], policy}` for the generate report.
   Neither format ever contains matched secret text.
6. Performance: single pass per file, compiled regexes, ≤ 2 s over 300k LOC on CI hardware
   (measured in issue 27's benchmark, not gated here).
7. Canary contract for tests: `tests/fixtures/fixture-secrets/` (created in this issue) contains
   fake-but-shaped secrets, e.g. AWS key `AKIAIOSFODNN7EXAMPLE`, an OpenAI-shaped key, a PEM block,
   plus an `archmap:allow-secret` annotated line. The exact canary list is exported from a test
   helper module (`tests/helpers/canaries.ts`) and asserted absent from every artifact in
   issue 39 (single-sourced import, no copies).

## Acceptance Criteria

- [ ] Every rule has positive and negative unit fixtures; findings carry no secret text.
- [ ] `exclude` policy: canary files flagged; `readAnalyzable` refuses them with
      `SecretAccessError`; unknown path throws `UnknownPathError`; name-based-excluded files
      from 07 are never read (fileReader spy).
- [ ] `redact` policy: returned content has `[REDACTED:aws-access-key]` at the right span; line
      count unchanged; NO `secret-excluded` flag added; accessor returns redacted content.
- [ ] `secretScan: false`: no content findings, accessor works, name-based exclusions intact.
- [ ] `allow-secret` annotation suppresses the finding; lockfile entropy exemption works; JWT
      with non-JSON first segment produces no finding.
- [ ] Report golden: stable ordering, correct counts/policy, no secret text in human or JSON
      output (grep against canary list).
- [ ] Scanner flags from 07 and content flags from 08 merge without duplication.

## Validation

```bash
npm run test -- core/secrets
```

## Dependencies

07.

## Non-goals

Git-history scanning (release-time concern, issue 41 checklist); custom user rules (v2).

## Design References

- DESIGN §13.3 (policy), §13.7 (canaries), §7.1 (flags)
