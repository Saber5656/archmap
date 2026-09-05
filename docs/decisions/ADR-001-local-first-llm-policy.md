# ADR-001: Local-first LLM policy

- Status: accepted
- Date: 2026-07-07

## Context

The product owner's requirement is "code should not leave the machine, if possible" (可能ならローカル
LLM のみ). A hard local-only rule would exclude users who accept a trusted remote endpoint and
would block quality comparisons. An unrestricted default would silently exfiltrate code.

## Decision

1. Default LLM and embeddings endpoint is loopback Ollama (`http://localhost:11434/v1`), speaking
   the OpenAI-compatible protocol so LM Studio/llama.cpp/vLLM also work unmodified.
2. A non-loopback `baseUrl` is refused (exit 6) unless `security.allowRemoteLlm: true` is set
   explicitly in `archmap.config.json`; when enabled, every run prints a warning naming the host.
3. `--no-llm` structural generation is a first-class, fully supported mode: diagrams, tables,
   navigation, citations, freshness manifest, FTS search, and MCP (except `ask_question`) all work
   without any model. The architecture (ADR-003) is designed so this mode carries the core value.
4. No other network egress exists anywhere in the tool (no telemetry, no update checks); this is a
   tested invariant, not a convention.

## Consequences

- Works offline; private code is private by default; secure default satisfies the OSS posture.
- Local-model quality becomes the main quality risk → mitigated by ADR-003 and the eval harness.
- Remote-provider users get one extra config line and a visible warning; acceptable friction.

## Alternatives rejected

- Cloud-first with local option (deepwiki-open posture): violates the stated requirement and
  erases differentiation.
- Hard local-only: needlessly hostile to self-hosted remote inference (e.g., a home-lab GPU box),
  which the loopback check would otherwise block; opt-in flag covers it with informed consent.
