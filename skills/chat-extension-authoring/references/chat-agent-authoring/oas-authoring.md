# OAS authoring reference — chat-agent-authoring

Deeper authoring material for the `chat-agent-authoring` skill: the package files, the
`oas.json` shape, the orchestrator pattern, the cross-cutting OAS rules, and the
chat-dispatch inline-HITL contract. Read `../chat-agent-authoring.md` first.

## Step 4 — Scaffold and write the package files

**Naming convention — kind at the END (mandatory for new/renamed extensions).** Extension packages (agent OR skill) are scoped `@cinatra-ai/...` and read as a noun phrase with the kind LAST. Shape: `@cinatra-ai/<domain>-<capability>-agent` (or the SINGULAR `-skill` for a skill package — the plural `-skills` suffix is retired). Good: `@cinatra-ai/email-test-delivery-agent`, `@cinatra-ai/media-feed-lister-agent`, `@cinatra-ai/web-research-skill`. WRONG: the type-prefix form `@cinatra-ai/agent-<x>` / `@cinatra-ai/skill-<x>` (groups by package type, reads like an internal registry index). The on-disk package directory `extensions/cinatra-ai/<slug>/` MUST match the slug after the scope. See `docs/developer/extensions.md` for the full rationale. The scope is ALWAYS `@cinatra-ai/` — both extension packages and workspace TypeScript packages now share it (the bare `@cinatra/` scope no longer exists; `cinatra-ai` is the vendor, `cinatra` is the product/app name and is NOT used as an npm scope). **Reserved-slug rule (enforced — agent creation/publish HARD-FAILS otherwise, see `packages/agents/src/reserved-workspace-slugs.ts`):** the `<slug>` MUST NOT collide with a workspace package slug — `agents`, `skills`, `chat`, `objects`, `registries`, `a2a`, `trigger`, `lists`, `permissions`, `projects`, `dashboards`, `llm-orchestration`, `mcp-client`, `mcp-server`, `connector-*`, `entity-*`, `asset-*`, `sdk-*`, `metric-*`, `extensions`, `extension-types`, `connectors`, `copilotkit`, `cli`, `agent-ui-protocol`, `google-oauth-connection`, `trigger-email-send`. The kind-at-end convention naturally avoids this — a `-agent`/`-skill` suffix is never a workspace slug. e.g. `@cinatra-ai/objects` is FORBIDDEN (collides with the `@cinatra-ai/objects` workspace package); `@cinatra-ai/object-extractor-agent` is fine.

For a brand-new agent, the PACKAGE itself carries two files. They land under the canonical source-package directory `extensions/cinatra-ai/<packageSlug>/...`. The disk path is server-controlled — you only pass `packageSlug` to the write tools and they place files correctly:

1. `extensions/cinatra-ai/<packageSlug>/cinatra/oas.json` — the OAS Flow definition (Step 5).
2. `extensions/cinatra-ai/<packageSlug>/package.json` — the npm manifest (with `name: "@<vendor>/<packageSlug>"`).

**The agent's behavioural spec is NOT embedded in the agent package.** The Anthropic-schema packaging contract forbids a `SKILL.md` inside a non-skill extension — the packaging gate (CI, store install, publish) rejects it. When the agent needs prompt-level methodology beyond its `system_prompt` (a research discipline, an output-envelope contract, editorial rules), that guidance lives in its own `kind:"skill"` extension (singular `-skill` suffix) and the agent declares a **dependency edge** on it in `package.json#cinatra.dependencies`. The canonical example is `@cinatra-ai/web-research-agent`, which depends on `@cinatra-ai/web-research-skill`:

```json
"cinatra": {
  "dependencies": [
    {
      "packageName": "@cinatra-ai/web-research-skill",
      "edgeType": "runtime",
      "versionConstraint": { "kind": "semver-range", "range": "*" },
      "requirement": "required",
      "kind": "skill"
    }
  ]
}
```

The host resolves the edge at install time and delivers the skill bundle into the agent's runs; the skill's fixes then propagate to every consumer without republishing the agent. (An optional `role` field — `matcher` or `authoring` — says WHICH host surface a skill edge feeds; omit it for a plain behavioural-guidance edge.) How to author the skill extension itself — the bundle shape, the SKILL.md frontmatter contract, the `skill_source_*` lifecycle — is `../chat-skill-extension-authoring.md` territory.

Call `agent_source_write_files` once with `packageSlug` and `packageJson` (JSON string) to create #2, then `agent_source_write` for #1. (The tool still accepts a legacy `skillMd` parameter for pre-contract packages — do not use it for new agents; author the guidance as a skill extension and wire the edge instead.) The server normalizes `package.json#name` to `@<vendor>/<packageSlug>` defensively, but you should still emit the correct value the first time.

Minimal `package.json` template (leaf):
```json
{
  "name": "@<vendor>/<slug>",
  "version": "0.1.0",
  "description": "<one-sentence purpose>",
  "private": false,
  "license": "MIT",
  "publishConfig": { "registry": "http://127.0.0.1:4873" }
}
```

The `license` field is **required** — `agent_source_publish` runs SPDX detection on the package directory and rejects with `LICENSE_DETECTION_REJECTED` when it cannot determine the license. Default to `"MIT"` for new agents unless the user specifies otherwise. If the user picks a copyleft license (GPL/AGPL/LGPL/MPL-2.0), pass `licenseAcknowledged: true` to `agent_source_publish` after confirming with the user.

Orchestrator package.json adds `"cinatra": { "agentDependencies": { "@cinatra-ai/<sub-slug>": "^0.1.0", ... } }`. Leaves NEVER carry npm `dependencies` or `cinatra.agentDependencies` — the publish contract rejects either. (The `cinatra.dependencies` EXTENSION edges — e.g. the skill edge above — are a different field and are valid on any agent.)

## Step 5 — Write the oas.json

Use the canonical OAS Flow 26.1.0 shape.

**Hard rule:** every user-facing input declared on the StartNode MUST appear in either `StartNode.metadata.cinatra.required` (the runtime prompts the user pre-run) or `StartNode.metadata.cinatra.hidden` (the value is always provided programmatically by the dispatcher). An input that appears in neither is silently dropped — the agent runs with whatever was passed in `inputParams`, never prompting the user. The deterministic review (`agent_source_review`) emits a warning for this pattern (`start_node_inputs_without_required`); resolve the warning before publish. Canonical pre-run HITL example: `@cinatra-ai/email-test-delivery-agent` (StartNode declares `metadata.cinatra.required: ["campaignId"]`).

### Chat dispatch & inline HITL contract (Phases 298.18 / 298.20 / 298.21)

When an agent is dispatched **from the chat** (not the `/agents/<v>/<s>/new`
workspace surface), four rules govern whether it can run unattended:

1. **Input extraction is "never invent" (298.18/298.19).** The chat hard
   pre-router asks gpt-5.5 to extract `inputParams` from the user's prompt
   against the StartNode schema, under a strict *never-synthesize* system
   prompt. It will faithfully pull a URL, a topic string, a boolean — but it
   will NOT fabricate a JSON object, an array of rows, or a nested schema the
   user didn't literally type. A deterministic fast-path also lifts an
   explicitly-pasted `... inputParams: { ... }` block verbatim.

2. **Structured StartNode inputs block unattended chat dispatch (298.21).**
   If a `metadata.cinatra.required` field is typed `object` / `array`, or is a
   JSON-string field (e.g. `oasJson`), the chat cannot supply it from a
   natural-language prompt. The run will instead surface a setup-loop HITL
   gate. **When authoring an agent meant to be chat-runnable unattended:**
   give every required `object`/`array` input a sensible non-empty `default`,
   OR keep the structured input `metadata.cinatra.hidden` and derive it from a
   simpler prompt-extractable field. Reviewer-style agents that genuinely need
   a full spec (planner/code-reviewer/security-reviewer take `oasJson`) are
   *expected* to require the operator to paste/HITL the spec — that is correct
   behavior, not a bug.

3. **HITL gates render inline in chat AND are prompt-drivable (298.20/298.21).**
   A mid-run HITL gate renders inline beneath the chat message via the same
   `AgenticRunPanel` the run-detail page uses, so the agent's `xRenderer` MUST
   resolve in `fieldRendererRegistry` (a missing renderer falls back to the
   bare schema-field form). The operator can answer the gate two ways: fill
   the embedded form, OR type the answer into the chat prompt window
   (classifier → same `approveReviewTask` path). For the prompt-window path to
   carry a value into a WayFlow gate, the resume contract requires
   `values.userResponse`; the panel sets this automatically. Authors don't
   need to do anything special — but be aware a single-required-primitive gate
   is the most ergonomic for prompt-window drive (the operator can just type
   the value).

4. **Avoid JSON-Schema `format` on chat-dispatched StartNode string
   inputs.** OpenAI's structured-output `response_format` validator (used by
   the hard pre-router's input extraction) rejects `format` values it
   doesn't recognize — `format:"uri"` returns a 400 and the whole
   extraction fails, so the agent dispatches with empty inputs and stalls.
   The host now strips `format` from the extraction schema defensively,
   but don't depend on it: keep StartNode string inputs as plain
   `{ "type": "string" }` (the title/description still guides the model).
   Reserve `format` for non-chat surfaces only.

Here is the smallest possible valid leaf (a `"node"`-type agent that prompts the user for a single input pre-run):

```json
{
  "agentspec_version": "26.1.0",
  "component_type": "Flow",
  "id": "<slug>-flow",
  "name": "<Display Name>",
  "description": "<one-sentence purpose>",
  "metadata": {
    "cinatra": {
      "type": "node",
      "packageName": "@<vendor>/<slug>",
      "hitlScreens": []
    }
  },
  "inputs":  [{ "title": "<input>",  "type": "string" }],
  "outputs": [{ "title": "<output>", "type": "string" }],
  "start_node": { "$component_ref": "start" },
  "nodes": [
    { "$component_ref": "start" },
    { "$component_ref": "do_work" },
    { "$component_ref": "end" }
  ],
  "control_flow_connections": [
    { "component_type": "ControlFlowEdge", "name": "start_to_do_work", "from_node": { "$component_ref": "start" }, "to_node": { "$component_ref": "do_work" } },
    { "component_type": "ControlFlowEdge", "name": "do_work_to_end",   "from_node": { "$component_ref": "do_work" }, "to_node": { "$component_ref": "end" } }
  ],
  "data_flow_connections": [
    { "component_type": "DataFlowEdge", "name": "start_to_do_work_in",  "source_node": { "$component_ref": "start" },  "source_output": "<input>",  "destination_node": { "$component_ref": "do_work" }, "destination_input": "<input>" },
    { "component_type": "DataFlowEdge", "name": "do_work_to_end_out",   "source_node": { "$component_ref": "do_work" }, "source_output": "<output>", "destination_node": { "$component_ref": "end" },     "destination_input": "<output>" }
  ],
  "$referenced_components": {
    "start":  { "component_type": "StartNode",   "id": "start", "name": "Inputs", "metadata": { "cinatra": { "required": ["<input>"] } }, "inputs":  [{ "title": "<input>",  "type": "string" }] },
    "do_work":{ "component_type": "AgentNode",   "id": "do_work", "name": "Do work", "agent": { "$component_ref": "agent" } },
    "end":    { "component_type": "EndNode",     "id": "end", "name": "End", "outputs": [{ "title": "<output>", "type": "string" }] },
    "agent":  { "component_type": "Agent", "id": "agent", "name": "<agent name>", "system_prompt": "<concise prompt>", "metadata": { "cinatra": { "packageName": "@<vendor>/<slug>" } } }
  }
}
```

For a flow with a HITL gate, use `@cinatra-ai/email-test-delivery-agent` as the template (call `agent_source_read packageSlug:"email-test-delivery-agent"` first).

## Orchestrator pattern (multi-agent compositions)

Orchestrators do NOT inline sub-agents as `Agent` components. They use `FlowNode` + inline subflow `Flow` components. The pattern (lifted from `@cinatra-ai/email-outreach-agent`):

```json
"$referenced_components": {
  "start": { "component_type": "StartNode", ... },
  "child_one_flow": {
    "component_type": "FlowNode",
    "id": "child_one_flow",
    "name": "Step 1 — child one",
    "flow": { "$component_ref": "child-one-subflow" },
    "metadata": {
      "cinatra": {
        "packageName": "@cinatra-ai/<child-one-slug>"
      }
    }
  },
  "child-one-subflow": {
    "agentspec_version": "26.1.0",
    "component_type": "Flow",
    "id": "child-one-subflow",
    "name": "Child One Agent",
    "inputs":  [{ "title": "agent_run_id", "type": "string", "default": "" }],
    "outputs": [{ "title": "<output-port>", "type": "string" }],
    "start_node": { "$component_ref": "<sub-start>" },
    "nodes":     [{ "$component_ref": "<sub-start>" }, { "$component_ref": "<sub-end>" }],
    "control_flow_connections": [...],
    "data_flow_connections": [...]
  },
  "end": { "component_type": "EndNode", ... }
}
```

Rules:

- **NEVER use `A2AAgent` for internal sub-agent composition** (calling another Cinatra agent that runs in the same WayFlow process). A2A is the cross-instance / external protocol. Internal composition must use `FlowNode` + subflow `Flow` (the pattern shown above). This is enforced as a blocker by the runtime-invariant lint rules (the OAS-RUNTIME family) — any `A2AAgent` whose `agent_url` points back at this instance (`{{CINATRA_BASE_URL}}`, `localhost`, `127.0.0.1`, `host.docker.internal`) will fail the pre-publish review and you'll have to rewrite it. Why: wayflowcore's `AgentExecutionStep` explicitly rejects typed `outputs` on `AgentNode`s wrapping `A2AAgent`, so the topology can't return findings to a parent flow; the previous attempt (`@cinatra-ai/agent-creation-finalizer` shipped May 2026) never mounted clean. If you genuinely need cross-instance A2A (different Cinatra deployment, external agent), the scanner emits a `warning` for ambiguous URLs and lets the publish through — read the warning carefully and only proceed if cross-instance is the actual intent.
- **`FlowNode.flow.$component_ref` MUST point to a `Flow` component in `$referenced_components`** — never to an `Agent`. Code review red flag: an `AgentNode` with an inline `Agent` ref inside an orchestrator. Use `FlowNode` + subflow `Flow` instead.
- **The subflow `Flow` is a complete OAS Flow** — it has its own `agentspec_version`, `start_node`, `nodes`, `control_flow_connections`, etc. Read the child agent's actual `oas.json` first (via `agent_source_read`) and copy/adapt its inputs + outputs into the subflow's signature.
- **`metadata.cinatra.packageName`** on the FlowNode must match the child package's `packageName` exactly (the dependency map in `package.json` resolves at install time).
- **HITL gates inside an orchestrator** are owned either by the FlowNode (`metadata.cinatra.gateSteps[]` — see email-outreach-agent for the multi-gate pattern) or by the child sub-flow's own HITL declarations. Pick one — never duplicate.

## Cross-cutting OAS Flow rules

When writing or reviewing an `oas.json`, check these in order — they're the most common failure modes:

1. **Exactly one `StartNode` and one `EndNode`** at the flow level. (Sub-flows each have their own pair.)
2. **Every `$component_ref` resolves**: every key referenced from `nodes`, `start_node`, `control_flow_connections`, `data_flow_connections`, `FlowNode.flow`, `AgentNode.agent` MUST appear in `$referenced_components`.
3. **`AgentNode.agent.$component_ref` points to an `Agent` component (NOT a `Flow`)**. The Agent has `system_prompt` + `metadata.cinatra.packageName`. For sub-agent orchestration, use `FlowNode.flow` → `Flow`, not `AgentNode.agent` → `Agent`.
4. **Data-flow port titles match exactly** between source `outputs[]`/inputs[]` and destination `inputs[]`/outputs[]`. A typo (`account_scope` vs `accountScope`) silently misroutes data.
5. **Flow-level `inputs[]` carry `default: ""` for any port that's optional under A2A start** — child flows started from a parent are not given inputs they don't have defaults for.
6. **`metadata.cinatra.hitlScreens`** is a flat array of renderer IDs — every renderer ID referenced by an `AgentNode.metadata.cinatra.renderer` (or by `FlowNode.metadata.cinatra.gateSteps[].renderer`) must be in this array. The default schema-field-fallback renderer (`@cinatra-ai/agent-builder:schema-field-fallback`) does NOT need to be listed.
7. **Don't use legacy `$component_ref` to global names** like `"shared-llm-config"` or `"cinatra-mcp-toolbox"` — those were aliases from an older OAS variant. Inline the LLM config + toolbox references inside the `Agent` component, or omit them entirely (the runtime defaults are applied).
8. **InputMessageNode contract** (when authoring a HITL flow with re-entrant input collection): exactly one string output, no nested objects. See `docs/developer/wayflow-input-message-node-contract.md` if you need the full contract.
