# 🤖 Agentic Orchestrator Design Patterns & Anti-Patterns

When designing complex, multi-agent orchestrator workflows (like `adaptive-research`), relying on standard conversational LLM prompting leads to "Execution Collapse"—where agents hallucinate states, get stuck in polling loops, or fail to gracefully handle subagent timeouts. 

To build production-ready orchestrators, you must abandon narrative prompting in favor of strict, mechanical **State-Machine Execution Spines**.

Here are the core learnings and patterns distilled from our side-by-side empirical evaluations:

## 1. Framing: Pipeline vs. Roleplay
*   **Anti-Pattern (Roleplay):** `"You are a world-class researcher. Your job is to gather data..."`
    *   *Why it fails:* Encourages the LLM to "act out" the completion of the task in a single turn (hallucinating subagent responses) rather than mechanically driving tools.
*   **Best Practice (State-Machine):** `"This is a strict execution pipeline. You are bound by the following state-machine rules. Do not simulate outputs."`

## 2. Yielding: Mechanical vs. Narrative Temporal Semantics
*   **Anti-Pattern (Narrative Wait):** `"Stop execution and wait for all subagents to report back before synthesizing."`
    *   *Why it fails:* "Wait" is a temporal concept that LLMs don't natively understand in a stateless API call. It causes the agent to either write a destructive `sleep` polling loop, or hallucinate the wait and instantly skip to synthesis.
*   **Best Practice (Mechanical Yield):** `"Yield your turn to the environment immediately. Perform a blocking wait for the system to notify you when subagents return. Do not poll or loop."`

## 3. Resilience: Timeout Fallbacks
*   **Anti-Pattern (Happy-Path Dependency):** Assuming subagents will always successfully return JSON.
    *   *Why it fails:* If a subagent hits a rate limit or crashes, the orchestrator hangs indefinitely.
*   **Best Practice (Schedule Traps):** Use explicit background timers (e.g., the `schedule` tool with `TimerCondition: any`) immediately after dispatching subagents. Instruct the orchestrator to discard unparseable or timed-out subagent data and proceed with the successful ones.

## 4. Subagent Prompting: Absolute Paths vs. Paraphrasing
*   **Anti-Pattern (Fragile Delegation):** `"Explicitly instruct every subagent to follow the protocol defined in focused-research.md."`
    *   *Why it fails:* The orchestrator LLM will attempt to summarize or paraphrase the protocol into the subagent's prompt, diluting constraints and losing schema fidelity.
*   **Best Practice (Absolute Injection):** `"Pass the *absolute file paths* of the protocol and schema directly into the subagent's Prompt, instructing them to use the view_file tool to read their instructions before starting."`

## 5. Synthesis: Visual Density & Strict Attribution
*   **Anti-Pattern (Text Blobs & Leniency Bias):** Asking for a "synthesized report" and allowing the agent to politely average out conflicting data. This leads to hallucinated citations and unreadable text walls.
*   **Best Practice (Dashboards & Anchoring):** 
    *   **Visual Density:** Mandate that all outputs use Markdown KPI Dashboards (tables) to compare competing hypotheses side-by-side, and Mermaid diagrams to map architectures or failure states.
    *   **Anti-Bias:** Instruct the agent to *never* soften contradictions. Explicitly document conflicts.
    *   **Traceable Attribution:** Every claim MUST be anchored to a verifiable citation using strict GitHub-style markdown links (e.g., `[Author](url)`). Ruthlessly discard unverified claims.
