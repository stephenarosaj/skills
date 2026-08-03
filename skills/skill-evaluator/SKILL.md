---
name: skill-evaluator
description: A meta-skill that combines the multi-persona static analysis of ce-code-review with the empirical quantitative evaluation loop of skill-creator. Use this to rigorously critique, rewrite, and empirically prove improvements to an existing skill.
---

# Skill Evaluator Protocol

You are the `skill-evaluator` orchestrator. Take an existing target skill, ruthlessly critique its structural logic using a dynamically selected roster of specialized persona sub-agents, automatically patch the identified loopholes, and run an empirical evaluation loop to *prove* the rewritten version is better than the original.

**Dependencies**:
- `generate_review.py` (external script from `skill-creator` for UI presentation).

## Stage 1: Scope & Selection
1. Read the target skill's `SKILL.md` using the absolute path provided by the user. If the user did not provide a path, ask them for it.
2. Read the `references/roster-selection.md` file to determine your evaluation roster. Apply the objective heuristics listed there to build the final list of active personas.

## Stage 2: Dispatch
1. Send a non-blocking status update to the user detailing the chosen roster and explaining *why* the conditional personas were activated, then immediately proceed.
2. Spawn parallel subagents for the selected roster. Pass the target skill's source text to each subagent and assign them their specific local persona from `references/personas/`.
3. Set a fallback timer using the `schedule` tool (e.g., 10 minutes, `TimerCondition="any"`) and yield your turn. 
4. If a subagent times out or fails, log the failure in a workspace artifact and proceed using only the successful critiques.

## Stage 3: Synthesis & Rewrite
1. Synthesize the findings from the personas into a single master checklist of flaws and save it as an artifact (`flaws_checklist.md`).
2. Backup the target `SKILL.md` to `SKILL.old.md` in its directory.
3. Rewrite the target skill in its entirety to surgically patch every flaw while preserving existing Markdown structure and headers. Write the output to `SKILL_proposed.md`.
4. Read `SKILL_proposed.md` to verify it retains a valid YAML frontmatter block.
5. Provide a summary of the structural changes made to the user.

## Stage 4: Empirical Testing
1. If the target skill does *not* have an `evals/evals.json` file, notify the user that empirical testing is skipped and proceed directly to Stage 5. If it does exist, validate its JSON structure.
2. Demand human confirmation before running the empirical tests, as the tests will execute the rewritten `SKILL_proposed.md` code.
3. Spawn parallel grader subagents (maximum 5 concurrently to prevent exhaustion) to execute the test prompts for both `SKILL.old.md` and `SKILL_proposed.md`. 
4. Mandate that grader subagents return strict JSON schemas. Save all test results directly to an `eval-results.json` artifact to prevent context bloat.

## Stage 5: Feedback & Handoff
1. If empirical testing was run, run `generate_review.py` to present the quantitative benchmark. If the script is unavailable or tests were skipped, present a detailed Markdown summary of the changes instead.
2. If the `AUTO_APPROVE` instruction is active (e.g. headless CI), skip waiting for user feedback and exit gracefully.
3. Wait for user feedback (yield your turn, do not loop). You may loop back to Stage 3 to incorporate feedback a maximum of 3 times. If unresolved, stop and output a final summary artifact.
