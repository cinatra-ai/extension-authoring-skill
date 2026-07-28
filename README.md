# Cinatra Extension Authoring Skill

The chat assistant's methodology for authoring Cinatra extension PACKAGES — the reusable, versioned, shippable units (agents, workflows, artifacts, skills) that install on any instance. One router carries the kind-agnostic spine: discover before authoring, confirm before implementing, the shared validator contract, and the six-stage scaffold-to-review lifecycle. Per-kind references layer the source formats on top. It is the content-authoring half of the assistant-skills consolidation, packaged as its own skill.

**Install:** Install `@cinatra-ai/extension-authoring-skill` in your Cinatra instance; the chat assistant consumes it when a user asks to build, author, or publish an extension.

**Usage:** The skill is delivered into chat sessions by the host — you do not invoke it directly. It routes package-authoring requests to the right kind and hard-gates implementation tools behind explicit user confirmation.

**Configuration:** None. The skill carries no credentials and reads no settings.

**Development:** Clone the repository and run `node extension-kind-gate.mjs --package-root .` to validate the manifest. The bundle lives in `skills/chat-extension-authoring/` — a `SKILL.md` router plus reference files (the agent-authoring reference keeps its own deep-dive files one hop below).

**Troubleshooting:** If package-authoring chats produce drafts or instances instead of packages, the delivering assistant is not mounting the bundle; if a kind's tools are rejected, check the platform-admin authorization baseline the router documents.

## Works with

- The Cinatra chat assistant (delegated chat sessions)
- Any extension declaring a skill dependency on this package

## Capabilities

- Distinguish a reusable extension PACKAGE from a one-off draft or instance and route accordingly
- Select the extension kind and walk the shared scaffold, write, validate, build, submit, review lifecycle
- Enforce the trust flow — discovery is free, implementation waits for explicit user confirmation
- Author OAS Flow agents, BPMN workflow packages, artifact semantic manifests and Anthropic-schema skill bundles via the per-kind references
- Wire behavioural guidance as skill dependency edges instead of embedding skills in non-skill packages
