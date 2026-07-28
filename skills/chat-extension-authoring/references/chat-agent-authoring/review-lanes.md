# agent_creation_review — the four review lanes — chat-agent-authoring

Read `../chat-agent-authoring.md` first (it covers the call signature, the return shape, how to use the
result, and the mandatory-before-publish rule). This reference is the deeper breakdown of
what the primitive runs and what NOT to do with it.

## What it does

The primitive runs four review lanes:

1. **`agent-lint-policy`** — every deterministic scanner (literal credentials, untrusted URLs, /api/llm-bridge wiring, and the runtime-invariant checks in the OAS-RUNTIME family). THE ONLY lane authorized to emit `severity: "blocker"`. Lint blockers SHORT-CIRCUIT the LLM lanes (no point spending tokens on an OAS that's structurally unfit).
2. **`agent-security-reviewer`** — advisory LLM security review using the system prompt from `extensions/cinatra-ai/security-reviewer-agent/cinatra/oas.json`.
3. **`agent-code-reviewer`** — advisory LLM code-quality review using the system prompt from `extensions/cinatra-ai/code-reviewer-agent/cinatra/oas.json`.
4. **`agent-planner`** — advisory LLM design review (only runs for non-trivial OAS — skipped when the OAS has only a Start→End with at most one executable step).

`normalizeReviewFindings` downgrades any non-policy `blocker` claim to `warning` (the lint lane is the sole authority for blockers; the LLM lanes can suggest issues but can't gate publish).

Lane source identity is re-stamped by the primitive — the LLMs cannot spoof `source: "agent-lint-policy"` to forge blocker authority.

## What NOT to do

- **Do NOT call** the reviewer agents (`@cinatra-ai/lint-policy-agent`, `@cinatra-ai/security-reviewer-agent`, `@cinatra-ai/code-reviewer-agent`, `@cinatra-ai/planner-agent`) individually via `agent_run` for creation-flow reviews. They run independently as agents, but calling them directly bypasses the primitive's source-identity stamping and the shared merge step's blocker-authority enforcement.
- **Do NOT decide blocker authority client-side.** The primitive is the trust boundary. If a finding has `severity: "blocker"` after normalization, it IS a blocker.
- **Do NOT flag the trigger target-run-id divergence.** An embedded `trigger-subflow` inside a scheduled-watcher orchestrator targets the **parent/orchestrator** run, NOT the subflow's own run — and the binding FIELD varies (a dedicated `parentRunId` input in some orchestrators, a `cinatra_run_id`→`agent_run_id` mapping in others). Do not assume a single field name and do not "unify" it: this is the INTENTIONAL contract, because a scheduled watcher must re-fire the whole orchestrator rather than just its trigger subflow.
  - **Historical case, still not flaggable.** The other half of the divergence was the STANDALONE `@cinatra-ai/trigger-agent`, which keyed its persist by its OWN run (`cinatra_run_id` → `agent_run_id`) because the standalone agent *was* the thing being scheduled. That package was **retired in cinatra#1034 and its repo is archived** — it is in no cinatra extension lock, and no shipping agent OAS presents that shape today. It is recorded here as history because a legacy or imported OAS can still present it (the archived package's OAS is still publicly readable and released, and cinatra still carries the standalone binding at runtime — the agents passthrough route's `trigger_config_set` run-id fallback chain, and the WayFlow trigger-wait executor's "preceding trigger-agent subflow" input contract). If you meet it in an OAS under review, it is the historical standalone mode, not an inconsistency to report.
  - Both modes stay documented in the platform reference — `agent-development.md` Rule 15 (`cinatra-ai/docs`, `guides/developer/agent-development.md`; published at <https://docs.cinatra.ai/guides/developer/agent-development/>).
