# Guidelines

Shared standards for code review and documentation in the cortex repository.

The Auto Review system (`scripts/auto-review/`) references each file directly
from its sub-agents.

## File list

| File | Scope | Auto Review agent |
|---------|------|-------------------|
| [architecture.md](./architecture.md) | App-type patterns, shared packages, data architecture | `arch-reviewer` |
| [graph-integrity.md](./graph-integrity.md) | Product Graph integrity (`@graph-*` tags, documentation consistency) | `graph-reviewer` |
| [security.md](./security.md) | Security (authentication, input validation, secrets) | `security-reviewer` |
| [gcp-sdk-usage.md](./gcp-sdk-usage.md) | GCP SDK usage restrictions (Cloud Run + OTel ban on Storage / Secret Manager SDKs) | `security-reviewer` |
| [testing.md](./testing.md) | Test quality, code quality, naming conventions | `test-reviewer` |
| [observability.md](./observability.md) | Log and notification message authoring (no truncation, split delivery) | `test-reviewer` |
| [ai-antipattern.md](./ai-antipattern.md) | Anti-patterns specific to AI-generated code (hallucinated APIs, fallback abuse, dead code, scope creep, etc.) | `ai-antipattern-reviewer` |
| [document-writing.md](./document-writing.md) | Documentation style and maintenance policy | `doc-reviewer` |
| [impact-analysis.md](./impact-analysis.md) | Impact analysis (catching missed changes via Product Graph) | `impact-reviewer` |
| [recurrence-prevention.md](./recurrence-prevention.md) | Decision matrix for incident-issue fixes (lintify / horizontal rollout / add guideline / do nothing) | orchestrator |
| [severity.md](./severity.md) | Severity classification (Critical/Major/Minor/Nit) and criteria | orchestrator |
| [external-api-clients.md](./external-api-clients.md) | Policy for shared external API client packages (OpenAPI spec driven) | - |
| [lint-rules.md](./lint-rules.md) | Custom ESLint rule authoring and operations (no inline `no-restricted-syntax`) | - |
| [internal-member-identifier.md](./internal-member-identifier.md) | Shared rules when an internal member is used as an identifier (accept email \| nickname, resolve against Google Workspace) | - |
| [cloud-run-deploy.md](./cloud-run-deploy.md) | Known traps in Cloud Run deploys (`.gcloudignore` re-includes, direct dependency declaration for `pnpm deploy --legacy`, missing OTel exporter env injection) | - |
| [package-publish.md](./package-publish.md) | Shared workflow for publishing `packages/*` publicly to GitHub Packages (`@air-closet`), package.json conventions, automatic publish on main merge | - |
| [frontend.md](./frontend.md) | UI/UX and layout standards for the `apps/web/` frontends | - |
