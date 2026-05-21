# Cloud Run Deploy Guidelines

Known traps in Cloud Run Job / Service deploy configuration, and the correct response for each.

## Pattern 1: Use a root anchor for the `scripts/` exclusion in `.gcloudignore`

### Problem

If you exclude `scripts/` unanchored (`scripts/`), a known BuildKit / gcloudignore behavior sweeps `apps/<x>/scripts/` into the exclusion as well, and any subsequent `!apps/<x>/scripts/` re-include has no effect. The result: a Cloud Run Job whose entrypoint is `tsx apps/<x>/scripts/*.ts` dies on container startup with `ERR_MODULE_NOT_FOUND: Cannot find module ...scripts/*.ts`.

Real example: biz-graph marketing-loader failed to pick up `apps/mcp/biz-graph-server/scripts/load-marketing-job.ts` because of an unanchored `scripts/` in `.gcloudignore`, and was failing to start for a full month from 4/16 onward (a `!apps/mcp/biz-graph-server/scripts/` line was present but had no effect).

### Resolution

Write the `scripts/` exclusion in both `.gcloudignore` and `.dockerignore` with a **root anchor: `/scripts/`**. This excludes only the `scripts/` directory at the repo root; `apps/<x>/scripts/` is automatically included, so no `!apps/<x>/scripts/` re-include is needed.

```text
/scripts/
```

### Checklist

When adding a new Cloud Run Job:

- [ ] The `scripts/` exclusion in `.gcloudignore` / `.dockerignore` is `/scripts/` (root-anchored)
- [ ] The script file referenced by the entrypoint `tsx apps/<x>/scripts/*.ts` is included in the build context (verify the `.gcloudignore` pattern manually, run `gcloud builds submit` and inspect the build log, or run the Job once after deploy)
- [ ] The script path matches what the Pulumi Cloud Run Job definition references

---

## Pattern 2: Missing transitive dependency declarations under `pnpm deploy --legacy --prod`

### Problem

In the `pnpm deploy --legacy --prod` Dockerfile pattern introduced in PR #678, **even transitive dependencies of workspace packages** must be declared directly in the app's `package.json` `dependencies`. Otherwise the runtime fails with `ERR_MODULE_NOT_FOUND`.

`pnpm deploy` generates `node_modules` at the deploy target, but under `--legacy` hoisting of transitive dependencies is limited, so packages that the code directly `import`s but does not list in its own `package.json` are not found.

### Known violations (latent)

- `apps/bot/libby-fassy` — `google-auth-library`
- `apps/graph/service-product/scripts/lib/metrics.ts` — `@google-cloud/firestore`

### Resolution

In any app whose Dockerfile uses `pnpm deploy --legacy --prod`, every package the app `import`s must be declared directly in the app's `package.json` `dependencies` — even if it would otherwise resolve through a workspace package.

```jsonc
// Bad: relies on a shared workspace package's indirect dependency, but does not declare it
{
  "dependencies": {
    "@cortex/otel": "workspace:*"
    // @google-cloud/bigquery is a transitive dependency of @cortex/otel, but is not declared
  }
}

// Good: directly imported packages are declared directly
{
  "dependencies": {
    "@cortex/otel": "workspace:*",
    "@google-cloud/bigquery": "catalog:"
  }
}
```

### Checklist

When adding or modifying an app whose Dockerfile uses `pnpm deploy --legacy --prod`:

- [ ] Every package the source code directly `import`s is declared in this app's `package.json` `dependencies`
- [ ] You have verified the app does not rely on transitive dependencies reached through a workspace package

---

## Pattern 3: Missing OTel exporter env injection

### Problem

The cortex convention is that every Cloud Run Service / Job calls `initOtel({ serviceName: '...' })` at the top of `apps/.../src/index.ts`. However, `@cortex/otel` skips initialization unless **both** `OTEL_EXPORTER_OTLP_ENDPOINT` and `GRAFANA_CLOUD_API_KEY` are set (`packages/infra/shared/otel/src/index.ts`).

If the Pulumi side forgets to inject either of these via `valueSource.secretKeyRef`:

- **Nothing** — no traces, no metrics, no structured logs — reaches Grafana Cloud
- When exceptions occur, no exception event is recorded, leaving only an empty `error` on stdout and making root cause analysis impossible
- The `Pipeline Sync Stale (...)` family of alerts in `infra/observability/grafana-alert-rules.ts` assume logs reach Grafana Cloud, so without a data source they cannot satisfy their firing criteria

This trap was triggered in real production in `infra/pipeline/channel-talk`: the `channel-talk-analyzer` Cloud Run Job failed for 8 consecutive days while remaining invisible to Grafana (Tier 2 stuck at 0 rows; `messages_masked` / `user_chats_safe` empty).

### Resolution

Always inject the core stack's Secret references into Cloud Run Service / Job definitions in `infra/pipeline/<name>/index.ts`.

```ts
const grafanaOtlpEndpointSecretId = coreStack.getOutput(
  'grafanaOtlpEndpointSecretId',
) as pulumi.Output<string>;
const grafanaApiKeySecretId = coreStack.getOutput('grafanaApiKeySecretId') as pulumi.Output<string>;

// Bind secretAccessor to the SA in question
new gcp.secretmanager.SecretIamMember(`${prefix}-<name>-grafana-endpoint-access`, {
  secretId: grafanaOtlpEndpointSecretId,
  role: 'roles/secretmanager.secretAccessor',
  member: serviceAccountMember(saEmail),
});
new gcp.secretmanager.SecretIamMember(`${prefix}-<name>-grafana-key-access`, {
  secretId: grafanaApiKeySecretId,
  role: 'roles/secretmanager.secretAccessor',
  member: serviceAccountMember(saEmail),
});

// Inject into envs
{
  name: 'OTEL_EXPORTER_OTLP_ENDPOINT',
  valueSource: { secretKeyRef: { secret: grafanaOtlpEndpointSecretId, version: 'latest' } },
},
{
  name: 'GRAFANA_CLOUD_API_KEY',
  valueSource: { secretKeyRef: { secret: grafanaApiKeySecretId, version: 'latest' } },
},
```

Reference implementation: the `secretEnvs` block in `infra/pipeline/jobcan/index.ts`.

### Checklist

When adding a new Cloud Run Service / Job:

- [ ] The app side (`apps/.../src/index.ts`) calls `initOtel({ serviceName: '...' })`
- [ ] The Pulumi side (`infra/.../index.ts`) injects both `OTEL_EXPORTER_OTLP_ENDPOINT` and `GRAFANA_CLOUD_API_KEY` into envs via `secretKeyRef`
- [ ] `roles/secretmanager.secretAccessor` is bound to the SA for both secrets via `gcp.secretmanager.SecretIamMember`
- [ ] After deploy, you have verified that the logs do not contain `[OTel] OTEL_EXPORTER_OTLP_ENDPOINT not set, skipping initialization`

---

## Pattern 4: FAILED revision when `secretKeyRef version:latest` references a Secret with no versions

### Problem

When Cloud Run v2 creates a revision that references a `secretKeyRef version:latest`, the revision creation FAILS if the target Secret has **no versions at all**.

For external PATs, passwords, and other secrets where `createSecret` cannot be passed a `value`, Pulumi creates only the Secret resource and adds no version. Deploying as-is produces a FAILED Cloud Run revision and the service never starts.

### Resolution

If you cannot pass a `value` to `createSecret`, create a placeholder version in the same `infra/*.ts` file with `new gcp.secretmanager.SecretVersion(...)`, using `secretData: 'PLACEHOLDER'` and `ignoreChanges: ['secretData']`, so that the first deploy can succeed. After deploy, replace the value manually from the GCP Console.

```ts
const mySecret = createSecret({
  name: 'my-external-pat',
  accessorServiceAccount: serviceAccount.email,
  // No value passed (external PAT that cannot be committed to source).
});

// Cloud Run secretKeyRef requires at least one version to exist or the revision
// creation will fail, so we create a placeholder for the first deploy.
new gcp.secretmanager.SecretVersion(`${prefix}-my-external-pat-placeholder`, {
  secret: mySecret.id,
  secretData: 'PLACEHOLDER',
  deletionPolicy: 'DELETE',
}, { ignoreChanges: ['secretData'] });
```

After deploy, open the target Secret in Secret Manager from the GCP Console and add the real value as a new version manually. The `ignoreChanges: ['secretData']` setting prevents subsequent `pulumi up` runs from overwriting this value.

### Checklist

When a Cloud Run Service / Job references a Secret via `secretKeyRef` and `createSecret` was called without a `value`:

- [ ] You have added `new gcp.secretmanager.SecretVersion(...)` with `secretData: 'PLACEHOLDER'` and `ignoreChanges: ['secretData']` in the same `infra/*.ts` file
- [ ] After deploy, you have replaced the placeholder with the real value manually from the GCP Console
- [ ] The runbook documents the flow: "Pulumi creates a PLACEHOLDER automatically → replace it manually with the real value → `ignoreChanges` prevents future overwrites"

---

## Pattern 5: `pnpm exec <cli>` is forbidden in `node:slim` production images

### Problem

`node:slim`-based production Docker images do not include `pnpm`. If application code tries to launch a CLI tool like `pnpm exec secretlint` at runtime via `execFile` / `spawn`, it fails with `pnpm: not found`.

`bot-secretlint` ran into this in production: because it invoked `pnpm exec secretlint`, no scan could run at all on the Cloud Run Service (PR #1011).

### Resolution

Resolve CLI binaries invoked at runtime via a module-relative path to `node_modules/.bin/<tool>`:

```ts
import { dirname, resolve } from 'node:path';
import { fileURLToPath } from 'node:url';

// Bad: fails on the node:slim image because pnpm is not installed
await execFile('pnpm', ['exec', 'secretlint', ...args], { cwd: workdir });

// Good: reference node_modules/.bin directly
const secretlintBin = resolve(
  dirname(fileURLToPath(import.meta.url)),
  '../node_modules/.bin/secretlint',
);
await execFile(secretlintBin, args, { cwd: workdir });
```

### Checklist

When launching a CLI tool at runtime from a Cloud Run Service / Job:

- [ ] You have verified that the first argument of `execFile` / `spawn` / `spawnSync` is not `'pnpm'`
- [ ] The binary is resolved via a relative path to `node_modules/.bin/<tool>`
- [ ] The package that owns the resolved binary is declared directly in this app's `package.json` `dependencies`

---

## Pattern 6: Never combine `pulumi up --refresh` with the `gcp-dev` ESC env

### Problem

The `gcp-dev` ESC environment passes `pulumiConfig.gcp:accessToken: ${gcp.login.accessToken}`, and Pulumi **stores this value in state** as the `accessToken` input of the default `pulumi:providers:gcp` resource.

The refresh phase of `pulumi up --refresh` reconstructs the provider from the saved provider input in state — meaning it hits the GCP API with the access token that was issued during the previous deploy (1-hour TTL, now expired) and immediately fails with `googleapi: Error 401 ACCESS_TOKEN_EXPIRED`.

`pulumi up` without `--refresh` does not hit this because the provider input is updated before deploy. Only `pulumi refresh` on its own, and `pulumi up --refresh`, break.

Note that this is not a token / authentication problem — the token itself has a 1-hour TTL and is not expired (a freshly issued token will work fine if you hit the API directly via curl). It is a state-semantics problem about *which* token Pulumi chooses to use.

When `#1201` added `--refresh` to every deploy step, all 82 stacks referencing `gcp-dev` started failing on their next deploy, and three consecutive stack failures surfaced in `#1206` / `#1203` / `#1202`. The change was reverted in `#1211`.

### Resolution

- **Do not** pass `--refresh` to `pulumi up` in `.github/workflows/deploy-stack.yml`
- If drift detection is needed, run it as an independent scheduled refresh job (weekly or daily), decoupled from the deploy path
- When `pulumi refresh` is genuinely needed (e.g. `pending_operations > 0`), run `pulumi refresh` on its own **as a separate step before deploy**, not `pulumi up --refresh`. Note that even this still hits the state-stored token problem; the root fix requires changing the ESC env

### Root cause fix (future work)

Remove `pulumiConfig.gcp:accessToken` from the `gcp-dev` ESC env and pass the token only via `environmentVariables.GOOGLE_OAUTH_ACCESS_TOKEN` (the Pulumi-recommended approach). Environment variables are not stored in state, so refresh stops breaking.

However, since the `accessToken` input is already recorded in the state of all 82 existing stacks, a migration step is needed: after changing the ESC env, run `pulumi up` (without `--refresh`) once per stack to clean the provider input.

### Checklist

When invoking `pulumi` from `.github/workflows/deploy-stack.yml` or similar:

- [ ] `pulumi up` is not invoked with `--refresh`
- [ ] If drift detection is needed, it runs as a scheduled job independent of the deploy path
- [ ] New ESC envs do not use `pulumiConfig.gcp:accessToken`; the token is passed via the env var `GOOGLE_OAUTH_ACCESS_TOKEN` instead

---

## References

- [gcp-sdk-usage.md](./gcp-sdk-usage.md): GCP SDK usage restrictions under Cloud Run + OTel
- [recurrence-prevention.md](./recurrence-prevention.md): Required outputs for recurrence prevention
