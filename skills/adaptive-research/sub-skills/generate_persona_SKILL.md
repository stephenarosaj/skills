---
name: generate-persona
description: Systematically produces constrained, high-fidelity system prompts and personas for subagents based on specific research goals, scopes, and expected tools.
---

# generate-persona

## Purpose
This skill generates robust, bounded system prompts (personas) for subagents participating in an orchestrated research task. It translates high-level research goals, scoping constraints, and a designated toolset into a rigorous system prompt. This prompt is specifically engineered to keep the subagent focused, minimize hallucination, enforce strict boundaries, and ensure structured, evidence-backed outputs.

## Inputs
When invoking this skill or performing this task, expect the following context:
- **Goal**: The specific objective the subagent needs to achieve.
- **Scope**: The strict boundaries of the research (what to focus on, what to ignore, and negative constraints).
- **Tools**: The specific capabilities or APIs the subagent will have access to.
- **Expected Deliverable**: The structure of the information the subagent must return (e.g., JSON schema, Markdown report with inline citations).

## Prompt Engineering Best Practices for Subagents
When generating the subagent's system prompt, you must incorporate the following prompt engineering principles to ensure robust execution:

1. **Explicit Role Framing**: Ground the LLM in a specific expert persona to tune its token probabilities toward high-quality, domain-specific reasoning.
2. **Negative Constraints**: Autonomous agents are prone to scope creep. Explicitly list what the agent *must not* do (e.g., "DO NOT write code", "DO NOT guess URLs", "DO NOT research topics outside of [X]").
3. **Chain of Thought / ReAct Prompting**: Provide a step-by-step methodology. Tell the agent *how* to approach the problem procedurally (e.g., "First, search... Second, read... Third, synthesize...").
4. **Strict Output Formatting**: Bind the agent to a rigid output structure (like a specific JSON schema) to ensure the overarching orchestrator can parse the results deterministically.
5. **Evidence Anchoring**: To prevent hallucination, explicitly command the agent to cite its sources natively using Markdown links or references mapped to actual tool outputs.

## Persona Generation Framework
Construct the final system prompt using the following sections:

### 1. Identity & Objective
Define the role and the singular, overarching goal.
> *Example*: "You are an expert Cybersecurity Analyst. Your objective is to audit the provided documentation for authentication vulnerabilities."

### 2. Strict Constraints & Boundaries
Translate the input `Scope` into hard rules. Use capitalization for emphasis.
> *Example*: 
> "- DO NOT review authorization or RBAC systems; focus ONLY on authentication."
> "- DO NOT execute arbitrary scripts."
> "- NEVER fabricate citations. If you cannot find the answer via your tools, state 'Unknown'."

### 3. Tool Usage Strategy
Explain *how* to use the provided `Tools` effectively. Do not just list them; provide strategic advice.
> *Example*: "Use `search_web` to find the official documentation first. If a page is too large, use `grep_search` to find specific keywords rather than reading the entire file."

### 4. Step-by-Step Methodology
Provide a clear execution path to achieve the `Goal`.
> *Example*:
> "1. Identify the core components by..."
> "2. Cross-reference the components with..."
> "3. Compile your findings into..."

### 5. Deliverable & Evidence Requirements
Define exactly how the subagent should output its final answer. Emphasize the need for inlined citations based on the overall Orchestrator's requirements.
> *Example*: "You must output your final report as a JSON object matching the following schema. Every claim in the `findings` array MUST include a Markdown link `[Source Name](url)` in the `evidence` field pointing to the tool output that supports it."

## Instructions for the Persona Architect
1. Analyze the provided Goal, Scope, Tools, and Deliverable requirements.
2. Draft the system prompt using the 5-part framework above.
3. Review the drafted prompt:
    - Is it sufficiently bounded? (Will the agent get distracted?)
    - Are the negative constraints strong enough?
    - Is the citation/evidence requirement clear?
4. Output the final generated system prompt enclosed in `<system_prompt>` XML tags.
