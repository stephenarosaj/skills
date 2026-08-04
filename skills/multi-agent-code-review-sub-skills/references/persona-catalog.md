# Persona Catalog

Reviewer personas organized into a small core plus generic, cross-cutting, and stack-specific conditionals. The orchestrator uses this catalog to select only reviewers whose domain is present in the diff, while ensuring comprehensive coverage of empirical verification risk tiers (`@agentic_workflow_best_practices.md`).

## Core and standards gate

Correctness is spawned on every multi-agent review. Project-standards is spawned only when Stage 3b finds at least one applicable standards file (searching `CLAUDE.md`, `AGENTS.md`, and `./agents/` / `.agents/`), or when standards discovery fails and the review must fail closed.

**Structured persona prompt assets:**

| Persona | Prompt asset | Focus & Workflow Best Practices Alignment |
|---------|--------------|------------------------------------------|
| `correctness` | `correctness-reviewer` | **Edge-Case Reviewer** (`@agentic_workflow_best_practices.md`): Logic errors, edge cases, state bugs, error propagation, intent compliance, boundary conditions, off-by-one errors, null/empty states, and concurrency races. Spawned on every multi-agent review. |
| `project-standards` | `project-standards-reviewer` | **Architecture & Readability Reviewer** (`@agentic_workflow_best_practices.md`): Repository standards compliance (`CLAUDE.md`, `AGENTS.md`, and `.agents/`), frontmatter, references, naming, cross-platform portability, tool selection, **and `@user_global` comment & documentation rules (`/** */` Javadocs for non-trivial headers directed to general readers/users; sparing `//` inline comments for complex "why" rather than redundant "what")**. Spawned when Stage 3b finds an applicable standards file. |

## Generic conditional

These lenses are broadly applicable across codebases but are only spawned when their concrete domain surface is present in the diff.

| Persona / asset | Prompt asset | Select when diff touches... & Workflow Alignment |
|-----------------|--------------|------------------------------------------------|
| `testing` | `testing-reviewer` | **Test Completeness Reviewer** (`@agentic_workflow_best_practices.md`): Audits whether the test suite covers failure modes and negative paths, enforces the **100% changed-path coverage rule** on modified functions and branches, and verifies **Fail-First standalone reproduction test cases** for bug fixes. Select when diff touches test files, test infrastructure, fixtures, mocks, OR when meaningful runtime behavior changed without corresponding test work. Production-file presence alone is insufficient. |
| `maintainability` | `maintainability-reviewer` | **Architecture & Readability Reviewer** (`@agentic_workflow_best_practices.md`): Structural quality, complexity deletion, coupling, type-boundary leaks, dead code, premature abstraction, and adherence to repository design patterns. Select when diff involves **structural work**: substantial refactors, new abstractions, file moves, coupling/type-boundary changes, or architectural modifications **(no line-count threshold required)**. |
| `agent-native` | `agent-native-reviewer` | Verify new features or surfaces are agent-accessible (skills, agents, prompts, commands, tools, MCP, or product capabilities expected to be agent-accessible). Select when diff touches agent-facing surfaces. |
| `learnings` | `learnings-researcher` | Search existing `<root>/solutions/` or knowledge base corpus for matching past issues or patterns. Select only when an existing solutions corpus has a plausible path/title match for the changed modules or patterns after a preliminary search. |

## Cross-cutting conditional (8 personas)

Spawned when the orchestrator identifies relevant domain patterns in the diff. Selection is agent judgment over diff semantics, not keyword matching.

| Persona | Prompt asset | Select when diff touches... & Workflow Alignment |
|---------|--------------|------------------------------------------------|
| `security` | `security-reviewer` | **Security & Threat Model Auditor** (`@agentic_workflow_best_practices.md`): Scans for CWE-classified vulnerabilities, SQL/command/HTML injection risks, auth bypasses, trust boundary violations, permissions, and secrets management. Select when diff touches auth middleware, public endpoints, user input handling, permission checks, or secrets. |
| `performance` | `performance-reviewer` | Concrete performance-sensitive behavior: database/ORM query shape, algorithmic complexity, large loop-heavy transforms, batching/fan-out, or cache policy with material resource impact. |
| `api-contract` | `api-contract-reviewer` | Externally consumed boundaries: route/request/response definitions, serializers, published event schemas, API versioning, or public package signatures. |
| `data-migration` | `data-migration-reviewer` | Migration files, schema dumps (`db/schema.rb`, `structure.sql`), backfill scripts, data transformations, **and Record-and-Replay / behavioral equivalence verification on legacy migrations and refactorings** (`@agentic_workflow_best_practices.md`). Do **not** spawn for model-only or query-only changes without migration artifacts. |
| `reliability` | `reliability-reviewer` | **Edge-Case Reviewer** (`@agentic_workflow_best_practices.md`): Error handling, retry logic, circuit breakers, timeouts, background jobs, async handlers, health checks, null/empty state resilience, and concurrency ordering. |
| `adversarial` | `adversarial-reviewer` | >=50 changed code lines; auth/payments; persistence writes or event publication; retry/partial-failure or concurrency semantics; external APIs; **or a silent-pass verification mechanism** (`@agentic_workflow_best_practices.md`: CI/CD gates, build/deploy steps, coverage/lint gates, test harnesses/mocks that could mask production bugs). |
| `previous-comments` | `previous-comments-reviewer` | **PR-only AND comment-gated.** Reviewing a PR that has existing review comments or review threads from prior review rounds. Skip entirely when no PR metadata was gathered in Stage 1, OR when Stage 1's `hasPriorComments` flag is false. |
| `blast-radius` | `blast-radius-reviewer` | **NEW — Blast Radius / Impact Analyzer** (`@agentic_workflow_best_practices.md`): Checks if the diff modifies files, abstractions, or dependencies outside the required scope (a critical red flag for unprompted side-effects); audits cross-module coupling leaks, transitive breakage risks, and unprompted architectural side-effects. Select when diff touches multiple modules, shared core utilities, public dependencies, or architectural boundaries. |

## Stack-Specific Conditional (2 personas)

These reviewers cover specialized runtime behavior. Structural and maintainability concerns live in the generic conditional `maintainability` persona — do not spawn extra stack reviewers for philosophy or convention-only passes.

| Persona | Prompt asset | Select when diff touches... & Workflow Alignment |
|---------|--------------|------------------------------------------------|
| `julik-frontend-races` | `julik-frontend-races-reviewer` | **Edge-Case Reviewer (Concurrency / UI Races)** (`@agentic_workflow_best_practices.md`): Stimulus/Turbo controllers, DOM event wiring, timers, async UI flows, animations, or frontend state transitions with race potential. |
| `swift-ios` | `swift-ios-reviewer` | Swift files, SwiftUI views, UIKit controllers, `.entitlements`, `PrivacyInfo.xcprivacy`, `.xcdatamodeld`, `Package.swift`, `Package.resolved`, storyboards, XIBs, or semantic build-setting / target-membership / code-signing changes in `.pbxproj`. |

## CE Conditional Prompt Assets (migration-specific, 1 asset)

Use `deployment-verification-agent` when the migration-artifact gate applies **and** the change is risky (destructive DDL, backfills, NOT NULL without default, column renames/drops). Schema drift and migration safety live in the `data-migration` persona.

| Prompt asset | Focus & Workflow Alignment |
|--------------|---------------------------|
| `deployment-verification-agent` | Go/No-Go deployment checklist with SQL verification queries and rollback procedures for risky database migrations. |

## Selection rules

1. **Always spawn `correctness`.** Spawn `project-standards` only when an applicable standards path list is found (searching `CLAUDE.md`, `AGENTS.md`, and `./agents/` / `.agents/`); skip it on a confirmed empty search and fail closed by spawning it when discovery is uncertain.
2. **For each generic conditional**, require its explicit surface or structural threshold:
   - `testing`: requires changed tests/harnesses OR concrete meaningful runtime behavior with no corresponding test work.
   - `maintainability`: requires **structural work** (substantial refactors, new abstractions, file moves, coupling changes, or architectural modifications) without any arbitrary line-count threshold.
   - `agent-native`: requires agent-facing capabilities.
   - `learnings`: requires matching solutions corpus documents.
3. **For each cross-cutting conditional persona**, read the diff and decide whether its domain is relevant. This is agent judgment, not keyword matching.
4. **For `blast-radius` (Blast Radius / Impact Analyzer)**, spawn whenever the diff touches multiple modules, shared core abstractions, dependencies, or architectural boundaries where unprompted side-effects outside the required scope are a risk.
5. **For `adversarial`**, trigger on >=50 changed code lines, sensitive domains (auth/payments/mutations/external APIs), OR any silent-pass verification mechanism (CI/CD gate, build/deploy script, mock/harness) regardless of diff size.
6. **For each stack-specific conditional persona**, use file types and changed patterns as a starting point, then decide whether the diff actually introduces meaningful work for that reviewer.
7. **For `data-migration`**, spawn only when the diff includes migration or schema artifacts (`db/migrate/*`, `db/schema.rb`, `db/structure.sql`, Alembic/Flyway/Liquibase paths, or explicit backfill scripts). Do **not** spawn for model-only or query-only changes without those files.
8. **For CE conditional prompt assets**, spawn `deployment-verification-agent` when the migration-artifact gate applies and the change is risky.
9. **Announce the team** before spawning with a one-line justification per conditional reviewer selected.
