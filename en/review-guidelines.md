# cortex Code Review Guidelines

This document is the entry point for code reviews in cortex. Detailed criteria are split across the files under [guidelines/](./guidelines/), and the Auto Review agents reference each file directly.

## How to Review

1. Use Product Graph MCP to understand the feature, its dependencies, and the scope of impact for the change.
2. Read the relevant detailed guideline based on the type of files changed.
3. Classify each finding by severity using [guidelines/severity.md](./guidelines/severity.md).
4. For Critical / Major / Minor findings, cite the specific reason for the fix and the affected lines. Nit findings are optional.

## Entry-Point Checklist

| Area | Detail | Primary Scope |
|------|--------|---------------|
| Architecture | [guidelines/architecture.md](./guidelines/architecture.md) | `apps/`, `packages/`, dependency direction, shared packages |
| Product Graph | [guidelines/graph-integrity.md](./guidelines/graph-integrity.md) | `@graph-*` tags, document consistency, Product Graph exceptions |
| Security | [guidelines/security.md](./guidelines/security.md) | Authentication, input validation, secrets, external APIs |
| GCP SDK Usage | [guidelines/gcp-sdk-usage.md](./guidelines/gcp-sdk-usage.md) | Cloud Run, GCS, Secret Manager, OTel |
| Testing & Quality | [guidelines/testing.md](./guidelines/testing.md) | Test coverage, edge cases, naming, code quality |
| Observability & Notifications | [guidelines/observability.md](./guidelines/observability.md) | Never truncate logs, Slack notifications, or alert bodies; split into multiple messages instead |
| AI Anti-Patterns | [guidelines/ai-antipattern.md](./guidelines/ai-antipattern.md) | Hallucinated APIs, fallback abuse, dead code, scope creep, unnecessary backward compatibility |
| Documentation | [guidelines/document-writing.md](./guidelines/document-writing.md) | Placement of docs, avoiding duplication, alignment with generated artifacts |
| Impact Analysis | [guidelines/impact-analysis.md](./guidelines/impact-analysis.md) | Missed edits, callers, downstream effects on DB / API / UI |
| Recurrence Prevention | [guidelines/recurrence-prevention.md](./guidelines/recurrence-prevention.md) | Incident issues, lint enforcement, horizontal propagation, guideline additions |
| Severity | [guidelines/severity.md](./guidelines/severity.md) | Determining Critical / Major / Minor / Nit |

## Root-Cause Principle

Review findings must be **addressed at the root cause within the same PR**. Symptom-only patches, workarounds, or deferring the fix to a separate PR are not accepted as a general rule.

- **Root-cause fix required**: Architecture violations, design defects, spec deviations, and safety defects raised in review must be fixed at the source within the PR. Closing the thread with a shallow `if` branch or by swallowing the error is not sufficient. Any response that violates this rule must be returned with `REQUEST_CHANGES`.
- **No deferral**: Excuses such as "to be handled in a separate PR", "next session", "out of scope", or "we'll do it incrementally" are not accepted as reasons to leave a finding unaddressed or partially addressed. Reviewers must return `REQUEST_CHANGES` when an author tries to close a thread with these justifications.
- **No deferral via TODO / FIXME**: Outstanding review feedback must not be merged in the form of `TODO` / `FIXME` comments in the code. It is equally forbidden to verbally promise "we'll fix it later" in a comment.
- **No demotion**: Do not demote a finding to Nit on the basis of deferral or incremental delivery. See the "No demotion rule" in [guidelines/severity.md](./guidelines/severity.md) for details.

This document does not enumerate exception conditions as fixed categories. Exceptions are judged case by case, and any agreement reached must be made explicit on the PR.

## Highest-Priority Gates

### Composable Architecture

- Is `packages/` kept as reusable building blocks and `apps/` kept as services that compose them?
- Is there no direct `import` between apps, and is shareable logic placed in the appropriate package?
- Do new apps and packages follow the structure and naming conventions of the existing categories?

See [guidelines/architecture.md](./guidelines/architecture.md) for details.

### Product Graph Integrity

- Do new or modified declarations carry the required `@graph-*` tags?
- Are the connections between code, DB, docs, and infra traceable in the Product Graph?
- Are manual, duplicated lists of implementations being introduced into generated documents or indexes?

See [guidelines/graph-integrity.md](./guidelines/graph-integrity.md) and [guidelines/document-writing.md](./guidelines/document-writing.md) for details.

### Impact Analysis

- Have you traced the forward and backward dependencies of every changed node in the Product Graph?
- Are the standard missed-edit patterns (untransmitted type changes, `via:` field drift, one-sided `shares_topic` changes, unpropagated Repository → UseCase → Handler chains, untouched tests, `documented_by` drift) cleaned up within the PR?
- Are similar implementations surfaced by semantic search updated in the same PR?

See [guidelines/impact-analysis.md](./guidelines/impact-analysis.md) for details.

### Security and Boundaries

- Are user inputs, external API responses, environment variables, and Secret values validated at the boundary?
- Are required values prevented from being hidden by fallbacks, and do misconfigurations fail fast? Forbidden patterns include `baseUrl ?? ''`, making fields `?: string` optional, and implicit fallbacks to empty string, same-origin, or `null`. Required configuration must be `required` at the type level and pinned by a startup-time throw.
- Are authentication, authorization, and permission checks performed at the right layer rather than being delegated to callers?

See [guidelines/security.md](./guidelines/security.md) for details.

### GCP SDK Usage (Cloud Run)

- Does the change avoid newly importing `@google-cloud/storage` or `@google-cloud/secret-manager` on Cloud Run + Node 24 + OTel?
- For GCS / Secret Manager, does the code obtain an OAuth token from the metadata server and call the REST API directly via `fetch`?
- `@google-cloud/bigquery` is not forbidden, but when adding a new GCP SDK, has its track record in the codebase and its OTel impact been verified?

See [guidelines/gcp-sdk-usage.md](./guidelines/gcp-sdk-usage.md) for details.

### OTel Instrumentation (Cloud Run)

- Does the Pulumi definition (`infra/.../index.ts`) of a new Cloud Run Service / Job inject `OTEL_EXPORTER_OTLP_ENDPOINT` and `GRAFANA_CLOUD_API_KEY` into `envs` via `valueSource.secretKeyRef`?
- Is `roles/secretmanager.secretAccessor` bound to the corresponding SA via `gcp.secretmanager.SecretIamMember` for both secrets above?
- Does the application code (`apps/.../src/index.ts`) call `initOtel({ serviceName: '...' })`? When the env vars are missing, `@cortex/otel` silently skips init, so no trace / metrics / log reaches Grafana and exceptions are never emitted as exception events.

See pattern 3 of [guidelines/cloud-run-deploy.md](./guidelines/cloud-run-deploy.md) for details.

### Public URLs, DNS, and Edge Router

- For new or modified `*.air-closet.ai` services exposed publicly, is the entire delivery path defined and not only the DNS record?
- For Cloud Run services (API / MCP / Bot / Pipeline), are the subdomain definition in `infra/dns/index.ts` and the `cloudRunRoutes` KV route in `infra/dns/sandbox.ts` both in place?
- For Cloudflare Pages web apps, are the Pages project, custom domain, and deploy settings complete, and is there no accidental Edge Router KV registration?
- When the frontend and the API are deployed separately, can both the web URL and the API URL be externally verified (e.g. `200`, `401 + X-Edge-Auth-Start`, `/health`)?

See infra/edge-router.md _(cortex internal reference)_ for details.

### Observability and Notification Messages

- Are log lines, Slack notifications, and alert bodies free of `slice`, `…and N more`, or character-count truncation?
- For high-volume cases, is the design to split the output across multiple sends or to capture the full data in structured logs?
- When truncation is genuinely unavoidable, does the message state that truncation occurred and point to where the full data lives (Logs Explorer, GCS, BQ URL, etc.)?
- In `catch` blocks, is the code avoiding silent failures such as returning `null`, an empty array, `undefined`, or a static fallback without emitting an error log? Unless the error is re-thrown, structured logging through the `@cortex/otel/logger` Pino logger with `{ err: serializeError(error), event, ...context }` is required.
- Is the error level (fatal / error / warn) chosen based on what the error means for the feature (**recovery required**, **scope of impact**, **whether it is an expected business state**) rather than the error class name (e.g. `NotFoundError`)? See the "Log level criteria" section of [guidelines/observability.md](./guidelines/observability.md) for the detailed decision rules.

See [guidelines/observability.md](./guidelines/observability.md) for details.

### Tests and Verification

- Are tests for the changed behavior added first?
- Are config contracts, boundary checks, and failure modes pinned down by regression tests?
- Are the project-defined commands such as `pnpm test`, `pnpm build`, and `pnpm lint` being executed in preference to ad-hoc commands?

See [guidelines/testing.md](./guidelines/testing.md) for details.

### AI-Generated Code Anti-Patterns

- Does the code avoid calling hallucinated APIs, non-existent methods, or incorrect argument shapes?
- Is uncertainty being hidden behind `??` / `||` fallbacks on required data, default arguments, or `catch { return ''; }` patterns?
- After refactoring, are there leftover unused functions, unreachable branches, stale imports, or orphaned re-exports?
- Has unrequested functionality, premature abstraction, premature caching, or unnecessary legacy compatibility mapping crept in?
- Does the change deviate without justification from existing codebase patterns (shared clients, naming conventions, error-handling style)?
- In response to review findings, does the author claim to have "addressed" them by adding tests, adding documentation, or doing unrelated refactoring instead of fixing the root cause?

See [guidelines/ai-antipattern.md](./guidelines/ai-antipattern.md) for details.

### Recurrence Prevention

- For incident / bug-fix issues, does the PR description explicitly state which of the following was chosen: lint enforcement, horizontal propagation, an addition to an existing guideline, or "do nothing"?
- Are recurrence patterns that can be detected mechanically being left to human review only?
- Does the PR include "let's just record this somewhere" style documentation additions (collections of notes that will rot, or `TODO` comments)?

See [guidelines/recurrence-prevention.md](./guidelines/recurrence-prevention.md) for details.

### Protecting Quality Standards

- Does the change weaken any quality standard, such as guideline documents (under `docs/guidelines/`), lint rules (e.g. `packages/eslint-plugin-graph/`), or coverage thresholds (`vitest.config.ts`)?
- Such loosening changes **require approval from a human reviewer**. The AI reviewer must not approve a PR that contains them and must return `REQUEST_CHANGES`.
- Loosening on the grounds that "the existing rule is too strict" or "let's align with reality" is treated the same way. The merit of changing the standard itself is a human judgment.
- "Existing code already uses this pattern" or "the existing implementation already violates this, so let's align the standard to it" is not valid justification for loosening. Existing violations are a separate problem to fix; they are never grounds for lowering the standard.

See the "Quality-standard loosening" section in [guidelines/severity.md](./guidelines/severity.md) for details.

## Decision Flow

| Verdict | Condition | GitHub Action | Effect |
|---------|-----------|---------------|--------|
| Critical | Security, data loss, severe production incident, OOM risk, documentation drift, `@graph-*` JSDoc mismatch or absence, lowering of coverage thresholds | `REQUEST_CHANGES` | Merge must be blocked |
| Major | Spec deviation, architecture violation, missing tests, operational breakage, severe performance issue | `REQUEST_CHANGES` | Merge blocked by default |
| Minor | Maintainability regressions, naming improvements, small refactors, local inconsistencies, latent bug risk | `REQUEST_CHANGES` | Must be resolved (cannot be ignored) |
| Nit | Style preferences, surface inconsistencies, trivial improvements, optional cleanup | `APPROVE` (raised as a comment) | Does not block |

For the canonical severity definitions, the no-demotion rule, and exception conditions, defer to [guidelines/severity.md](./guidelines/severity.md).

## Auto Review Operating Rules

- Do not leave positive-only comments. Focus on what needs to change.
- Each finding must include the target file, line number, problem, reason for the fix, and how to verify it.
- Do not duplicate findings with the same `family_tag`; check for related occurrences at the same time.
- Even when issuing an `APPROVE`, explicitly state any remaining risks or verification steps that were not performed.

## Related Documents

- DESIGN.md _(cortex internal reference)_ - Entry point for design direction and current state of the system
- VISION.md _(cortex internal reference)_ - Product vision
- [docs/guidelines/README.md](./guidelines/README.md) - Index of split review and documentation standards
- [docs/guidelines/document-writing.md](./guidelines/document-writing.md) - Documentation authoring and management policy
- docs/infra/README.md _(cortex internal reference)_ - Infrastructure layout and operations
- docs/product-graph/README.md _(cortex internal reference)_ - Product Graph
- docs/code-graph/README.md _(cortex internal reference)_ - Code Graph
