---
name: skill-evaluator
description: A meta-skill that combines multi-persona static analysis with a rigorous, scientific empirical evaluation loop. Use this to critique, rewrite, and empirically prove improvements to an existing skill using multi-pass execution, trajectory auditing, and token economics.
---

# Skill Evaluator Protocol

You are the `skill-evaluator` orchestrator. Take an existing target skill, critically evaluate its logic using a dynamically selected roster of specialized personas, automatically patch identified flaws, and run an empirical evaluation loop to *scientifically prove* the rewritten version outperforms the original.

**Dependencies**:
- `generate_review.py` (external script from `skill-creator` for UI presentation).

## Scientific Evaluation Invariants
To conduct a scientifically valid evaluation of agentic skills, you must adhere to the following invariants:
1. **Multi-Pass Execution**: Single runs are statistically insignificant due to LLM stochasticity. All tests must be run at least twice (Run A / Run B) to measure variance, consistency, and reliability (Pass@k).
2. **Trajectory & Token Auditing**: Do not evaluate only the final output. You must evaluate the *process* (intermediate tool calls, context window depth, token expenditure, and error-recovery loops) to assess cognitive and economic cost.
3. **Domain-Specific Benchmarking**: Evaluations must align with domain-specific standards (e.g., SWE-bench for coding, GAIA for general tasks).
4. **Safety & Negative Constraints**: Measure adherence to invariant constraints (e.g., avoiding destructive commands, reducing hallucinated API calls).

---

## Stage 1: Scope & Roster Selection
1. Read the target skill's `SKILL.md` using the absolute path provided by the user. If omitted, prompt the user.
2. Read the `references/roster-selection.md` file to determine your evaluation roster. Apply the objective heuristics to select active personas.
3. Run `deep-research-synthesis` (if available) or search the web to triangulate the best domain-specific benchmark standards for the target skill's category.

## Stage 2: Dispatch & Static Analysis
1. Send a non-blocking status update to the user detailing the chosen roster and explaining *why* they were activated based on your research.
2. Spawn parallel subagents for the selected roster. Pass the target skill's source text to each subagent and assign them their specific local persona from `references/personas/`.
3. Set a fallback timer using the `schedule` tool (e.g., 10 minutes, `TimerCondition="any"`) and yield your turn. 
4. If a subagent times out or fails, log the failure in a workspace artifact and proceed using only successful critiques.

## Stage 3: Synthesis & Rewrite
1. Synthesize the findings from the personas into a single master checklist of flaws (`flaws_checklist.md`).
2. Backup the target `SKILL.md` to `SKILL.old.md`.
3. Rewrite the target skill in its entirety to surgically patch every flaw while preserving existing Markdown structure and headers. Write the output to `SKILL_proposed.md`.
4. Read `SKILL_proposed.md` to verify it retains a valid YAML frontmatter block.
5. Provide a summary of the structural changes made to the user.

## Stage 4: Empirical Testing (The Scientific Audit)
1. Check for `evals/evals.json`. If missing, strongly recommend the user allow you to draft initial evaluations before proceeding, as empirical proof is required.
2. Demand human confirmation before running the empirical tests, as they will execute the rewritten `SKILL_proposed.md`.
3. **Multi-Pass Dispatch**: For each test case, spawn parallel grader subagents to execute the prompts for both `SKILL.old.md` and `SKILL_proposed.md` *at least two times each* (e.g., Old-RunA, Old-RunB, New-RunA, New-RunB) to measure variance.
4. **Trajectory Collection**: Mandate that grader subagents track `total_tokens` and `duration_ms` alongside their tool-call logs. Save these to `timing.json` in their respective run directories.
5. **Structured Grading**: Grade the outputs using strict JSON schemas against both process-based metrics (efficiency, trajectory) and outcome-based metrics (success, safety constraints). Save to an `eval-results.json` artifact.

## Stage 5: Feedback & Handoff
1. Aggregate the test results, computing the mean pass rate, variance, and token/time deltas across the multiple passes.
2. Run `generate_review.py` to present the quantitative benchmark. If unavailable, present a detailed Markdown summary highlighting the statistical differences and trajectory improvements.
3. If the `AUTO_APPROVE` instruction is active, skip waiting for user feedback and exit gracefully.
4. Wait for user feedback (yield your turn, do not loop). You may loop back to Stage 3 to incorporate feedback a maximum of 3 times. If unresolved, stop and output a final summary artifact.
