---
name: multi-agent-code-review
description: "Structured code review for bugs, regressions, tests, and standards. Use before PRs or when asked for review; report-only by default, with explicit local apply available for user-directed fix workflows."
argument-hint: "[mode:agent] [apply:local] [plan:<path>] [depth:full] [PR/branch/base]"
---

# Code Review

Reviews code changes using dynamically selected reviewer personas. Dispatches bounded specialist subagents that return structured JSON, then merges and deduplicates findings into a single report.

## Setup

Run this once at the start of this invocation, before any subagent dispatch, and follow the directives it prints — except where one conflicts with this skill's own rules on asking the user questions, whether those rules are scoped to a non-interactive mode or apply in every mode, in which case this skill's rules win and no blocking question is asked. Run the fence exactly as written, as its own command: do not pipe or filter it (no `head`, `tail`, or `grep`), do not truncate its output, and do not bundle it into a batch with other commands. Its output opens with a `=== skill context` header and ends with `CE_CONTEXT_END`; if you received one of those lines without the other, the output was truncated — rerun the fence verbatim once. That recovery is the only rerun: otherwise do not rerun it within the same invocation; a later invocation of this or any other skill runs its own. If no Node runtime is available the skill proceeds unchanged.

```bash
SKILL_DIR="<absolute path of the directory containing the SKILL.md you just read>";
NODE="$(for c in node nodejs; do command -v "$c" >/dev/null 2>&1 && "$c" -e '' >/dev/null 2>&1 && { echo "$c"; break; }; done)";
if [ -n "$NODE" ]; then
"$NODE" "$SKILL_DIR/scripts/context.mjs" || echo "context script failed; continue with the skill's normal behavior";
else
echo "no Node runtime; continue with the skill's normal behavior";
fi
```

## When to Use

- Before creating a PR
- After completing a task during iterative implementation
- When feedback is needed on any code changes
- Can be invoked standalone
- Can run inside larger workflows; use `mode:agent` when the caller needs JSON instead of markdown tables

## Artifact Root

This skill discovers plans under `<root>/plans/`, scans learnings under `<root>/solutions/`, and passes the resolved root to `review-scope.py` (`--docs-root`) and to its persona subagents. Resolve `<root>` before you first compose a `<root>/` path or the `--docs-root "<root>"` argument (per the block below), and substitute it everywhere those appear.

<!-- ce-docs-root:start -->
**Resolve the CE artifact root `<root>` before composing any artifact path.**

- **Read** `docs_root` from `<repo-root>/.compound-engineering/config.local.yaml`, then `config.yaml`; first non-empty value wins (`<repo-root>` = `git rev-parse --show-toplevel`). Unset -> `<root>` is `docs`, exactly as before.
- **Validate** a set value: a repo-relative directory whose real, symlink-resolved path stays inside the repo and is neither the repo root nor under `.git/`. Otherwise stop with an error naming `docs_root` and the value -- never fall back to `docs`.
- **Use** `<root>` as the sole artifact location: create it if absent, compose each path as `<root>/<subdir>` with this skill's own subdirectory, and never also read `docs`.
<!-- ce-docs-root:end -->

## Execution spine

Follow these boundaries in order; references supply the detail but never change the order:

1. Resolve the reviewed diff and intent.
2. Read `references/persona-catalog.md`, then select the risk-driven reviewer roster and discover applicable standards paths (searching `CLAUDE.md`, `AGENTS.md`, and `./agents/` / `.agents/`). Do not select or dispatch personas without that catalog load.
3. Before any local dispatch, read `references/dispatch-reviewers.md`; if it is not loaded, stop and load it. Then dispatch the materialized local roster as a foreground concurrent batch sized to the host's active-agent cap — spawn multiple reviewers in one message with background execution off where the harness runs same-message calls concurrently, and collect every reviewer before synthesis (one blocking wait on Claude-style harnesses; repeated non-polling collection waits on async `spawn_agent` harnesses); degrade to serial where it does not. Detaching local review into a polled background job is forbidden. Shell no-ops and wakeup polling are forbidden.
4. After the reviewer returns are ready, read `references/finish-review.md`; if it is not loaded, stop and load it. Run the documented findings mechanics, run every validator the reference selects, and only then return the report. Never synthesize directly from raw reviewer artifacts. The exact Actionable Findings, Coverage, and Verdict completion fields are required. In the multi-agent path, emit only this skill's report; do not also invoke a harness-native findings/reporting tool. The native review tool belongs only to the explicit Quick Review Short-Circuit. Bare and `mode:agent` reviews never apply fixes; only explicit `apply:local` can enter the apply stage.

Bundled helper contracts in the stage references are authoritative. Run the documented commands directly; do not inspect helper source, grep model mappings, dry-run adapters, or probe `--help` unless a documented command actually fails with an incompatibility.

## Task Visibility

For the multi-agent path, once the review scope is resolved, use the platform's task-tracking capability when available to show a short user-facing view derived from the execution spine. Track review outcomes, not individual personas, setup mechanics, or tool calls; add conditional work only when its gate fires, and update the view at meaningful transitions. If no task-tracking capability is available, continue with the normal progress and final report without simulating a task list in chat.

## Argument Parsing

Parse the arguments you were invoked with for optional tokens (`mode:agent`, `apply:local`, `plan:<path>`, `depth:full`, `grouping:*`, `base:*`). Strip each recognized token before interpreting the remainder as a PR number, GitHub URL, or branch name. For full argument token definitions, examples, and conflicting-argument rules, read `references/invocation-arguments.md`.

## Operating principles

Same review pipeline for default and `mode:agent`:

- **Report-only by default; never push.** A bare `multi-agent-code-review` invocation produces findings and does not apply them. Local mutation requires `apply:local` or an explicit user request in the invoking prompt to apply/fix this review's findings. `mode:agent` never mutates the tree, even when nested inside a workflow that later applies findings. Never push, open PRs, or file tickets in any mode.
- **No blocking prompts.** Never use `AskUserQuestion`, `request_user_input`, `ask_user`, or other blocking question tools. Infer intent, plan, and scope from explicit tokens, git state, PR metadata, and conversation. Note uncertainty in Coverage or the verdict — do not stop to ask.
- **Explicit mutations only.** Never run `gh pr checkout`, `git checkout`, `git switch`, or similar branch-switch commands. Passing a PR number, URL, or branch name selects **review scope**, not permission to mutate the working tree. To review local uncommitted work on a feature branch, check out that branch yourself (or stay on it) and pass `base:` or no target.
- **Smart defaults.** Untracked files: review tracked changes only and list excluded paths in Coverage. Plan: use `plan:` when passed; otherwise discover conservatively from PR body or branch keywords. Weak advisory P2/P3 from testing/maintainability alone: demote to `testing_gaps` / `residual_risks` per Stage 5.
- **Report outcomes, not machinery.** What you show the user is about the review: what's being examined (the PR/branch), which coverage is included and the one-line reason for each conditional lens, and the findings. Keep the skill's internals out of user-facing text — model-tier assignments, raw scope-mode codenames (`local-aligned`/`pr-remote`), staging the diff to disk, loading persona files, parallel-dispatch bookkeeping, and step-by-step narration of your own setup. Name what the user would recognize (a PR number, a reviewer's concern), not the plumbing. This governs *what* you surface and suppress; it does not script the wording — use your own voice.

## Output format

| Invocation | Deliverable |
|------------|-------------|
| **Default** | Report-only markdown (pipe-delimited finding tables) + Actionable Findings summary |
| **Explicit local apply** | The same markdown report plus verified local fixes and an Applied section |
| **`mode:agent`** | One JSON object (see ### JSON output format below) + the same `/tmp/.../multi-agent-code-review/<run-id>/` artifacts |

Default and `mode:agent` are **report-only**. `mode:agent` changes only the serialization from markdown to JSON for programmatic callers; it does not change reviewer selection, merge logic, or scope rules. `apply:local` is separate mutation authority, not an output mode. The default markdown is the human view; keep it ASCII-safe (pipe tables, `->` not middot `·`, no box-drawing) so it degrades gracefully across terminals.

## Quick Review Short-Circuit

If the invocation arguments indicate the user wants a quick, fast, or light code review — and **`mode:agent` is not active** — do not dispatch the multi-agent flow.

**Announce the chosen path** before any other work (Quick review vs Multi-agent review). Skip this announcement when `mode:agent` is active.

Sequence:

1. **Run the harness's built-in code review.** Forward any review target after stripping tokens. Then stop — do not dispatch the multi-agent pipeline.
2. **Exemption:** If no built-in review exists, continue into the full multi-agent review.
3. **`mode:agent` bypasses this short-circuit** — always run the full multi-agent review and return JSON.

**Deprecated:** `mode:autofix` is no longer supported. If passed, ignore it and proceed report-only; it does not grant local apply authority.

## Severity Scale

All reviewers use P0-P3:

| Level | Meaning | Action |
|-------|---------|--------|
| **P0** | Critical breakage, exploitable vulnerability, data loss/corruption | Must fix before merge |
| **P1** | High-impact defect likely hit in normal usage, breaking contract | Should fix |
| **P2** | Moderate issue with meaningful downside (edge case, perf regression, maintainability trap) | Fix if straightforward |
| **P3** | Low-impact, narrow scope, minor improvement | User's discretion |

## Action Routing

Severity answers **urgency**. `autofix_class` and `owner` are **signal** describing follow-up shape for callers; this metadata does not grant apply permission. Apply authority is separate, explicit, and checked before Stage 5c. See `references/action-class-rubric.md` for persona guidance.

| `autofix_class` | Default owner | Meaning |
|-----------------|---------------|---------|
| `gated_auto` | `downstream-resolver` or `human` | Concrete `suggested_fix` proposed; caller applies after judgment |
| `manual` | `downstream-resolver` or `human` | Actionable work needing design input or handoff |
| `advisory` | `human` or `release` | Report-only — learnings, rollout notes, residual risk |

Routing rules:

- **Synthesis owns the final route.** Persona-provided routing metadata is input, not the last word.
- **Choose the more conservative route on disagreement.** A merged finding may move from `gated_auto` to `manual`, but never widen without stronger evidence.
- **Reject `safe_auto` and `review-fixer` if present** — drop the finding or remap to `gated_auto` / `downstream-resolver` during synthesis.
- **`requires_verification: true` means any caller-applied fix needs targeted tests or ## Reviewers & Scope

Reviewer personas are selected in layers from `references/persona-catalog.md`:
- **Edge-Case Reviewer**: `correctness-reviewer`, `reliability-reviewer`, `julik-frontend-races-reviewer`.
- **Test Completeness Reviewer**: `testing-reviewer` (enforces 100% changed-path coverage & Fail-First standalone repros).
- **Blast Radius / Impact Analyzer**: `blast-radius-reviewer` (checks for out-of-scope modifications & cross-module coupling leaks).
- **Security & Threat Model Auditor**: `security-reviewer` (CWE vulnerability scans, injection risks, auth bypasses).
- **Architecture & Readability Reviewer**: `project-standards-reviewer`, `maintainability-reviewer` (enforces `@user_global` Javadoc/inline comment rules).

For full reviewer selection rules, conditional spawn gates, protected artifact rules, and plan requirements completeness checks, read `references/roster-selection.md` and `references/intent-and-plan.md`.

## How to Run (Orchestration Spine)

Execute the code review workflow sequentially across these 6 stages. Read each reference file on demand at the stage that requires it:

### Stage 1 & 1b: Resolve Scope & Diff Signals
Read `references/scope-resolution.md`. Follow its rules to:
- Resolve diff base (`base:<ref>` fast path, PR number/URL, branch, or standalone working tree).
- Execute the PR-state probe and trivial-PR judgment skip check.
- Sanitize PR remote ref names (`headRefName`) before calling `git fetch`.
- Compute deterministic line and signal metrics using `scripts/review-scope.py` (with automatic shell fallback if Python 3 is absent).

### Stage 2, 2b & 2c: Discover Intent & Plan
Read `references/intent-and-plan.md`. Follow its rules to:
- Write a 2–3 line intent summary from PR/branch metadata and commits.
- Discover design documents or implementation plans **only when explicitly specified by the user** (`plan:<path>` or user-specified context doc). Do not auto-scrape random docs.
- Extract approved Key Technical Decisions (KTDs) for Stage 5 triage while isolating reviewers from settlement annotations.

### Stage 3, 3b, 3c & 3d: Select Reviewers & Stage Run Dir
Read `references/roster-selection.md` and `references/persona-catalog.md`. Follow their rules to:
- Select the active reviewer team (generic before domain reviewers).
- Discover applicable project standards files (`CLAUDE.md`, `AGENTS.md`, `./agents/*.md`, `.agents/*.md`) for `project-standards-reviewer`.
- Evaluate the small-diff fast path (`lite_eligible` roster for trivial low-risk application code diffs).
- Create the temporary scratch run directory (`RUN_DIR`) and announce the team to the user.

### Stage 4: Dispatch and Collect Reviewers
Read `references/dispatch-reviewers.md`. Follow its rules to:
- Execute the inline fast pass for trivial code-level checks.
- Assign local model tiers (session tier for `correctness`/`security`/`adversarial`; mid-tier for others).
- Emit a progress alert in non-tracking UI environments.
- Dispatch selected reviewer subagents as a bounded foreground concurrent batch and collect all compact JSON results.

### Stage 5: Finish the Review (Merge, Validate & Apply)
Read `references/finish-review.md`. Follow its rules to:
- Merge and mechanically deduplicate findings across reviewers.
- Execute Stage 5b validator batching (chunked into batches of 6 max per batch when surviving P0/P1 blockers exceed 8).
- Execute Stage 5c explicit local apply only when authorized by `apply:local`.

### Stage 6: Synthesize & Present Report
Read `references/review-output-template.md` and follow Stage 6 in `references/finish-review.md` to:
- Render the final report (markdown in default mode; raw JSON in `mode:agent`).
- Ensure any surviving P0 or P1 `validation-degraded` finding forces a `"Not ready"` verdict.
- Always surface P0/P1 plan deviations in a dedicated `### Plan Deviations & Settled Decision Conflicts` section.

## After Review

After Stage 6, stop. Never push, open PRs, or file tickets from this skill. Bare and `mode:agent` reviews mutate nothing. When local apply was explicitly authorized, Stage 5c may already have applied and, on a clean pre-review tree, committed verified fixes. Otherwise the caller or user decides what to apply from the report and artifacts.

### Emit actionable findings summary (default mode only)

After Stage 6 **in default mode**, emit a compact **Actionable Findings** summary for callers:

- List each actionable finding (`gated_auto` or `manual` with `downstream-resolver`) with stable `#`, severity, file:line, title, `autofix_class`, whether `suggested_fix` is present, and `confidence`.
- Include the resolved run-artifact path when one was written.
- When the actionable queue is empty, state `Actionable findings: none.` explicitly.

In `mode:agent` do **not** emit this markdown summary — the actionable findings are carried solely by the `actionable_findings` field of the JSON object. Emit nothing after the JSON object, so the response stays a single parseable JSON value.

Do not run post-review triage (no per-finding walk-through, bulk ticket filing, or routing questions). The report and summary are the complete handoff.

### Mode-specific completion

| Mode | After Stage 6 + actionable summary |
|------|-----------------------------------|
| **Default** | Markdown tables + Actionable Findings summary. |
| **`mode:agent`** | JSON object + `review.json` in run artifact dir. |

Do not offer push/PR/create-branch next steps from this skill.

#### Run artifacts

Always write run artifacts under the resolved `<run-dir>`:

- synthesized findings
- actionable findings list
- advisory outputs
- per-agent `{reviewer_name}.json` from Stage 4
- `report.md` — the rendered markdown report exactly as presented to the user (default mode only), so format and numbering stay auditable after the run

`metadata.json` minimum fields:

```json
{
  "run_id": "<run-id>",
  "branch": "<git branch --show-current at dispatch time>",
  "head_sha": "<git rev-parse HEAD at dispatch time>",
  "verdict": "<Ready to merge | Ready with fixes | Not ready>",
  "completed_at": "<ISO 8601 UTC timestamp>"
}
```

Capture `branch` and `head_sha` at dispatch time (no in-skill fixes will land afterward).

**Automatic Scratch Cleanup:** When all subagent review is done and the main skill/agent completes its run (after the final report has been delivered or read), automatically delete the temporary scratch run directory (`rm -rf "$RUN_DIR"`) so temporary review files never accumulate across invocations.

## Fallback

If the platform doesn't support parallel sub-agents, run reviewers sequentially. If the platform supports sub-agents but caps active concurrency, use the bounded queueing rules in Stage 4 rather than treating cap-related spawn failures as reviewer failures. Everything else (stages, output format, merge pipeline) stays the same.

---

## References

Every reference lives in this skill's directory and loads **on demand at the stage that needs it** — none is `@`-inlined, because all of them are late-sequence and inlining would carry their full weight through the orchestrator's many early-stage turns and subagent dispatches. Each stage below already names the file to read; this is the maintainer index. Do not reintroduce `@` includes here.

| Reference | Load at | Purpose |
|-----------|---------|---------|
| `references/invocation-arguments.md` | Setup / Argument Parsing | Full argument token definitions, examples, and conflicting-argument rules |
| `references/scope-resolution.md` | Stage 1 & 1b | Diff endpoint resolution, PR probes, remote ref sanitization, and scope signals |
| `references/intent-and-plan.md` | Stage 2, 2b & 2c | Intent summary, user-specified plan discovery, and KTD extraction |
| `references/roster-selection.md` | Stage 3, 3b, 3c & 3d | Persona selection rules, standards discovery, small-diff lite path, and run dir staging |
| `references/persona-catalog.md` | Stage 3 | Full per-persona selection criteria and spawn gates |
| `references/dispatch-reviewers.md` | Stage 4 | Inline fast pass, model tiers, persona dispatch contract, and collection |
| `references/subagent-template.md` | Stage 4 via dispatch protocol | Dispatch shape for every persona subagent |
| `references/diff-scope.md` | Stage 4 via dispatch protocol | Shared diff-scope rules passed to each subagent |
| `references/findings-schema.json` | Stage 4 via dispatch protocol | JSON output contract passed to each subagent |
| `references/finish-review.md` | Stage 5 | Merge, validation, action routing, and final report |
| `references/action-class-rubric.md` | Action Routing (as needed) | Persona guidance for `autofix_class` |
| `references/review-output-template.md` | Stage 6 | Canonical section skeleton for the report |

Selected reviewer prompt assets live under `references/personas/`. Read only the prompt files selected for the current review.
