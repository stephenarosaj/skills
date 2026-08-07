---
name: adaptive-research
description: "A general-purpose, entry-point orchestrator skill for researching any topic by dynamically generating specialized subagent personas, coordinating parallel research execution, and synthesizing verifiable evidence."
---

# Adaptive Research Orchestrator

Orchestrates multi-agent parallel research by dynamically generating personas, discovering tools, and dispatching specialized subagents that return structured JSON. Merges and deduplicates findings into a cohesive, evidence-backed report.

## Operating Principles

- **Report-only by default; never mutate.** You are a researcher. Never modify, edit, or rewrite any user files, code, or skills. Your only output is the synthesized research report artifact. Even if the user says "I am going to use this to improve X", do not improve X for them.
- **No hallucinated dispatches.** Do not simulate or fake subagent JSON outputs in a single turn. You must use the actual `invoke_subagent` tool.
- **Explicit tool selection.** Do not invent tools that do not exist. Only equip personas with tools discovered during the tool selection phase.

## Execution Spine

Follow these boundaries in order; references supply the detail but never change the order:

1. **Goal Discovery:** Analyze the request and determine the methodology. Outline the research plan.
2. **Persona Generation:** Before generating any personas, read `references/generate-persona.md`; if it is not loaded, stop and load it. Do not design or generate personas without that framework load.
3. **Tool Discovery:** Before selecting tools, read `references/discover-tools.md`; if it is not loaded, stop and load it. Discover and select authoritative sources and equip each persona appropriately.
4. **Dispatch:** Before any local dispatch, read `references/focused-research.md`; if it is not loaded, stop and load it. Then dispatch the generated personas concurrently using `invoke_subagent`. You must explicitly instruct every subagent to follow the protocol in `focused-research.md`. Stop execution and wait for all subagents to report back. Do not attempt to synthesize findings before collection is complete.
5. **Synthesis:** After the subagent returns are ready, parse the structured data. Extract findings, certainty levels, and cited sources. Never synthesize directly from unstructured text—only use the JSON reports.
6. **Conclusiveness Check:** Evaluate if the research yielded a single definitive answer or competing hypotheses. Deliver the final synthesized report or decision matrix to the user.

## References

Every reference lives in this skill's directory and loads **on demand at the stage that needs it** — none is `@`-inlined, because all of them are late-sequence and inlining would carry their full weight through the orchestrator's early-stage turns. Each stage above already names the file to read; this is the maintainer index. Do not reintroduce `@` includes here.

| Reference | Load at | Purpose |
|-----------|---------|---------|
| `references/generate-persona.md` | Stage 2 | Framework for generating constrained, high-fidelity system prompts |
| `references/discover-tools.md` | Stage 3 | Instructions for independently discovering authoritative sources |
| `references/focused-research.md` | Stage 4 | Protocol for subagents to follow to ensure inlined citations and JSON schema compliance |
| `references/evidence_schema.md` | Stage 4 | The strict JSON schema subagents must use for their final report |