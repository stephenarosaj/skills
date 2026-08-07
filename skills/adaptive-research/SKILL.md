---
name: adaptive-research
description: "A general-purpose, entry-point orchestrator skill for researching any topic by dynamically generating specialized subagent personas, coordinating parallel research execution, and synthesizing verifiable evidence."
argument-hint: "[topic/research_question]"
---

# Adaptive Research Orchestrator

Orchestrates multi-agent parallel research by dynamically generating personas, discovering tools, and dispatching specialized subagents that return structured JSON. Merges and deduplicates findings into a cohesive, evidence-backed report.

## Operating Principles

- **Report-only by default; never mutate.** This is a strict execution pipeline. You are bound by state-machine rules. Never modify, edit, or rewrite any user files, code, or skills. Your only output is the synthesized research report artifact. Even if the user says "I am going to use this to improve X", do not improve X for them.
- **No hallucinated dispatches.** Do not simulate or fake subagent JSON outputs in a single turn. You must use the actual `invoke_subagent` tool.
- **Explicit tool selection.** Do not invent tools that do not exist. Only equip personas with tools discovered during the tool selection phase.
- **Missing Dependency Fallback.** If a required reference file is missing or inaccessible, immediately halt the workflow, inform the user of the specific missing dependency, and do not attempt to hallucinate the missing protocol.

## Execution Spine

Follow these boundaries in order; references supply the detail but never change the order:

1. **Goal Discovery (Research Formulation):** Deconstruct the topic into clear, falsifiable hypotheses and establish specific orthogonal evaluation dimensions (quantitative and qualitative). Do not ask the user for clarification; if underspecified, default to an exhaustive multi-source comparative analysis. Provide a brief, human-readable status update to the user outlining the research plan and generated personas before dispatching.
2. **Persona Generation:** You MUST use the `view_file` tool to read `references/generate-persona.md`. Do not design or generate personas without that framework loaded.
3. **Tool Discovery:** You MUST use the `view_file` tool to read `references/discover-tools.md`. Discover and select authoritative sources and equip each persona appropriately.
4. **Dispatch:** You MUST use the `view_file` tool to read `references/focused-research.md` and `references/evidence_schema.md`.
   - Dispatch the generated personas concurrently using `invoke_subagent`.
   - You must pass the *absolute file paths* of both `references/focused-research.md` and `references/evidence_schema.md` directly into the subagents' `Prompt`, instructing them to use `view_file` to read their protocol and schema before starting.
   - Use the `schedule` tool (Condition: `any`) to set a timeout fallback.
   - **Yield your turn to the environment immediately.** Perform a blocking wait for the system to automatically notify you when subagents have asynchronously returned. Do not poll or loop. To ensure the user receives a comprehensive and unbiased report, you must wait for all subagents to finish.
5. **Synthesis (Anti-Bias & Triangulation):** After returns are ready, parse the structured data.
   - Never synthesize directly from unstructured text—only use the JSON reports.
   - If a subagent times out or returns malformed JSON violating the schema, discard the invalid data and proceed with successful subagents.
   - **Anti-Bias Protocol:** Triangulate claims across multiple subagent reports. Actively guard against confirmation and leniency bias. Do not average out or soften contradictions; explicitly document conflicts, highlight methodological differences, and score claims based on objective source authority. Certainty levels must be quantifiable.
   - **Strict Traceable Attribution:** Every synthesized claim, hypothesis, or metric MUST be anchored to a verifiable citation using clickable GitHub-style markdown links (e.g., `[Source Name](url)` or `[file.py:L10](file:///path)`). Ruthlessly discard unverified claims.
   - Never dump raw JSON or verbose transcripts into the chat.
6. **Conclusiveness Check (Visual Density):** Deliver the final synthesized report to the user by writing it to `<appDataDir>/brain/<conversation-id>/adaptive_research_synthesis.md`.
   - You MUST enforce **Visual Density**. Organize complex comparisons into a **Comparative KPI Dashboard (Markdown table)** mapping evidence against hypotheses.
   - Use syntax-verified **Mermaid diagrams** (e.g., flowcharts, sequence diagrams) to visually contrast competing architectural theories or workflows.
   - Use GitHub-style alerts to highlight conclusive answers vs. conflicting data.

## References

Every reference lives in this skill's directory and loads **on demand at the stage that needs it** — none is `@`-inlined, because all of them are late-sequence and inlining would carry their full weight through the orchestrator's early-stage turns. Each stage above already names the file to read; this is the maintainer index. Do not reintroduce `@` includes here.

| Reference | Load at | Purpose |
|-----------|---------|---------|
| `references/generate-persona.md` | Stage 2 | Framework for generating constrained, high-fidelity system prompts |
| `references/discover-tools.md` | Stage 3 | Instructions for independently discovering authoritative sources |
| `references/focused-research.md` | Stage 4 | Protocol for subagents to follow to ensure inlined citations and JSON schema compliance |
| `references/evidence_schema.md` | Stage 4 | The strict JSON schema subagents must use for their final report |
