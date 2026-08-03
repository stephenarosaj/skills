# Persona Roster Selection

The `skill-evaluator` relies on a multi-layered conditional architecture to select personas.

## Always-On Roster
These 6 personas are selected for *every* skill evaluation:
- `correctness-reviewer`
- `testing-reviewer`
- `maintainability-reviewer`
- `project-standards-reviewer`
- `agent-native-reviewer`
- `ux-reviewer`

## Conditional Roster
Scan the target `SKILL.md` for the following heuristics to determine if these personas should be activated:
- `security-reviewer`: Activate if the skill instructs file writes (e.g., `write_file`, `edit_file`, "save to file") or arbitrary command execution.
- `performance-reviewer`: Activate if the skill instructs reading massive files, logs, or databases.
- `api-contract-reviewer`: Activate if the skill instructs usage of external APIs or MCPs.
- `reliability-reviewer`: Activate if the skill has more than 3 distinct execution steps.
- `adversarial-reviewer`: Activate if the skill is >100 lines long or contains execution loops.
