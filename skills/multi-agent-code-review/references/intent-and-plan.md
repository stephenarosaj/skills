# Intent & Plan Discovery (Stage 2, 2b & 2c)

## Stage 2: Intent Discovery

Understand what the change is trying to accomplish. The source of intent depends on which Stage 1 path was taken:
- **PR/URL mode:** Use the PR title, body, and linked issues from `gh pr view` metadata. Supplement with commit messages from the PR if the body is sparse.
- **Branch mode:** Run `git log --oneline ${BASE}..<branch-ref>` using the resolved merge-base and resolved branch ref from Stage 1. Use `<branch-ref>`, not the raw `<branch>` argument.
- **Standalone (current branch):** Run:
  ```bash
  echo "BRANCH:" && git rev-parse --abbrev-ref HEAD && echo "COMMITS:" && git log --oneline ${BASE}..HEAD
  ```

Combined with conversation context (plan section summary, PR description), write a 2-3 line intent summary:
```
Intent: Simplify tax calculation by replacing the multi-tier rate lookup
with a flat-rate computation. Must not regress edge cases in tax-exempt handling.
```
Pass this to every reviewer in their spawn prompt. Intent shapes *how hard each reviewer looks*, not which reviewers are selected. Keep any `session-settled:` annotations (from a plan or the conversation) out of this summary — reviewers stay blind to settlement (Stage 2b).

**When intent is ambiguous:** Infer from branch name, commits, PR title/body, diff, `plan:`, and conversation. Write the best-effort intent summary and note uncertainty in Coverage — never block on a clarifying question.

## Stage 2b: Plan & Context Discovery (User-Specified Docs Only)

Locate the design document, implementation plan, brief, or requirements specification **only when explicitly specified by the user**:
1. **`plan:` argument.** If the caller passed an explicit path (`plan:<path>`), use it directly. Read the file to confirm it exists (`plan_source: explicit`).
2. **Explicitly Designated In-Context Document.** If the user explicitly referenced, attached, or designated a design document, implementation plan, or brief in their invocation prompt as the plan to verify against, use it (`plan_source: user-specified context doc`).
3. **Do not automatically scrape or auto-discover.** If the user did not specify a plan via `plan:` or an explicit reference, **do not automatically scrape random documents found in context or search disk paths**. Requirements verification is additive and only runs against user-specified specifications.

When a plan *is* specified by the user, quickly call out in the Stage 3 team announcement that requirements are being verified against that specified document (e.g., `> [!NOTE] Verifying requirements against specified plan: docs/plans/feat-foo.md`). If no plan was specified, do not call out its absence.

If a specified plan is found, read its **Requirements** section and **Implementation Units** (numeric subsections or task items). Store the extracted requirements list and `plan_source` for Stage 6.

When the specified source of truth carries approved design decisions or Key Technical Decisions (whether explicitly marked with `session-settled:` or clearly stated as an approved design decision), extract each labeled decision and rejected alternative for your own use in Stage 5 triage (step 6c). Settlement annotations are **orchestrator-only context**: exclude them from the Stage 2 intent summary and from every reviewer bundle. Reviewer independence is the point: lenses must stay free to re-derive the rejected alternative on the merits; the orchestrator triages settlement conflicts post-hoc.

## Stage 2c: Keep Grounding Review-Specific

Use the project's active instructions already in context plus the current diff and source. Give each reviewer only the task-relevant context for its lens; the `project-standards` reviewer reads the actual standards sources. If a reviewer cannot scope the affected area from the diff and supplied context, allow one targeted probe.

In `pr-remote` / `branch-remote`, current source and any targeted probe must use `git show` against the supplied reviewed head ref, or the supplied diff hunks when no head ref is available; never inspect workspace paths.
