# Case Study: Improving `adaptive-research` Reliability

## The Challenge
Our multi-agent orchestrator, [`adaptive-research`](../skills/adaptive-research/), exhibited instability during complex research tasks. We observed three primary failure modes:
1. **Orchestrator Hangs:** When subagents failed or delayed, the orchestrator frequently fell into infinite polling loops instead of yielding.
2. **Hallucinated Synthesis:** When subagents returned insufficient data, the orchestrator would occasionally hallucinate citations or force a false consensus (leniency bias).
3. **Unstructured Outputs:** Multi-dimensional comparisons were returned as dense text rather than scannable formats.

## The Intervention
We evaluated the skill using the `skill-evaluator` protocol with 8 persona profiles, including a custom `execution-spine-reviewer` designed to find orchestration anti-patterns. 

The evaluation identified that the skill relied too heavily on conversational roleplay (e.g., "You are a researcher") and narrative temporal semantics (e.g., "Wait for subagents"). We refactored the skill into a deterministic state-machine workflow with the following requirements:
1. **Mechanical Yielding:** Explicit instructions to yield to the system event loop with timeout fallbacks, rather than narrative waiting.
2. **Structured Outputs:** Hard requirements for Markdown tables and Mermaid diagrams to represent comparisons and workflows.
3. **Traceability:** Strict formatting for citations (e.g., `[Source](url)`) and explicit instructions not to smooth over contradictory evidence.

## Side-by-Side Evaluation
To verify the changes, we ran the original and refactored skills concurrently against 5 complex research prompts (e.g., aerospace engineering, macroeconomics).

### 1. Standard Execution (Aerospace Engineering Prompt)
*   **Original Skill:** Failed. The orchestrator lacked mechanical yielding instructions. When a subagent delayed, it got stuck in a polling loop and had to be terminated.
*   **Refactored Skill:** Succeeded. The orchestrator yielded to the system event loop, collected all subagent returns asynchronously, and generated a structured report with a Markdown table comparing engine metrics.

### 2. Rate-Limited Execution (Macroeconomics & Urban Planning Prompts)
*   **Original Skill:** Failed. Subagents hit API rate limits. Without a defined failure state, the orchestrator hallucinated a text-heavy report and cited its own execution logs as a primary source to compensate for the missing data.
*   **Refactored Skill:** Succeeded. Subagents hit the same rate limits. The orchestrator triggered its fallback protocol, discarded the failed subagent data, and generated a partial report. It included a clear warning block about the missing data, an empty table highlighting the failed dimensions, and a Mermaid diagram mapping which subagent requests failed.

## Conclusion
Replacing conversational roleplay prompts with strict state-machine constraints and deterministic fallback mechanisms significantly improves the reliability and output quality of multi-agent orchestrators.
