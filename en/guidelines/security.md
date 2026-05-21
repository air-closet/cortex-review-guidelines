# Security

## Checklist

Each item declares an explicit **severity** and **scope**. The auto-reviewer does not fire an item on a PR that falls outside its scope.

### Authentication / Authorization

| severity | scope | Item |
|---|---|---|
| Critical | `apps/api`, `apps/bot` | Is the authentication middleware applied to authenticated endpoints? |
| Critical | `apps/api`, `apps/bot` | Do write operations (POST / PUT / DELETE) enforce appropriate role checks? |

### Secrets / Sensitive Information

| severity | scope | Item |
|---|---|---|
| Critical | all apps | Are secrets such as API keys, passwords, tokens, private keys, and webhook secrets free of hard-coding in the source? |
| Critical | all apps | Is Secret Manager used to store and distribute secrets (passwords, API keys, OAuth client secrets, private keys, webhook secrets, and similar)? |
| Critical | apps using Firestore | Are secrets kept out of Firestore **as plaintext**? Only references to Secret Manager (e.g. `...SecretId`) should be stored, never the secret itself. |
| Critical | all apps | Are secrets, tokens, passwords, and personal information kept out of logs, error messages, Slack notifications, and Sheets exports? |
| Critical | all apps (including docs / tests / samples) | Are documentation, test data, and sample code free of real secrets? Only dummy values may be used. |
| Major | `infra/` | Are the SAs and IAM bindings that read Secret Manager scoped to least privilege? Only the necessary runtime SAs receive `roles/secretmanager.secretAccessor`. |

### IAM / Permissions

| severity | scope | Item |
|---|---|---|
| Major | `infra/` | Do Cloud Run SA permissions follow least-privilege? |
| Major | `infra/` | Are unnecessary IAM roles kept off the SA? (Both under- and over-grant are problems — we have real cases of silent failures from missing grants.) |

### Input Validation / Injection Prevention

| severity | scope | Item |
|---|---|---|
| Major | `apps/api`, apps accepting external input | Are user input and external API responses validated at the **boundary** against a type-level schema (Zod, etc.)? |
| Critical | apps using a DB | SQL / NoSQL injection prevention: are parameterized queries in use? |
| Major | apps using the Sheets API | When `USER_ENTERED` is used, is there no formula-injection risk (user input must not be written as a cell starting with `=...`)? |
| Minor | apps using the Sheets API | Are spreadsheet IDs kept out of hard-coded literals? |

### Required Configuration and the Fallback Ban

Kill the kind of "silent fallback" that turns missing configuration into a runtime feature outage. This is enforced at the boundary.

| severity | scope | Item |
|---|---|---|
| Critical | all apps | For required configuration (API host, Secret, environment variables, etc.), does the code avoid **silently falling back** to same-origin, empty string, `null`, etc. when the value is missing? The patterns `baseUrl ?? ''`, `process.env.X ?? ''`, making the field `?: string` optional, and returning a default from `try/catch` are forbidden. If the value is required, it must be `required` at the type level and the process must explicitly throw at startup. |
| Critical | `apps/web` | Is there a function (e.g. `getApiBaseUrl()`) that validates `import.meta.env.VITE_*` at startup and throws when a value is missing? Is the design free of optional `baseUrl` / `endpoint` props on components or functions that fall back to same-origin? |

Verification is not only that the bundle / build passes. It must be pinned down by tests that show (a) the build / startup reliably fails when a required value is missing, and (b) the configured values flow all the way to the leaves on a normal startup.

### LLM / External API Clients

| severity | scope | Item |
|---|---|---|
| Major | apps using LLMs | Are errors from LLM calls not swallowed? (Swallowing is a common cause of silent failures.) |
| Major | apps using Vertex AI | Are the Vertex AI SDK initialization parameters correct? (Without `vertexai: true`, the client points at Google AI Studio.) |

### Lint / Conventions

| severity | scope | Item |
|---|---|---|
| Critical | all repositories | Does the code avoid `eslint-disable` / `oxlint-disable` comments? (These are strictly prohibited in cortex.) |
