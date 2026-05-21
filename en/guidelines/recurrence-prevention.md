# Recurrence Prevention Guideline

How to decide "we will not let the same bug happen twice" when fixing an incident or defect. Do not let a bug fix end as a **point**; expand it into an **area**.

## Philosophy

The point of a bug fix is not "make the symptom go away" but "close the class of recurrences." If we fall into the same trap twice, treat it as a team-systems problem, not a code-quality one.

Mirroring the priority in [document-writing.md](./document-writing.md) (types > lint > tests > prose), **mechanize what can be mechanized. Do not try to enforce discipline through text.**

"Just leave a memo," "leave a TODO comment," or "handle it later" is prohibited. These contradict the "documentation value depends on freshness" principle in [document-writing.md](./document-writing.md) and inevitably go stale.

## Prohibit "We'll Get to It Later"

Complete the recurrence-prevention action **in this PR**. Defer-flavored phrasing like the following is prohibited.

- "Consider linting in the future," "Strengthen later," "Promote to error later"
- "Look at it during the next refactor," "Address it in the next phase," "Handle in a separate PR"
- "Introduce as `warn` because existing violations remain, then promote to `error` later"
- Leaving `TODO: ...` or `FIXME: ...` in code

If it can be addressed, **address it in this PR**. If you add a lint rule, also drive existing violations to zero in the same PR and ship it at `error`. If existing violations are so plentiful that root-cause work is impractical, do not introduce the lint rule in that PR — ship it alongside the root-cause fix instead (and if you must split, do root-cause fix first, then rule introduction at `error` from the start).

The rollout technique of "introduce as `warn` because existing violations remain" is **rejected**. It's effectively a deferral, and the responsibility to promote `warn` to `error` ends up floating and decaying. If you cannot introduce at `error`, redesign the PR scope to include the root-cause fix.

## Decision Matrix

For a bug-fix PR, walk this matrix in order and execute the applicable action **within the same PR**.

| Situation | Required action | Form |
|------|---------------|------|
| Same trap hit two or more times | **Lint rule required** (custom ESLint rule / type constraint / CI guard) | Mechanization |
| Pattern may exist elsewhere | **Horizontal expansion required** (sweep similar nodes via Product Graph; fix discovered occurrences in the same PR) | Investigation + fix |
| Mechanical verification is impossible but the principle is valuable | **Add an item to an existing guideline** (creating a new file is a last resort) | Guideline |
| One-off; no principle to extract | **Do nothing** (just the bug fix; rediscover via git log / blame if needed) | — |

## Per-Action Criteria

### When a Lint Rule Is Required

Linting is required if any of the following applies:

- **The same pattern has occurred twice or more** (verified via git log / blame)
- **The trap fires frequently when adding new files** (naming conventions, import constraints, missing infrastructure declarations, etc.)
- **An AST-level detectable prohibited pattern**

Implementation procedure follows [lint-rules.md](./lint-rules.md). Direct use of `no-restricted-syntax` / `no-restricted-imports` is forbidden; implement as a custom rule in `@cortex/eslint-plugin-graph`.

### When Horizontal Expansion Is Required

If any of the following applies, sweep and fix other locations within the same PR:

- **Multiple functions or configurations follow a common pattern (naming, imports, initialization)**
- **Multiple stacks reference the same data source, API, or table**
- **They share the same Pub/Sub topic or Cloud Scheduler job**

Start sweeps from the Product Graph MCP:

```text
search_product_graph_nodes(query: "<characteristic of the trap>", search_mode: "semantic")
trace_product_graph_connections(start_node: "<the relevant node>", direction: "both")
```

See [impact-analysis.md](./impact-analysis.md) for details.

### When to Add an Item to a Guideline

Cases where mechanical verification is impossible but the principle will influence future decisions:

- Design decisions (DDD layer placement, shared-package extraction criteria, etc.) → [architecture.md](./architecture.md)
- Security principles (auth boundaries, secret handling, etc.) → [security.md](./security.md)
- Test quality (matcher choice, flaky elimination, etc.) → [testing.md](./testing.md)
- Documentation operations → [document-writing.md](./document-writing.md)

**A new guideline file is the last resort.** Try to fold into an existing guideline first. A proliferation of new files breaks cross-referencing.

### When to Choose "Do Nothing"

If **all** of the following apply, finish with just the bug fix:

- A single occurrence with low recurrence likelihood
- Mechanization is technically infeasible (external API rate-limit volatility, third-party dependencies, etc.)
- Cannot be folded into an existing guideline (too narrow or too unique to generalize)

Choose explicitly not to document. "Recording it just in case" is the start of staleness.

## Applied Examples

### Example 1: Forbidding New Imports of `@google-cloud/storage` / `@google-cloud/secret-manager`

**Trap**: On Cloud Run + Node 24 + OTel, GCP SDKs interfere with trace and metrics export. GCS / Secret Manager must obtain an OAuth token from the metadata server and call the REST API directly via `fetch`.

**Verdict**: Multiple apps have hit the same trap historically, and the mistake materializes in a single import line → **lint required**

**Action**: Forbidden in the shared `no-restricted-imports` config in `packages/core-ecosystem-config/oxlint/index.ts`. See [gcp-sdk-usage.md](./gcp-sdk-usage.md) for details. When adding new forbidden targets, follow [lint-rules.md](./lint-rules.md).

### Example 2: Missing `.gcloudignore` Re-include

**Trap**: When adding an app that uses `tsx scripts/*.ts` as a Cloud Run Job entrypoint, omitting the `!apps/<path>/scripts/` re-include in `.gcloudignore` causes the script to be missing after deploy, killing the job immediately. Same pattern hit four times.

**Verdict**: Four recurrences + mechanically verifiable (`.gcloudignore` parsing + cross-check against Cloud Run Job Pulumi definitions) → **lint or CI-script required**

**Action**: `scripts/check-gcloudignore-consistency.ts` is implemented as a CI guard (run by the guards job in `.github/workflows/test.yml`). It automatically detects apps that need an `apps/<path>/scripts/` re-include based on src and Dockerfile content, and fails CI if any are missing from `.gcloudignore`. See pattern 1 in [cloud-run-deploy.md](./cloud-run-deploy.md) for details.

### Example 3: Missing OTel Exporter Env Injection

**Trap**: Forgetting to inject `OTEL_EXPORTER_OTLP_ENDPOINT` and `GRAFANA_CLOUD_API_KEY` via `secretKeyRef` into the envs of a new Cloud Run Service / Job Pulumi definition causes `@cortex/otel` to skip init. Traces / logs / metrics never reach Grafana, **making incidents undetectable**.

**Verdict**: Mechanically verifiable (Pulumi resource analysis) → **lint already in place**

**Action**: `scripts/check-otel-env-injection.ts` is implemented as a CI guard (run by the guards job in `.github/workflows/test.yml`). When `infra/**/*.ts` constructs a `gcp.cloudrunv2.Service` / `gcp.cloudrunv2.Job` and the referenced app's top-level entrypoint calls `initOtel()`, it requires declarations of `OTEL_EXPORTER_OTLP_ENDPOINT` / `GRAFANA_CLOUD_API_KEY`. See pattern 3 in [cloud-run-deploy.md](./cloud-run-deploy.md) for details.

Existing violations are explicitly recorded in `ALLOWLIST` and resolved incrementally via per-stack Pulumi PRs (new violations are blocked by CI).

### Example 4: BigQuery Named TIMESTAMP Parameter Binding Mistake

**Trap**: Passing a raw string to `query()` / `createQueryJob()` as `params: { ts: isoString }` + `types: { ts: 'TIMESTAMP' }` causes the `@google-cloud/bigquery` serializer to set the wire-format `parameterValue.value` to `undefined`, which BigQuery interprets as NULL. `INSERT INTO ... <REQUIRED column>` then rejects with `Required field ... cannot be null`, and channel-talk's Tier2 ETL halted in production. Unit tests that fully mocked `bigquery.query()` could not detect this — SQL and parameter binding were never exercised (this "mock-fixture pitfall" class is already documented in [gcp-sdk-usage.md](./gcp-sdk-usage.md)).

**Verdict**: AST-level detectable prohibited pattern (a temporal type string in `types`) + foldable into an existing guideline → **lint + guideline addition**

**Action**: Introduced the ESLint rule `graph/no-bq-string-timestamp-param` (detects explicit `TIMESTAMP` / `DATE` / `DATETIME` / `TIME` entries in `types`, at `error` level). Added the canonical pattern of passing values as `BigQueryTimestamp` instances via the shared helper `bqTimestampParam()` from `@cortex/bigquery` to [gcp-sdk-usage.md](./gcp-sdk-usage.md). Regression at the serializer level is pinned by a contract test in `@cortex/bigquery`. Existing violations including channel-talk / github-activity-core / member-daily-report-core / basic-design-archive / db-account / meet were all migrated to `bqTimestampParam()` in the same PR.

### Example 5: Silent Swallowing in try/catch and Promise#catch

**Trap**: Swallowing patterns like `catch (e) {}` / `.catch(() => null)` / `.catch(() => [])` are common in AI-generated code. Errors cannot be observed and execution continues, so production incidents proceed in a "neither count nor cause is known" state. "Type-name-based downgrades" — e.g. downgrading a `NotFoundError` to warn based on the type name alone — produce the same problem.

**Verdict**: AST-level detectable (analysis of `catch`-body content / `.catch` handler bodies) + foldable into a guideline → **lint + guideline addition**

**Action**: Introduced the ESLint rule `graph/no-silent-catch` (at `error` level). It flags empty catch blocks and literal-fallback / empty-body handlers in Promise#catch, and forces either a structured log via `@cortex/otel/logger` or a rethrow. The criteria for log levels (fatal / error / warn / info / debug) are centralized in the "How to Decide Log Level" section of [observability.md](./observability.md). At introduction, all 39 existing violations were root-cause fixed in the same PR (rethrow / `serializeError` structured log / ENOENT / SyntaxError discriminator, etc.).

### Example 6: Missing Setup Steps When Adding a New db-account-pipeline Target

**Trap**: When adding a new DB target to db-account-pipeline, two **kinds of missing values** rely on a manual checklist and cannot be statically detected in code, making them easy to leave undone.

1. **Bastion SSH private-key Secret with no version**: `BASTION_KEY_SECRETS` in `infra/pipeline/db-account/index.ts` declares Secret names, but the `createSecret()` helper does not take a value (PEM) argument by design, so Pulumi creates empty Secret shells. Forgetting to run `gcloud secrets versions add` manually still lets the pipeline boot and continue processing, but **the moment a standard (non-AI-only) request with an SSH public key arrives**, `getSecret(...adminKeySecret)` immediately before `[DB] Lambda executor` returns 404 with `Secret ... not found or has no versions`, partially failing that DB. During periods when only AI-only requests arrive, the issue stays fully latent, and the `db_account_processed` table remains `completed`, so anomaly detection cannot fire (real case: ecosale-prod went unnoticed for over seven weeks from target addition on 2026-03-31 to the first standard request on 2026-05-18).
2. **Missing tabs in the management spreadsheet**: When the `{prefix}_view` / `{prefix}_edit` / `{prefix}_delete` / `{prefix}_stg` tabs corresponding to `(spreadsheetId, sheetNamePrefix)` in `db-configs.ts` are absent in the existing spreadsheet, processing emits a non-critical WARN `[Processor] Spreadsheet append failed: Unable to parse range` and returns `completed`. The applicant still receives credentials via Firestore / Google Docs / Slack DM, so **nobody notices**, but it becomes impossible to track who holds which permission on the management sheet.

**Verdict**: Not detectable at the AST level. External verification via Pulumi state / Sheets API queries is technically possible, but integrating it into routine CI would require auth for the GCP state backend and Sheets API credentials injected into CI — too much infrastructural coupling — so **CI-guard is deliberately not adopted**. With one occurrence of (1) (ecosale-prod) and four simultaneous occurrences of (2) (ecosale_{view,edit,delete} + phoenix_styling prod tabs) = **horizontal-expansion target**. Address by **integrating into an interactive script + codifying in the guideline** the items that currently rely on a manual checklist when adding a new DB.

**Action**:

- Extend `scripts/add-db-account-target.ts` to inject the bastion SSH key for a new VPC group interactively (accept a PEM file path and run `gcloud secrets versions add` + IAM binding to the Pipeline SA). If the value is skipped, surface a warning log with the `gcloud secrets versions add` command so it can be injected before the pipeline runs.
- Add **"Create tabs in the management spreadsheet"** as a separate item to the script's trailing manual checklist. Spell out the required tab names per prod/stg, the procedure to copy header rows from existing tabs, and the latent-behavior consequence when not created.
- Update the "Adding a New Database" operational procedure in `docs/pipeline/db-account.md` to make both items mandatory, and record as an `IMPORTANT` callout that the issue stays latent under AI-only traffic.

### Example 7: Stale Workflow `--filter` Reference After Workspace Package Deletion

**Trap**: When a PR deletes a workspace package in the monorepo, forgetting to remove references from `.github/workflows/*.yml` filters such as `pnpm exec turbo run build --filter='@cortex/<pkg>'` causes the deploy / build of stacks that run those filters to fail with `x No package found with name '@cortex/<pkg>' in workspace`. Stacks that are not built routinely (e.g. the aws stack's Lambda build) lie dormant until the relevant stack is triggered next. In `feat(line-user-id-resolver): Phase 5 — drop dedicated Lambda` (#974), `@cortex/line-user-id-resolver-executor` was deleted, but the filter in `deploy-stack.yml` was left in place, and the issue first surfaced during an aws stack deploy (PR #1006).

**Verdict**: Mechanically verifiable (parse `--filter` tokens in workflow YAML + cross-check against `pnpm list -r --json`) → **CI script required**

**Action**: `scripts/check-workflow-filter-references.ts` is implemented as a CI guard (run by the guards job in `.github/workflows/test.yml`). It extracts `--filter[ =]['"]?@cortex/<name>['"]?` (including turbo's `...` / `^` / `[ref]` qualifiers) from `.github/workflows/*.{yml,yaml}` and uses `pnpm list -r --depth=-1 --json` to detect anything that doesn't actually exist in the workspace. New violations fail CI at PR time.

## Review Checks

When reviewing a bug-fix PR:

- [ ] The PR description states the recurrence-prevention action chosen (lint / horizontal expansion / guideline addition / nothing)
- [ ] When "do nothing" is chosen, the reviewer has confirmed all the conditions above are met
- [ ] If horizontal expansion was taken, the PR description records the Product Graph sweep scope and the discovered occurrences
- [ ] Any added lint rule or guideline item does not duplicate an existing one

Incident fixes that fail to meet these standards are flagged at Major or higher for insufficient recurrence prevention. When the recurrence risk could lead to data destruction, broken authentication, or a halted production job, raise it to Critical.
