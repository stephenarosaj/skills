# Scope Resolution & Diff Endpoints (Stage 1 & 1b)

## Stage 1: Resolve Diff & Intent

Determine the review target from arguments or git status.
- **`base:<sha-or-ref>`**: Diff base on the current checkout (`DIFF_A="$BASE"`, `DIFF_B` empty). Do not combine `base:` with a PR number or branch target.
- **PR number or GitHub URL**: Scope comes from GitHub read APIs plus optional local alignment when HEAD already matches the PR head branch.

### Skip-Condition Pre-Check (PR Mode)
Before scope detection, run a PR-state probe:
```bash
gh pr view <number-or-url> --json state,title,body,files
```
Apply skip rules in order:
1. `state` is `CLOSED` or `MERGED` -> stop with reason `PR is closed/merged; not reviewing.`
2. **Trivial-PR judgment**: spawn a lightweight subagent on the cheapest capable model to judge if the PR is an automated dependency/release/chore bump with no substantive code changes. If yes -> stop with reason `PR appears to be a trivial automated PR; not reviewing.`
When any skip rule fires, stop without dispatching reviewers (in `mode:agent`, emit JSON: `{"status":"skipped","reason":"<same message>"}`).

### Fetch PR Metadata (Without Checkout)
```bash
gh pr view <number-or-url> --json title,body,baseRefName,headRefName,headRefOid,isCrossRepository,url,files,reviews,comments --jq '{title, body, baseRefName, headRefName, headRefOid, isCrossRepository, url, files: [.files[].path], hasPriorComments: ((.reviews | map(select(.state != "APPROVED" or .body != "")) | length) > 0 or (.comments | length) > 0)}'
```
If `gh pr view` or `gh pr diff` fails or returns malformed JSON, stop with an actionable error advising the user to check `gh auth status` or verify their GitHub CLI version — do not fall back to checkout.

Set `BASE:` to `pr:<number-or-url>` (logical marker). Set `UNTRACKED:` from `git ls-files --others --exclude-standard` on the current checkout.

### PR Scope Classification
Classify as **`local-aligned`** only when ALL of these hold; otherwise use **`pr-remote`**:
1. `git rev-parse --abbrev-ref HEAD` equals `headRefName`.
2. The PR is not cross-repository (`isCrossRepository` is false).
3. The PR head commit is contained in local checkout (`git merge-base --is-ancestor <headRefOid> HEAD` exits 0).

- **`local-aligned`**: Compute `BASE=$(git merge-base HEAD <resolved-base-ref>)`, set `FILES:` from `git diff --name-only $BASE`, and `DIFF:` from `git diff -U10 $BASE`. Do NOT call `gh pr diff` or append remote hunks.
- **`pr-remote`**: Set `FILES:` from PR `files` array and `DIFF:` from `gh pr diff <number-or-url> --color=never`.
  - Before executing `git fetch`, strictly validate `<number>` as numeric (`^[0-9]+$`) and `<headRefName>` against safe POSIX git ref character sets (`^[A-Za-z0-9_./-]+$`). If validation fails, skip fetch and rely on diff hunks only.
  - Fetch PR head (`refs/review/pr-<number>-head`) and PR base (`FETCH_HEAD` -> `PR_BASE_REF`). If either fails, omit ref and rely on diff hunks only.
  - Reviewers and Stage 5b validators in `pr-remote` mode must NOT Read/Grep workspace paths for files in `FILES:`. Inspect via `git show <PR_HEAD_REF>:<path>` or diff hunks.

### Branch Mode
Substitute provided branch name as `<branch>`. Do NOT check out `<branch>`.
- If `git rev-parse --abbrev-ref HEAD` equals `<branch>`, use standalone current branch path.
- Otherwise diff remote/local ref without checkout: prefer open PR URL/number if `gh pr view <branch>` succeeds; else resolve `<branch>` after `git fetch --no-tags origin <branch>` when needed. Compute `BASE=$(git merge-base <base-ref> <branch-ref>)` and `git diff -U10 $BASE <branch-ref>`.

### Standalone (Current Branch)
Apply base-detection logic using current branch (`gh pr view --json baseRefName,url` defaults to current branch). If no base can be resolved, STOP — do not fall back to `git diff HEAD`. Produce diff:
```bash
echo "BASE:$BASE" && echo "FILES:" && git diff --name-only $BASE && echo "DIFF:" && git diff -U10 $BASE && echo "UNTRACKED:" && git ls-files --others --exclude-standard
```
Always inspect `UNTRACKED:`; list excluded files in Coverage and continue on tracked changes only.

## Stage 1b: Compute Scope Signals
Derive deterministic signals once with `scripts/review-scope.py`.
```bash
SKILL_DIR="<absolute path of the directory containing the SKILL.md you just read>";
PY="$(for c in python3 python py; do command -v "$c" >/dev/null 2>&1 && "$c" -c '' >/dev/null 2>&1 && { echo "$c"; break; }; done)";
if [ -n "$PY" ]; then
  if [ "$SCOPE_MODE" = "pr-remote" ] || [ "$SCOPE_MODE" = "branch-remote" ]; then
    "$PY" "$SKILL_DIR/scripts/review-scope.py" --base "${DIFF_A:-}" --head "${DIFF_B:-}" --docs-root "<root>";
  else
    "$PY" "$SKILL_DIR/scripts/review-scope.py" --base "$DIFF_A" --docs-root "<root>";
  fi
else
  echo '{"exec_lines":null,"uncounted_files":1,"signals":{"test_files_changed":true,"agent_surface":true,"has_learnings_corpus":true}}'
fi
```
`exec_lines: null`, any `uncounted_files > 0`, or helper failure disqualifies the small-diff lite path.
