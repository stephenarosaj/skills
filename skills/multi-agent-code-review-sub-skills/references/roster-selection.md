# Roster Selection, Standards Discovery & Run Staging (Stage 3, 3b, 3c & 3d)

## Stage 3: Select Reviewer Team

Read the diff and file list from Stage 1 and helper JSON from Stage 1b. Read `references/persona-catalog.md` for per-persona selection criteria and spawn gates.
- Correctness is always-on.
- `project-standards` is governed by Stage 3b path results.
- Select generic reviewers before domain reviewers:
  - `testing` for changed test/harness surfaces, or when meaningful runtime behavior changed without corresponding test work (require concrete diff evidence such as new/changed branches, state mutation, API/control-flow behavior, error handling).
  - `maintainability` for large or structural work (refactoring, file moves, complexity changes).
  - `agent-native` for agent-facing work.
  - `learnings-researcher` only after a search finds matches in `<root>/solutions/`.
- Instruction-prose files (Markdown skills, JSON schemas, configs) do not benefit from runtime-focused reviewers unless they describe auth, payment, data-mutation, or are silent-pass verification mechanisms (CI/CD workflows, build/deploy gating logic).
- `previous-comments` is PR-only AND comment-gated (`hasPriorComments: true`).
- `data-migration` / `deployment-verification-agent` fire only when schema/migration artifacts (`db/migrate/*`, `db/schema.rb`, `structure.sql`, Alembic/Flyway/Liquibase) are present.

## Stage 3b: Discover Project Standards Paths

Before spawning subagents, find file paths (not contents) of all relevant standards files for the `project-standards` persona:
1. Search for all `**/CLAUDE.md`, `**/AGENTS.md`, and any agent instruction/rule markdown files inside `./agents/` and `.agents/` (`.agents/rules/*.md`, `.agents/*.md`, `./agents/*.md`, etc.) in the repo.
2. Filter to those whose directory is an ancestor of at least one changed file, or global repository-level agent instruction files in `.agents/` / `./agents/`.

- **One or more applicable paths:** Select `project-standards` and pass the path list inside a `<standards-paths>` block in Stage 4 context.
- **Empty successful search:** Do not dispatch `project-standards`; record `project standards: not run (no applicable standards files)` in Coverage.
- **Search failure / uncertain scope:** Fail closed by dispatching `project-standards` with uncertainty stated.

## Stage 3c: Small-Diff Fast Path (Lite Roster)

- **`depth:full` hard-disables this gate.**
- **This gate fails closed: it only ever fires for a positive count of low-risk application code (1-39 executable lines, zero uncounted files, no path signals), AND no content-based risk in Stage 3, AND Stage 3b completed successfully, AND no conditional persona other than `project-standards` was selected.**
- **Lite roster:** Inline fast pass (Stage 4) plus `correctness-reviewer`, and `project-standards-reviewer` only when Stage 3b found applicable paths. Announce actual roster plainly and note in Coverage.
- Do not collapse when any gate condition fails; when in doubt, run the full roster.

## Stage 3d: Materialize Final Roster & Run Directory

Complete this stage before reading persona prompt assets or entering Stage 4.
Generate the review run ID:
```bash
SCRATCH_ROOT="/tmp/compound-engineering-$(id -u)";
if [ -L "$SCRATCH_ROOT" ]; then echo "unsafe scratch root symlink: $SCRATCH_ROOT" >&2; exit 1; fi;
(umask 077; mkdir -p "$SCRATCH_ROOT") || exit 1;
if [ -L "$SCRATCH_ROOT" ] || [ ! -O "$SCRATCH_ROOT" ]; then echo "scratch root is not owned by the current user: $SCRATCH_ROOT" >&2; exit 1; fi;
chmod 700 "$SCRATCH_ROOT" || exit 1;
RUN_ID=$(date +%Y%m%d-%H%M%S)-$(head -c4 /dev/urandom | od -An -tx1 | tr -d ' ');
RUN_DIR="$SCRATCH_ROOT/multi-agent-code-review-sub-skills/$RUN_ID";
(umask 077; mkdir -p "$RUN_DIR") || exit 1; chmod 700 "$RUN_DIR" || exit 1;
echo "$RUN_DIR";
```
Announce the final team before spawning as a user-facing summary: name always-on reviewers plainly, and for each conditional reviewer give the one-line reason it was added. Do not put local reviewer model-tier labels (`[session model]`/`[mid-tier]`) in this announce — those are internal.
