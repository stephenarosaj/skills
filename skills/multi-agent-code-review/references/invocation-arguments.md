# Invocation Arguments & Token Parsing

Parse the arguments you were invoked with for optional tokens. Strip each recognized token before interpreting the remainder as a PR number, GitHub URL, or branch name.

## Argument Tokens

| Token | Example | Effect |
|-------|---------|--------|
| `mode:agent` | `mode:agent` | **Report-only**: return **JSON** instead of markdown tables and skip the Stage 5c apply (the caller applies). Does not change reviewer selection, merge logic, or scope rules |
| `mode:headless` | `mode:headless` | **Deprecated alias** for `mode:agent` |
| `apply:local` | `apply:local` | Explicitly authorize Stage 5c to apply verified findings to the reviewed local checkout. This is authority, not an output mode; bare review remains report-only. |
| `base:<sha-or-ref>` | `base:abc1234` or `base:origin/main` | Diff base on the **current checkout** (explicit; skips auto base detection) |
| `plan:<path>` | `plan:<root>/plans/feat-foo-plan.md` | Plan file for requirements verification (explicit). Supports markdown and HTML unified plans. |
| `depth:full` | `depth:full` | **Force the full reviewer roster** — skip the Stage 3c small-diff lite path so every always-on persona runs regardless of diff size. Use when a deep/thorough review is explicitly requested. |
| `depth:auto` | `depth:auto` | **Default** — self-right-size via Stage 3c (lite roster for trivial, low-risk, code-only diffs; full roster otherwise). |
| `grouping:auto` | `grouping:auto` | **Default** — build thematic triage groups when findings span distinct concerns (Stage 5 step 9b) |
| `grouping:off` | `grouping:off` | Suppress triage groups: no Triage Groups section, empty `triage_groups` in JSON |
| `grouping:always` | `grouping:always` | Always build triage groups, even for small reviews |

**Grouping is presentation, not a mode.** The `grouping:` tokens change how the finding set is organized for triage — never reviewer selection, merge logic, scope rules, or the Stage 5c apply decision.

## Conflicting Arguments

Stop without dispatching reviewers when:
- Multiple incompatible scope selectors appear together (e.g. `base:` **and** a PR number/branch target — `base:` means "review the current checkout against this base")
- Multiple distinct `mode:` tokens other than the `mode:agent`/`mode:headless` alias pair
- `mode:non-interactive` (alone or with other modes) — not valid for this skill; use `mode:agent` for JSON
- `apply:local` together with `mode:agent` — pipeline handoffs are always report-only
- Multiple distinct `grouping:` tokens (e.g. `grouping:off` **and** `grouping:always`)

Deprecated `mode:autofix` is **not** a conflict — ignore the token and proceed with the normal flow.
Emit a one-line failure reason. In `mode:agent`, return JSON: `{"status":"failed","reason":"..."}`.
