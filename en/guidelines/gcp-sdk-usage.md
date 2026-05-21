# GCP SDK Usage Guidelines

On Cloud Run + Node 24 + OTel, several GCP SDKs hit known issues with `gaxios`, multipart upload, and checksum verification. In review, direct SDK usage is treated as Critical; the canonical pattern is to obtain an OAuth token from the metadata server and call the REST API directly via `fetch`.

## Prohibited

- New imports of `@google-cloud/storage`. GCS uploads bleed multipart boundaries, hit CRC32C / MD5 checksum mismatches, and trigger `gaxios` `URL is required` errors.
- New imports of `@google-cloud/secret-manager`. The auth path is unstable on Cloud Run + OTel, and calling the Secret Manager REST API directly is the canonical, proven pattern.
- New imports of `@google-cloud/pubsub`. Under Cloud Run + Node 24 + OTel, gRPC credential acquisition is broken, and publish fails for every record with `7 PERMISSION_DENIED: Method doesn't allow unregistered callers`. This was hit in real production at channel-talk-pipeline (4 runs / 189 chunks across 5/14–5/17 were stuck in `publish_failed`, halting Tier 2 for 7 months); the fix was migrating to REST. When Pub/Sub publish is needed under Cloud Run + OTel, hit `pubsub.googleapis.com/v1/projects/{p}/topics/{t}:publish` directly via `fetch`.

Violations are Critical. If you find an existing legacy import, do not add more in new or modified code — migrate to the REST implementation whenever you touch the file.

## Canonical Pattern

1. Fetch an OAuth token from `http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token` with the `Metadata-Flavor: Google` header.
2. Call the target service's REST API directly via `fetch`.
3. For GCS upload, use `POST https://storage.googleapis.com/upload/storage/v1/b/{bucket}/o?uploadType=media&name={encodedPath}`.
4. Pass GCS custom metadata via `x-goog-meta-{key}` headers.
5. For Secret Manager, call `https://secretmanager.googleapis.com/v1/projects/{project}/secrets/...` directly.

## Allowed

- `@google-cloud/bigquery` has a track record in cortex's Cloud Run / batch implementations and is not on the prohibited list.

## Reference Implementations

- `apps/pipeline/jobcan/src/gcs.ts`: Jobcan CSV upload. The canonical implementation for avoiding checksum mismatches.
- `apps/pipeline/shopify/src/gcs.ts`: GCS JSON API upload that sidesteps the `gaxios` issues on Cloud Run Node 24.
- `apps/pipeline/backlog/src/gcs.ts`: Earlier example of GCS JSON API upload and SDK checksum issues.
- `apps/pipeline/meet/backfill-gcs.ts`: Recording-file upload that avoids multipart boundary contamination.
- `apps/pipeline/db-account/src/secret-manager.ts`: Direct Secret Manager REST API calls.
- `apps/pipeline/channel-talk/src/pubsub-rest.ts` (`createRestPublisher`): The canonical Pub/Sub REST publish implementation. Uses a metadata-server token and hits `pubsub.googleapis.com/v1/projects/{p}/topics/{t}:publish` directly via `fetch`.

## Mechanical Checks

The shared `no-restricted-imports` config in `packages/core-ecosystem-config/oxlint/index.ts` forbids new imports of `@google-cloud/storage` / `@google-cloud/secret-manager` / `@google-cloud/pubsub`. Existing legacy imports are confined behind explicit overrides; do not add to the override list — migrate the call site to a REST implementation instead.

## BigQuery: Don't use `CAST(... AS STRING)` in SQL that re-inserts a TIMESTAMP column

### Problem

In BigQuery, `CAST(ts AS STRING)` on a TIMESTAMP column returns the **shortened `+00` timezone offset** form: `2023-10-29 06:24:29.237+00`. If you write this string back into another table's TIMESTAMP column via streaming insert (`tabledata.insertAll`), the server-side parser rejects it with:

```text
Could not parse '2023-10-29 06:24:29.237+00' as a timestamp.
Required format is YYYY-MM-DD HH:MM[:SS[.SSSSSS]]
```

The streaming insert TIMESTAMP parser accepts the full `+HH:MM` form, `Z`, no suffix, `UTC`-suffixed, and epoch float values — but **rejects the shortened `+00` form specifically** (confirmed empirically against `sandbox_logs`).

### Canonical Pattern

When an ETL intermediate SELECT stringifies a TIMESTAMP, passes it through JS, and later inserts it into a TIMESTAMP column, always emit RFC3339 (`T...Z`).

```sql
-- BAD: streaming insert rejects "+00"
SELECT CAST(src.created_at AS STRING) AS created_at FROM ...

-- OK: emit RFC3339 directly
SELECT FORMAT_TIMESTAMP('%FT%H:%M:%E*SZ', src.created_at, 'UTC') AS created_at FROM ...
```

`%E*S` gives sub-second precision without trailing zeros (works fine even with microsecond-precision sources), and the `Z` literal makes UTC explicit.

### Test fixture pitfall

When mocking `bigquery.query()` in unit tests, it is tempting to write fixtures as `'2026-04-10T00:00:00.000Z'` (ISO 8601) — but this format differs from the production `CAST(... AS STRING)` output (`2023-10-29 06:24:29.237+00`). **Because the fixture matches the happy path, the insert reject that only happens in production goes undetected**. Add a regression test that asserts on the SELECT statement itself (e.g. `expect(query).toContain("FORMAT_TIMESTAMP('%FT%H:%M:%E*SZ'")`).

### Reference Implementation

- `apps/pipeline/channel-talk-analyzer/src/etl.ts`: messages_raw / user_chats_raw → masked / safe table ETL. Uses `FORMAT_TIMESTAMP RFC3339`.

## BigQuery: MERGE on Policy Tag-protected JOIN keys requires fine-grained reader on the writer SA too

### Problem

In BigQuery, `MERGE INTO target USING source ON target.<col> = source.<col>` **treats the target-side column in the JOIN predicate as a read**. If that column is protected by a Column-level Policy Tag (`pii_high` / `pii_medium`, etc.), even an SA that looks like it only writes will be rejected without `roles/datacatalog.categoryFineGrainedReader` granted on the Policy Tag resource itself:

```text
Access Denied: BigQuery: User has neither fine-grained reader nor masked get permission
to get data protected by policy tag "pii-classification-v1 : pii_medium"
on column <project>.<dataset>.<table>.<col>.
```

`WHEN MATCHED THEN UPDATE SET col = source.col` and `WHEN NOT MATCHED THEN INSERT (...)` do not read the target column, so fine-grained reader is not required for Policy Tag columns outside the JOIN — letting you keep least privilege.

### Canonical Pattern

For each SA executing the MERGE, grant `roles/datacatalog.categoryFineGrainedReader` **directly on the Policy Tag resource** for every Policy Tag-protected column used as a JOIN key (project-level grants are forbidden).

```typescript
// infra/core/channel-talk-iam.ts
new gcp.datacatalog.PolicyTagIamMember(`${prefix}-channel-talk-worker-pii-medium-reader`, {
  policyTag: piiMediumPolicyTag.name,
  role: 'roles/datacatalog.categoryFineGrainedReader',
  member: pulumi.interpolate`serviceAccount:${workerSa.email}`,
});
```

Do not grant on Policy Tag columns outside the JOIN (e.g. `email` / `name` under `pii_high`). When you add a new Policy Tag or change a MERGE's JOIN keys, review the writer SA's grants in the same pass.

### Review Checks

When a new or modified SQL contains `MERGE`, verify the following at review time:

1. Whether the target column in the `ON` predicate is Policy Tag-protected (check `infra/core/data-catalog.ts` and dataset definitions).
2. If so, whether the writer SA has fine-grained reader granted via `PolicyTagIamMember` (check `infra/core/<domain>-iam.ts`).
3. That Policy Tag columns outside the JOIN are **not** granted (least privilege).

### Reference Implementation

- `infra/core/channel-talk-iam.ts`: grants `pii_high` / `pii_medium` to the analyzer SA, but only `pii_medium` to the worker SA. The worker holds `pii_medium` solely to satisfy the `users_raw.user_id` read path of the `mergeUsers` JOIN target column.

## BigQuery: Use `bqTimestampParam()` for TIMESTAMP / DATE / DATETIME in named query parameters

### Problem

When you pass a named parameter to `@google-cloud/bigquery`'s `query()` / `createQueryJob()` as a raw ISO 8601 string in `params` together with the temporal type in `types`, the client-side serializer sets the wire-format `parameterValue.value` to **`undefined`**. BigQuery interprets this as **NULL** and rejects `INSERT INTO ... <REQUIRED column>` with:

```text
Required field <column> cannot be null
```

```typescript
// BAD: parameterValue.value becomes undefined and BQ treats it as NULL
await client.query({
  query: `INSERT INTO t (ts) SELECT TIMESTAMP(@ts)`,
  params: { ts: '2026-05-13T18:43:36.544Z' },
  types: { ts: 'TIMESTAMP' },
});
```

`BigQuery.valueToQueryParameter_('2026-05-13T18:43:36.544Z', 'TIMESTAMP')` returns
`{ parameterType: { type: 'TIMESTAMP' }, parameterValue: { value: undefined } }`.

### Canonical Pattern

Convert temporal parameters into a `BigQueryTimestamp` instance via `@cortex/bigquery`'s `bqTimestampParam()`. `BigQueryTimestamp` is self-describing, so neither the explicit `types` declaration nor the `TIMESTAMP(@ts)` cast on the SQL side is needed.

```typescript
import { bqTimestampParam } from '@cortex/bigquery';

// OK: parameterValue.value holds the ISO string, and parameterType is self-described as TIMESTAMP
await client.query({
  query: `INSERT INTO t (ts) SELECT @ts`,
  params: { ts: bqTimestampParam('2026-05-13T18:43:36.544Z') },
});
```

### Automated checks

- The ESLint rule `graph/no-bq-string-timestamp-param` flags explicit `TIMESTAMP` / `DATE` / `DATETIME` / `TIME` entries in the `types` field of any object literal that also carries a `query` key (currently `warn` during the migration of existing violations, to be promoted to `error` once cleared).
- Mock-only unit tests cannot detect the wire-format dropout. Pin serializer-level regressions in `@cortex/bigquery`'s contract tests (which assert on the return value of `BigQuery.valueToQueryParameter_`).

### Reference Implementation

- `packages/infra/shared/bigquery/src/index.ts`: the `bqTimestampParam()` implementation and its wire-format contract test.
- `apps/pipeline/channel-talk/src/analyzer-enqueue.ts`: uses `bqTimestampParam()` in `insertAnalyzerRun` / `insertChunkItems`.

## BigQuery: Don't double-wrap `SUM()` in ORDER BY when the SELECT alias shares the column name

### Problem

In patterns like `SELECT SUM(col) AS col ... GROUP BY ... ORDER BY SUM(col)`, where the SELECT aliases the aggregate back to the original column name, writing `SUM(col)` again in ORDER BY can cause BigQuery to interpret it as an aggregation of the alias (i.e. `SUM(SUM(col))`), producing a nested-aggregate error.

```sql
-- BAD: SUM(impressions) in ORDER BY can wrap the SELECT alias and become a nested aggregate
SELECT
  appeal_axis,
  SUM(impressions) AS impressions
FROM t
GROUP BY appeal_axis
ORDER BY SUM(impressions) DESC

-- BAD: same problem in a compound expression
SELECT
  SUM(input_tokens) AS input_tokens,
  SUM(output_tokens) AS output_tokens
FROM t
GROUP BY nickname_key
ORDER BY (SUM(input_tokens) + SUM(output_tokens)) DESC
```

### Canonical Pattern

Reference the alias directly in ORDER BY (do not wrap with `SUM()` again).

```sql
-- OK: reference the single alias directly
ORDER BY impressions DESC

-- OK: compound expressions reference the aliases too
ORDER BY (input_tokens + output_tokens + cache_creation_input_tokens + cache_read_input_tokens) DESC
```

### Reference Implementation

- `apps/api/product/src/features/cc-usage/infrastructure/bigquery-cc-usage-repository.ts`: example fix in `getUserBreakdown` / `getRepositoryBreakdown`.

## BigQuery: `createQueryJob` must wait for completion via `getQueryResults()` / `promise()`

### Problem

`@google-cloud/bigquery`'s `BigQuery#createQueryJob()` **only waits until the job resource is created**. The returned `Job` instance is in a state that precedes the actual query / DML execution on the BQ side.

```ts
// BAD: waits for job creation, but not for DML execution
const [job] = await client.createQueryJob({ query: 'INSERT INTO t SELECT ...' });
const [metadata] = await job.getMetadata(); // numDmlAffectedRows may be undefined
return Number(metadata.statistics?.query?.numDmlAffectedRows ?? 0); // treated as 0
```

- A **read against the same table** immediately afterwards does not see the write — read-after-write is missed, returning an empty snapshot.
- `getMetadata()` is just a snapshot fetch; it does not synchronize with job completion.
- For the seconds-to-tens-of-seconds window before BQ finishes executing, downstream processing proceeds destructively.

This trap caused an incident in `apps/pipeline/channel-talk/src/analyzer-enqueue.ts`: chunks_planned=0, zero Pub/Sub publishes (the analyzer never started).

For comparison, `BigQuery#query()` is a convenience method that internally calls `createQueryJob` → waits for completion → returns rows. That is why every other query appeared to synchronize on completion.

### Canonical Pattern

For any `Job` obtained from `createQueryJob`, **always `await` `getQueryResults()` or `promise()`**. Even when you only want metadata such as `numDmlAffectedRows`, call `getMetadata()` after the completion wait.

```ts
const [job] = await client.createQueryJob({ query: 'INSERT INTO t SELECT ...' });
await job.getQueryResults(); // wait for DML completion
const [metadata] = await job.getMetadata();
return Number(metadata.statistics?.query?.numDmlAffectedRows ?? 0);
```

`getQueryResults()` returns rows, but for DML / DDL it returns an empty array. We only care about the completion side effect, so the return value can be discarded.

`job.promise()` is a thin wrapper that internally calls `getQueryResults()`. The two are semantically equivalent.

### Automated checks

- The ESLint rule `graph/no-bq-create-query-job-without-await-completion` uses the ESLint scope manager to verify that, for the `Job` binding returned by `createQueryJob`, a member access of `getQueryResults` or `promise` exists somewhere in scope. Wrapper patterns that `return` the binding are allowed.
- Mock-only unit tests cannot reproduce the async behavior of real BQ. Add a regression test that asserts invocation ordering on `vi.fn` calls — specifically that the next SELECT is not issued before `getQueryResults` resolves. See `apps/pipeline/channel-talk/src/analyzer-enqueue.test.ts`: "issues insertChunks SELECT only after insertChunkItems DML completion (getQueryResults)".

### Reference Implementation

- `apps/pipeline/channel-talk/src/analyzer-enqueue.ts` (`insertChunkItems`): the canonical `createQueryJob` → `getQueryResults` → `getMetadata` pattern.
- `apps/pipeline/channel-talk-analyzer/src/merge.ts`: waiting for MERGE statement completion.
- `packages/eslint-plugin-graph/src/rules/no-bq-create-query-job-without-await-completion.ts`: the lint rule itself.
