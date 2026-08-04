---
name: generate-pr-description
description: Generate a clean, structured Pull Request (PR) description with a generated PR title, sparkle-prefixed Overview and Highlights, and a collapsible Changelog dropdown separating implementation and test changes. Use whenever the user asks to generate a PR description, summarize branch changes, or draft release notes for a diff.
argument-hint: "[mode:agent] [mode:codeblock] [base:<ref>] [PR/branch]"
---

# Generate PR Description (generate-pr-description)

A skill for creating informative, well-structured Pull Request summaries that help reviewers understand changes instantly.

## When to use this skill

- Whenever asked to write a description or summary for a GitHub pull request ("PR") or branch diff.
- Can be invoked standalone (outputs inside a markdown code block by default for easy one-click copying).
- Can run inside larger workflows or automated agent pipelines; use `mode:agent` when the caller needs raw markdown directly without enclosing code block fences.

## Argument Parsing

Parse the arguments you were invoked with for optional tokens (`mode:agent`, `mode:codeblock`, `base:<ref>`). Strip each recognized token before interpreting the remainder as a PR number, GitHub URL, or branch name.

| Token | Example | Effect |
|-------|---------|--------|
| `mode:agent` | `mode:agent` | **Direct raw markdown output**: outputs the raw markdown directly without surrounding code block fences, suitable for programmatic callers, automated PR creation workflows, or direct consumption. |
| `mode:codeblock` | `mode:codeblock` | **Default**: outputs the PR description enclosed in a ````markdown ```` code block for easy one-click copying in chat interfaces. |
| `base:<sha-or-ref>` | `base:main` or `base:origin/main` | Diff base on the **current checkout** (skips auto base detection). |

**Mode aliases**: `mode:raw` and `mode:direct` normalize to `mode:agent`.

## Output format

| Invocation | Deliverable |
|------------|-------------|
| **Default** (Human / Interactive) | 1-line status banner + PR description wrapped in a ````markdown ```` code block |
| **`mode:agent`** (Programmatic / Direct) | Raw markdown PR description directly (no surrounding ````markdown ```` code fences) |
| **Explicit file request** | Written to the specified file path only when requested |

Default and `mode:agent` are **report-only**. `mode:agent` changes only the serialization from a fenced code block to raw unquoted markdown for programmatic callers or direct agent workflows; it does not change the PR description structure or diff analysis rules.

## Tool Selection Priority

1. **Primary**: Use GitHub MCP server tools if available.
2. **Secondary**: Use GitHub CLI (`gh`) if installed and authenticated.
3. **Fallback**: Use local `git` CLI commands (`git diff`, `git log`, `git status`).

## Instructions

1. **Determine PR Context & Target Branch**:
   * If `base:` is provided, use the provided ref as the diff base directly.
   * If a PR number is specified, retrieve its details via GitHub MCP or `gh pr view`.
   * If no PR number is specified, determine the current branch and default target branch (`main`, `master`, or base branch from `git symbolic-ref refs/remotes/origin/HEAD`).
   * If no PR exists yet on GitHub, do NOT halt; proceed using local branch diffs relative to the target branch (`git diff origin/<target-branch>...HEAD` and `git diff HEAD` for uncommitted changes).

2. **Analyze Changes**:
   * Inspect the diff against the target branch.
   * Review commit messages, PR title, and modified files for context.

3. **Draft Description & Output**:
   * Construct the PR description adhering strictly to [`assets/pr_template.md`](assets/pr_template.md) and the structure rules below.
   * Suppress conversational preamble or postscript commentary. Output a clean 1-line status banner (e.g. `📄 PR description generated for branch: <branch-name>`) followed directly by the PR description according to the active mode:
     * **Default (Human / Interactive)**: Wrap the markdown in a ````markdown ```` code block so it can be easily copied with a single click.
     * **`mode:agent`**: Output the raw markdown description directly without surrounding code block fences so automated callers or agent workflows can consume it directly.
   * Do NOT write persistent files (such as `PrDescription.md`) unless the user explicitly requests saving to a file.

## PR Description Structure

PR descriptions **MUST** be formatted in GitHub-flavored markdown adhering strictly to this layout:

### 1. Title Header (`# <PR Title>`)
* Every PR description **MUST** begin with a top-level `# <PR Title>` header containing a concise, high-level title generated for the PR (e.g., `# Add SQLite Caching Layer to DataConnect`). Synthesize the diff and commit messages into this title.

### 2. Overview (`## Overview`)
* Heading: `## Overview`
* The overview content **MUST** start with a sparkle emoji (`✨`).
* Write a 1-2 sentence summary synthesizing the generated PR title and the overarching theme of the changes.

### 3. Highlights (`## Highlights`)
* Heading: `## Highlights`
* The highlights content **MUST** start with a sparkle emoji (`✨`) (e.g., `✨ Key changes:` followed by bullet points).
* Summarize notable changes, new features, major refactors, or user-facing fixes in a concise bulleted list. Group related changes into single bullet points.

### 4. Changelog (`<details><summary><b>Changelog</b></summary>...`)
* Enclosed inside a `<details>` collapsible block.
* **CRITICAL GFM RULE**: Blank lines **MUST** follow `<summary>` and precede `</details>` for inner markdown to render properly:
  ```html
  <details>

  <summary><b>Changelog</b></summary>

  ### Implementation Changes
  * **FileName.ext**
    * Terse bullet point describing code modification

  ### Test Changes
  * **TestFileName.ext**
    * Terse bullet point describing test modification

  </details>
  ```
* Separate implementation changes and test changes into distinct `### Implementation Changes` and `### Test Changes` sub-sections.
* List individual affected files using **ONLY THE FILE NAME** without full directory paths (unless two files have identical names, in which case use the shortest path necessary to disambiguate).
* **Test Inclusion**: Only list tests that were added or modified in code (do NOT list tests that were merely executed). If no test files were modified, write `* No test changes included` under `### Test Changes`.
* **Terseness Rule**: Keep bullets extremely brief.
  * ❌ *Created a new helper function to facilitate X across multiple modules*
  * ✅ *Added helper function for X*

## Error Handling

If any git or tool command fails, abort and report the exact error to the user.

## Bundled Resources

* **Template layout**: [pr_template.md](assets/pr_template.md)
* **Reference example 1**: [PR7714.md](assets/PR7714.md)
* **Reference example 2**: [PR7720.md](assets/PR7720.md)
* **Reference example 3**: [PR7759.md](assets/PR7759.md)
* **Branch PR lookup guide**: [GetCurrentGitBranchPrNumber.md](references/GetCurrentGitBranchPrNumber.md)

*(Note: Reference examples are derived from Kotlin SDK PRs but serve strictly as stack-agnostic layout templates).*
