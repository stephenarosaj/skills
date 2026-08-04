# Agent Skills

A collection of agent skills designed to extend and empower AI coding assistants and autonomous agent frameworks.

You can start using them in one of two ways:
- Add using the skill's GitHub URL: `npx skills add <https link to skill folder on github>`
- Clone the repo and use the local path: `npx skills add <path/to/skill/folder>`

---

## Skills Directory

Skills in this repository are categorized below:

### 🔀 Version Control & GitHub

| Skill | Description | Link |
| :--- | :--- | :--- |
| **`generate-pr-description`** | Generates clean, structured Pull Request descriptions with auto-synthesized titles, sparkle-prefixed overviews & highlights, and collapsible GFM changelogs split by implementation and test changes. | [Skill Folder](./skills/generate-pr-description/) |

### 🔍 Code Review & Verification

| Skill | Description | Link |
| :--- | :--- | :--- |
| **`multi-agent-code-review`** | Structured code review using tiered persona agents (17 personas including Blast Radius & Impact), confidence-gated findings, fuzzy plan/context requirements verification, and a merge/dedup pipeline. | [Skill Folder](./skills/multi-agent-code-review/) |

### 🛠️ Agent Context

| Skill | Description | Link |
| :--- | :--- | :--- |
| **`skill-evaluator`** | A meta-skill that combines multi-persona static analysis with empirical quantitative evaluation loops to critique, patch, and benchmark agent skills. | [Skill Folder](./skills/skill-evaluator/) |