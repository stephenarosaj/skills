## Code Review Results

**Scope:** Standalone working branch (`main` against merge base `5a8c97265111a566d79182996267af46df4f061a`) — 5 files changed, +753 insertions, -127 deletions (`bce8e60`)
**Intent:** Add comprehensive Johnny Decimal node and child directory lifecycle commands (`jd add`, `rename`, `mv`, `rm`) with context-aware parent resolution from `$PWD`, unnumbered child folder adoption, opt-in shortcut creation, and filesystem sync.
**Mode:** markdown report-only

**Reviewers:**
- **`correctness`** *(Always-on)* — Core logic, edge cases, off-by-one errors, null/empty state resilience, and state transition correctness.
- **`testing`** *(Conditional)* — Added/changed tests in `test_jd_engine.py` and new lifecycle runtime behavior (`jd add`, `rename`, `mv`, `rm`); enforcing 100% changed-path coverage and standalone repro validation.
- **`maintainability`** *(Conditional)* — Structural refactoring across `jd_engine.py` and `jd.zshrc` (+753 insertions across 5 files), checking for complexity deletion, coupling, and clean command architecture.
- **`agent-native`** *(Conditional)* — Agent-accessible CLI capabilities and opt-in shortcut generation (`agy*`, `jet*`, `edit*`, `cd*`) for automated Johnny Decimal navigation.
- **`security`** *(Conditional)* — Physical filesystem mutations (`os.remove`, `shutil.rmtree`, `os.rename`, `os.makedirs`), path traversal prevention, and CLI argument handling in `jd_engine.py`.
- **`reliability`** *(Conditional)* — Error handling, edge-case resilience, directory existence checks, and empty/null state safety during schema and filesystem operations.
- **`adversarial`** *(Conditional)* — Large diff threshold (>=50 changed code lines) involving persistence writes to `jd_schema.yaml` and filesystem directory structures; checking for race conditions, assumption violations, and cascade failures.
- **`blast-radius`** *(Conditional)* — Modifications to shared core utility `args.zshrc` (`process_args`) and multi-module Johnny Decimal CLI wrappers; checking for out-of-scope side effects and transitive breakage.
- *`project-standards` (Skipped)* — No applicable project standards files (`CLAUDE.md`, `AGENTS.md`, or `./agents/` / `.agents/`) found in repository root.

---

### Triage Groups

| Group | Findings | Context | Preferred Resolution | Why |
|-------|----------|---------|----------------------|-----|
| Filesystem Path Safety & State Desynchronization *(apply-queue)* | #1, #2, #10, #14 | `jd_engine.py` modifies `jd_schema.yaml` before physical directory operations (`os.rename`, `shutil.move`, `shutil.rmtree`), without path-traversal checks (`..`), destination directory existence checks, or non-empty directory checks | Do #1 and #2 first. Extract a safe path-containment helper (`#1`) and destination existence check (`#10`). Move schema persistence (`save_schema`) to *after* physical disk operations succeed (`#2`), and require `--recursive` before deleting non-empty folders (`#14`) | Encapsulating path verification and ordering disk operations before schema persistence prevents permanent schema-disk desynchronization and accidental data loss |
| Johnny Decimal Node ID Allocation & Lookup *(apply-queue)* | #8, #9, #11, #12, #18 | `jd_engine.py` indexes nodes by `str(node["id"])` in `build_tree_paths` (overwriting duplicate unnumbered child names like `apps` or `demo`), uses ad-hoc string checks for numbering (`isdigit()`, `-`), allows moving a node into its own descendant, and silently resolves the first matching lowercase name | Do #8 and #12 first. Extract canonical helpers `is_numbered_container` and `is_numbered_node` (`#12`). In `build_tree_paths`, index numbered nodes by ID and unnumbered children by path (`#8`). Add cycle prevention to `cmd_mv` (`#9`), fix node-type checks when moving across areas/categories (`#11`), and error on ambiguous bare name lookups (`#18`) | Fixing the ID map index and centralizing node-type classification eliminates lookups operating on the wrong relative path and prevents schema corruption during moves |
| Shell Wrapper & Alias Integrity *(apply-queue)* | #4, #5, #6, #17 | `jd.zshrc` reads sub-command outputs without tab IFS (`#4`), writes unquoted variables to the shortcut cache (`#5`), does not re-source the cache when external agents mutate the schema (`#6`), and leaves stale aliases in memory after renaming or removal (`#17`) | Do #4 and #5 first. Prefix all `read -r` invocations with `IFS=$'\t'` (`#4`) and quote variable expansions when emitting `.cache/jd_materialized_shortcuts.zshrc` (`#5`). Add a precmd cache-timestamp check (`#6`) and a helper to unalias removed shortcuts during `sync_jd_shortcuts` (`#17`) | Protecting tab-delimited parsing and variable quoting prevents whitespace mangling and shell command injection, while automatic cache reloading ensures shell state stays synchronized |
| Out-of-Scope Architecture & Agent Documentation *(apply-queue)* | #3, #7, #16 | The PR modifies shared core utility `args.zshrc` to inject a default help flag (`#16`), checks machine-local `.cache/jd_materialized_shortcuts.zshrc` into git tracking (`#3`), and leaves `jd_schema.yaml` undocumented in `.agents/skills/zshrc/SKILL.md` (`#7`) | Do #16 first. Revert `args.zshrc` edits and define `-h/--help` in `jd()`'s schema (`#16`). Remove `.cache/jd_materialized_shortcuts.zshrc` from version control (`#3`), and update `.agents/skills/zshrc/SKILL.md` with Johnny Decimal node lifecycle instructions (`#7`) | Keeping `args.zshrc` untouched prevents unintended global side-effects across all Zsh commands, while untracking cache files and documenting schemas enables safe agent automation |
| Test Fidelity & Recursive Workflow Coverage *(apply-queue)* | #13, #15 | `test_jd_engine.py` skips creating physical directories before testing moves and deletions (`#15`), and lacks tests for recursive directory removal and child node checks (`#13`) | Do #15 first. Add physical directory setup (`os.makedirs`) and assertion (`os.path.isdir`) to `test_move_node` (`#15`), and add tests for `cmd_rm` with and without `--recursive` (`#13`) | Testing physical directory moves and recursive deletions against disk ensures test suites catch regression failures in production filesystem workflows |

---

### P0 -- Critical

| # | File | Issue | Reviewer | Confidence |
|---|------|-------|----------|------------|
| 1 | `shared/jd/jd_engine.py:468` | Path traversal allows arbitrary directory deletion and modification | security | 100 |
| 2 | `shared/jd/jd_engine.py:468` | Schema saved before fallible filesystem operations causing permanent state desynchronization | adversarial, blast-radius, maintainability, reliability | 100 |

- **#1** — User-supplied relative path in `os.path.join(base_dir, rel_path)` is not checked for directory traversal (`../`) before `os.makedirs`, `os.rename`, `shutil.move`, `shutil.rmtree`, or `os.remove`. An attacker or erroneous input could traverse outside `base_dir` and delete or rename arbitrary system directories. Add a helper verifying `os.path.realpath(full_path).startswith(os.path.realpath(base_dir) + os.sep)` before any disk operation.
- **#2** — `save_schema(schema, schema_path)` is invoked before physical directory operations (`os.rename`, `shutil.move`, `shutil.rmtree`) succeed. If a disk operation fails due to permissions, existing destination files, or read-only directories, `jd_schema.yaml` remains permanently desynchronized from the physical filesystem. Perform filesystem operations before saving the schema, or wrap disk mutations in a `try...except` block that rolls back the in-memory schema on failure.

---

### P1 -- High

| # | File | Issue | Reviewer | Confidence |
|---|------|-------|----------|------------|
| 3 | `.cache/jd_materialized_shortcuts.zshrc:1` | Tracked machine-local shortcut cache prevents regeneration across environments | blast-radius | 100 |
| 4 | `shared/jd/jd.zshrc:158` | Zsh read without tab IFS splits Johnny Decimal paths containing spaces | adversarial, correctness | 100 |
| 5 | `shared/jd/jd.zshrc:20` | Shortcut cache generation enables shell command injection on source | security | 100 |
| 6 | `shared/jd/jd.zshrc:34` | Shell shortcuts out of sync after agent-driven Johnny Decimal mutations | agent-native | 100 |
| 7 | `shared/jd/jd.zshrc:48` | Johnny Decimal schema and lifecycle commands undocumented in agent skill | agent-native | 100 |
| 8 | `shared/jd/jd_engine.py:489` | Global ID map overwrites duplicate unnumbered folder names | correctness | 100 |
| 9 | `shared/jd/jd_engine.py:519` | Abuse case: Moving node into descendant creates cyclic schema recursion | adversarial | 100 |
| 10 | `shared/jd/jd_engine.py:533` | shutil.move nests directory inside destination when target directory already exists on disk | adversarial, reliability | 100 |
| 11 | `shared/jd/jd_engine.py:792` | Node move corrupts IDs across areas and unnumbered folders | correctness | 100 |
| 12 | `shared/jd/jd_engine.py:794` | Divergent ad-hoc node type classification across lifecycle commands | maintainability | 100 |
| 13 | `shared/jd/jd_engine.py:828` | Recursive directory removal and child node check are untested | testing | 100 |
| 14 | `shared/jd/jd_engine.py:845` | Disk removal deletes non-empty directories without recursive flag | correctness | 100 |
| 15 | `shared/jd/test_jd_engine.py:197` | test_move_node skips physical directory creation and post-move assertion on disk | adversarial, testing | 100 |
| 16 | `shared/util/args.zshrc:195` | Out-of-scope help flag injection in shared process_args utility | blast-radius | 100 |

- **#3** — `.cache/jd_materialized_shortcuts.zshrc` is tracked in git version control instead of being ignored, causing dirty working trees on any machine that sources or regenerates local shortcuts. Remove `.cache/jd_materialized_shortcuts.zshrc` from version control and add `.cache/` to `.gitignore`.
- **#4** — `read -r` invocations parsing tab-delimited output from `jd_engine.py` across `jd add` (line 186), `jd rename` (line 213), `jd mv` (line 234), and `jd rm` (line 261) split relative paths on spaces when tab IFS is omitted. Prefix `read -r` invocations with `IFS=$'\t'` to ensure space-containing folder paths are parsed correctly.
- **#5** — Unquoted variable expansions when generating `.cache/jd_materialized_shortcuts.zshrc` (`echo "PROJECT_SHORTCUTS[$sc_key]=\"\${JD_FOLDER}/${sc_rel_path}\"" >> "$tmp_file"`) could allow shell command injection if a shortcut or node name contains Zsh metacharacters. Escape or quote expansions using `(q)` flag (`${(q)JD_FOLDER}/${(q)sc_rel_path}`).
- **#6** — `sync_jd_shortcuts()` only regenerates the shortcut cache if `.cache/jd_materialized_shortcuts.zshrc` is missing on disk. When an external agent or script mutates `jd_schema.yaml` via `jd add` or `jd rename`, interactive shell sessions retain stale aliases. Add a precmd hook checking cache timestamps against `_JD_CACHE_LAST_SOURCED` to automatically reload updated shortcuts.
- **#7** — `.agents/skills/zshrc/SKILL.md` does not document `jd_schema.yaml` or node lifecycle commands (`jd add`, `rename`, `mv`, `rm`), leaving AI agents unaware of how to allocate nodes or navigate Johnny Decimal workspaces safely. Document the schema and commands in `.agents/skills/zshrc/SKILL.md`.
- **#8** — In `build_tree_paths`, indexing nodes by `str(node["id"])` (`results[str(node["id"])] = entry`) overwrites entries when unnumbered child folders in different tree branches share the same name (e.g. `10.00/apps/demo` and `11.02/games/demo`). As a result, `cmd_add` and `cmd_mv` look up and operate on the wrong `rel_path`. Only index numbered nodes by ID; index unnumbered children by relative path (`path:...`).
- **#9** — `cmd_mv` lacks a cycle-detection check before reparenting a node, allowing a category or node to be moved inside its own descendant and creating infinite recursion in `build_tree_paths` and `dump_yaml_node`. Climb the parent chain from `new_parent_node` before attaching to verify `target_node` is not encountered.
- **#10** — Calling `shutil.move(old_full, new_full)` without checking `os.path.exists(new_full)` silently moves `old_full` *inside* `new_full` if the destination directory exists on disk, corrupting physical folder alignment. Check `if os.path.exists(new_full):` and raise an explicit `FileExistsError` before moving.
- **#11** — Using `if new_parent_id.isdigit():` in `cmd_mv` fails to reallocate node IDs when moving a category (`00`) into an area (`10-19`), or moving unnumbered child folders into numbered categories. Replace with an explicit check reallocating slots whenever moving across container boundaries.
- **#12** — Ad-hoc checks for categories and areas (`isdigit()`, `"-" in id`) are duplicated across `cmd_add`, `cmd_mv`, `format_folder_name`, and `get_next_available_id`. Extract canonical helper functions `is_numbered_container(node)` and `is_numbered_node(node)` and reuse them across all commands.
- **#13** — `test_jd_engine.py` lacks unit tests for `cmd_rm` with and without `--recursive` (`-r`) on nodes with children. Add automated test methods verifying that `cmd_rm` raises `SystemExit` without `-r` when children exist, and recursively removes child entries and directories when `-r` is passed.
- **#14** — In `cmd_rm`, `shutil.rmtree(full_path)` deletes non-empty physical directories even when `--recursive` (`-r`) is omitted on the command line. Before calling `shutil.rmtree`, check `if not recursive and os.listdir(full_path):` and raise an error instructing the user to pass `--recursive` (`-r`).
- **#15** — `test_move_node` in `test_jd_engine.py` tests schema relocation but skips creating physical source directories (`os.makedirs`) and asserting folder moves on disk (`os.path.isdir`), masking physical move defects. Create temporary source folders before `cmd_mv` and assert their relocation after.
- **#16** — The PR injects `-h/--help` handling directly into `args.zshrc` (`process_args`), creating shared global side-effects across all Zsh functions in the repository. Revert `args.zshrc` modifications and declare `"help|h:flag:::Display this help message"` explicitly in `jd()`'s schema array (`shared/jd/jd.zshrc:48`).

---

### P2 -- Moderate

| # | File | Issue | Reviewer | Confidence |
|---|------|-------|----------|------------|
| 17 | `shared/jd/jd.zshrc:89` | Stale shell shortcuts persist after removal or renaming | maintainability | 100 |
| 18 | `shared/jd/jd_engine.py:558` | Ambiguous fallback name resolution silently operates on first match | maintainability | 100 |

- **#17** — `sync_jd_shortcuts()` does not clear `PROJECT_SHORTCUTS` or unalias old shortcut names (`cd<old>`, `edit<old>`, `jet<old>`, `agy<old>`) when shortcuts are removed or renamed. Create a helper that unaliases existing shortcut aliases and resets `PROJECT_SHORTCUTS=()` before sourcing the refreshed cache.
- **#18** — `resolve_node_entry`'s case-insensitive name fallback loop silently operates on the first match when multiple nodes share a bare name across categories. Collect all name matches and print an error listing ambiguous node IDs and paths when more than one match exists.

---

### Actionable Findings

| # | File | Issue | Route | Notes |
|---|------|-------|-------|-------|
| 1 | `shared/jd/jd_engine.py:468` | Path traversal allows arbitrary directory deletion and modification | `gated_auto -> downstream-resolver` | `suggested_fix` present — verify path containment before disk operations |
| 2 | `shared/jd/jd_engine.py:468` | Schema saved before fallible filesystem operations causing permanent state desynchronization | `gated_auto -> downstream-resolver` | `suggested_fix` present — reorder disk mutations before `save_schema` |
| 3 | `.cache/jd_materialized_shortcuts.zshrc:1` | Tracked machine-local shortcut cache prevents regeneration across environments | `gated_auto -> downstream-resolver` | `suggested_fix` present — remove from git tracking and ignore `.cache/` |
| 4 | `shared/jd/jd.zshrc:158` | Zsh read without tab IFS splits Johnny Decimal paths containing spaces | `gated_auto -> downstream-resolver` | `suggested_fix` present — prefix `read -r` with `IFS=$'\t'` |
| 5 | `shared/jd/jd.zshrc:20` | Shortcut cache generation enables shell command injection on source | `gated_auto -> downstream-resolver` | `suggested_fix` present — quote variable expansions using `(q)` flag |
| 6 | `shared/jd/jd.zshrc:34` | Shell shortcuts out of sync after agent-driven Johnny Decimal mutations | `gated_auto -> downstream-resolver` | `suggested_fix` present — add precmd cache-timestamp check |
| 7 | `shared/jd/jd.zshrc:48` | Johnny Decimal schema and lifecycle commands undocumented in agent skill | `gated_auto -> downstream-resolver` | `suggested_fix` present — document schema and commands in `.agents/skills/zshrc/SKILL.md` |
| 8 | `shared/jd/jd_engine.py:489` | Global ID map overwrites duplicate unnumbered folder names | `gated_auto -> downstream-resolver` | `suggested_fix` present — index unnumbered children by path in `build_tree_paths` |
| 9 | `shared/jd/jd_engine.py:519` | Abuse case: Moving node into descendant creates cyclic schema recursion | `gated_auto -> downstream-resolver` | `suggested_fix` present — check parent chain for ancestry cycles in `cmd_mv` |
| 10 | `shared/jd/jd_engine.py:533` | shutil.move nests directory inside destination when target directory already exists on disk | `gated_auto -> downstream-resolver` | `suggested_fix` present — assert destination directory does not exist before moving |
| 11 | `shared/jd/jd_engine.py:792` | Node move corrupts IDs across areas and unnumbered folders | `gated_auto -> downstream-resolver` | `suggested_fix` present — reallocate IDs whenever moving across container boundaries |
| 12 | `shared/jd/jd_engine.py:794` | Divergent ad-hoc node type classification across lifecycle commands | `gated_auto -> downstream-resolver` | `suggested_fix` present — extract canonical `is_numbered_container` helper |
| 13 | `shared/jd/jd_engine.py:828` | Recursive directory removal and child node check are untested | `gated_auto -> downstream-resolver` | `suggested_fix` present — add automated tests for `cmd_rm` with and without `-r` |
| 14 | `shared/jd/jd_engine.py:845` | Disk removal deletes non-empty directories without recursive flag | `gated_auto -> downstream-resolver` | `suggested_fix` present — require `--recursive` before `shutil.rmtree` |
| 15 | `shared/jd/test_jd_engine.py:197` | test_move_node skips physical directory creation and post-move assertion on disk | `gated_auto -> downstream-resolver` | `suggested_fix` present — setup physical source folders and assert relocation |
| 16 | `shared/util/args.zshrc:195` | Out-of-scope help flag injection in shared process_args utility | `gated_auto -> downstream-resolver` | `suggested_fix` present — revert `args.zshrc` and declare `--help` in `jd()` schema |
| 17 | `shared/jd/jd.zshrc:89` | Stale shell shortcuts persist after removal or renaming | `gated_auto -> downstream-resolver` | `suggested_fix` present — unalias removed shortcuts before sourcing cache |
| 18 | `shared/jd/jd_engine.py:558` | Ambiguous fallback name resolution silently operates on first match | `gated_auto -> downstream-resolver` | `suggested_fix` present — error when multiple nodes share a bare name |

---

### Agent-Native Gaps

- **CLI inspection commands lack structured JSON output mode for agents (`shared/jd/jd_engine.py:479` | P2 Moderate):** Add a `--json` flag to `jd_engine.py` (`jd ls --json` / `jd shortcuts --json`) to serialize node entries directly to structured JSON so agents can query the tree without regex-parsing human presentation text.
- **Johnny Decimal schema undocumented in Zsh skill (`shared/jd/jd.zshrc:48` | P1 High):** Update `.agents/skills/zshrc/SKILL.md` to document `jd_schema.yaml` as the source of truth for `PROJECT_SHORTCUTS` and instruct agents to use `jd add`, `rename`, `mv`, and `rm` for node lifecycle operations.
- **Interactive shell shortcuts out of sync after agent mutations (`shared/jd/jd.zshrc:34` | P1 High):** Implement a precmd cache-timestamp check so interactive shells automatically reload `.cache/jd_materialized_shortcuts.zshrc` when an agent mutates the schema.

---

### Coverage

- **Suppressed:** 0 findings suppressed below anchor 75 (25 soft-bucket / absence-of-coverage items demoted or consolidated during synthesis).
- **Residual risks:**
  - Concurrent invocations of `jd add`, `rename`, `mv`, or `rm` from multiple terminal tabs have no file locking on `jd_schema.yaml`, risking race conditions and lost updates.
  - Renaming a node with only a case change (e.g. `AI` -> `ai`) on case-insensitive filesystems (macOS default HFS+/APFS) may fail or behave inconsistently without explicit cross-platform filesystem tests.
  - Custom YAML parser edge cases: when PyYAML is not installed, `load_yaml_file` and `save_yaml_file` use an indentation-based fallback parser that could silently misparse if `jd_schema.yaml` is manually edited with non-standard indentation.
- **Testing gaps:**
  - No shell integration or BATS tests for `shared/jd/jd.zshrc` wrapper subcommands (`jd add`, `rename`, `mv`, `rm`, `cd`, `ls`).
  - No unit tests in `test_jd_engine.py` for moving category nodes across different area boundaries (`00` to area `10-19`) or moving unnumbered child folders into categories.
  - No unit tests in `test_jd_engine.py` for `cmd_rm -d` without `-r` on physical directories that contain files on disk but no registered children in `jd_schema.yaml`.
  - No unit tests in `test_jd_engine.py` for name collisions among unnumbered child folders (e.g. multiple nodes having an `apps` or `demo` subfolder) when resolving targets in `resolve_node_entry`.
- **Reviewers completed:** 8/8 (`correctness`, `testing`, `maintainability`, `agent-native`, `security`, `reliability`, `adversarial`, `blast-radius`). All 18 primary findings survived mechanical validation at `confidence: 100`.

---

### Verdict

> **Verdict:** Ready with fixes
>
> **Reasoning:** 2 critical (P0) filesystem path-containment and state-desynchronization bugs must be resolved to prevent arbitrary directory deletion and permanent schema corruption. 14 high-priority (P1) issues covering ID map overwrites, tab IFS whitespace splitting, shell command injection, cyclic move recursion, global `args.zshrc` pollution, and test fidelity should be addressed before merging.
>
> **Fix order:** P0 Path-traversal & Schema-persistence order (`#1`, `#2`) -> P1 ID map indexing & cycle prevention (`#8`, `#9`, `#10`, `#11`, `#12`) -> P1 Shell IFS parsing, quoting & global args revert (`#3`, `#4`, `#5`, `#6`, `#7`, `#16`) -> P1 Test suite hardening (`#13`, `#14`, `#15`) -> P2 Alias cleanup & ambiguous lookup (`#17`, `#18`)

#### Prioritized Actionable List

1. **#1** | `shared/jd/jd_engine.py:468` | [P0] Path traversal allows arbitrary directory deletion and modification -> `gated_auto -> downstream-resolver`
2. **#2** | `shared/jd/jd_engine.py:468` | [P0] Schema saved before fallible filesystem operations causing permanent state desynchronization -> `gated_auto -> downstream-resolver`
3. **#3** | `.cache/jd_materialized_shortcuts.zshrc:1` | [P1] Tracked machine-local shortcut cache prevents regeneration across environments -> `gated_auto -> downstream-resolver`
4. **#4** | `shared/jd/jd.zshrc:158` | [P1] Zsh read without tab IFS splits Johnny Decimal paths containing spaces -> `gated_auto -> downstream-resolver`
5. **#5** | `shared/jd/jd.zshrc:20` | [P1] Shortcut cache generation enables shell command injection on source -> `gated_auto -> downstream-resolver`
6. **#6** | `shared/jd/jd.zshrc:34` | [P1] Shell shortcuts out of sync after agent-driven Johnny Decimal mutations -> `gated_auto -> downstream-resolver`
7. **#7** | `shared/jd/jd.zshrc:48` | [P1] Johnny Decimal schema and lifecycle commands undocumented in agent skill -> `gated_auto -> downstream-resolver`
8. **#8** | `shared/jd/jd_engine.py:489` | [P1] Global ID map overwrites duplicate unnumbered folder names -> `gated_auto -> downstream-resolver`
9. **#9** | `shared/jd/jd_engine.py:519` | [P1] Abuse case: Moving node into descendant creates cyclic schema recursion -> `gated_auto -> downstream-resolver`
10. **#10** | `shared/jd/jd_engine.py:533` | [P1] shutil.move nests directory inside destination when target directory already exists on disk -> `gated_auto -> downstream-resolver`
11. **#11** | `shared/jd/jd_engine.py:792` | [P1] Node move corrupts IDs across areas and unnumbered folders -> `gated_auto -> downstream-resolver`
12. **#12** | `shared/jd/jd_engine.py:794` | [P1] Divergent ad-hoc node type classification across lifecycle commands -> `gated_auto -> downstream-resolver`
13. **#13** | `shared/jd/jd_engine.py:828` | [P1] Recursive directory removal and child node check are untested -> `gated_auto -> downstream-resolver`
14. **#14** | `shared/jd/jd_engine.py:845` | [P1] Disk removal deletes non-empty directories without recursive flag -> `gated_auto -> downstream-resolver`
15. **#15** | `shared/jd/test_jd_engine.py:197` | [P1] test_move_node skips physical directory creation and post-move assertion on disk -> `gated_auto -> downstream-resolver`
16. **#16** | `shared/util/args.zshrc:195` | [P1] Out-of-scope help flag injection in shared process_args utility -> `gated_auto -> downstream-resolver`
17. **#17** | `shared/jd/jd.zshrc:89` | [P2] Stale shell shortcuts persist after removal or renaming -> `gated_auto -> downstream-resolver`
18. **#18** | `shared/jd/jd_engine.py:558` | [P2] Ambiguous fallback name resolution silently operates on first match -> `gated_auto -> downstream-resolver`
