# Architecture and Design Patterns

## Core Philosophy: Composable Architecture

cortex's design philosophy is **"Design small functional units, then compose them into services"**, i.e. Composable Architecture.

- Place **reusable building blocks** in `packages/` (domain logic, infrastructure utilities, type definitions, technical foundations)
- Each service under `apps/` is built by composing parts from `packages/`
- When building a new service, first check whether you can assemble it from existing parts. Design new parts only for what's missing

Each item below carries an explicit **severity** and **scope**. The auto-reviewer does not raise an item on a PR that falls outside its scope.

| severity | scope | Check |
|---|---|---|
| Major | all apps | No direct imports between apps (`../../apps/other/`) — always go through `packages/` |
| Major | all apps | New logic is designed as appropriately-sized "parts" (one responsibility per file) |
| Major | all apps | App-specific logic is separated from shareable logic |
| Major | all apps | When multiple apps reference the same writer/reader contract (BQ column values, Firestore keys, Pub/Sub payloads, shared enums, etc.), the contract values themselves (literal constants and types) are extracted into `packages/` and imported from both apps[^writer-reader-sot] |
| Minor | all apps | Existing parts in `packages/` are reused (no wheel reinvention) |
| Minor | all apps | Extraction to `packages/` follows the **rule of three** (extract once duplication actually appears in two or more apps). Speculative extraction "in case it's useful later" produces zero-consumer packages and is forbidden as a YAGNI violation |

[^writer-reader-sot]: Real-world example: the styling-pattern-watch transformer wrote a default `segment_name` value of `'全体（在庫あり）'` to BQ, while the api used `'all'` in its WHERE clause for dashboard rendering. The two values were independently defined in each app, so a deploy that updated only one side caused the API to return an empty array and the dashboard to display all zeros. The fix was to create `@cortex/styling-pattern-watch-core` and have both apps import `defaultSegments()` as the SoT. General AST-level lint detection is difficult here (any string literal could be a contract), so this guideline check covers it instead.

## Structural Patterns by App Type

`apps/` is organized into category subdirectories, and each category has an established structural pattern. When adding a new app, **follow the same pattern by referencing existing apps in the same category**.

### API (`apps/api/`)

Feature-Based DDD + Clean Architecture. Each `features/{name}/` contains `api/`, `domain/`, `application/`, and `infrastructure/` layers.

| severity | scope | Check |
|---|---|---|
| Major | `apps/api` | The dependency direction `API → Application → Domain ← Infrastructure` is preserved |
| Major | `apps/api` | Use cases follow single-responsibility (one file = one business flow) |
| Major | `apps/api` | OpenAPI schemas are documented via `.openapi()` |

### Bot (`apps/bot/`)

Slack Bot. **The bot itself holds no business logic** — it only receives, routes, and replies to Slack events.

| severity | scope | Check |
|---|---|---|
| Major | `apps/bot` | The bot does not implement business logic directly (delegate to `packages/domain-*`) |
| Major | `apps/bot` | Handlers stay thin; logic is separated into features |

### Pipeline (`apps/pipeline/`)

Data ingestion from external services. Flat structure.

| severity | scope | Check |
|---|---|---|
| Critical | `apps/pipeline` | Idempotency is guaranteed against Pub/Sub retries |
| Major | `apps/pipeline` | Streaming transfer is used (entire dataset is not held in memory) |

### CLI (`apps/cli/`)

Workspace packages for local execution only. CLI groups invoked via `pnpm exec <bin-name>`, organized by bucket.

| severity | scope | Check |
|---|---|---|
| Critical | `apps/cli` | Product Graph exception procedures are reflected consistently in [graph-integrity.md](./graph-integrity.md) and [../review-guidelines.md](../review-guidelines.md), with no contradictions in CLI-facing docs |
| Major | `apps/cli` | Designed as a local operations tool, not as a deployable app |
| Major | `apps/cli` | Stands alone as a package with its own `package.json`, `tsconfig.json`, `oxlint.config.ts`, and `vitest.config.ts` |
| Minor | `apps/cli` | The external invocation surface is consolidated in `bin`, not relying on direct calls into `src/*.ts` |

### Generator (`apps/generator/`)

AI analysis and generation batches (KPI, Bug Evaluation, Stylings Analysis, etc.). A flat `src/` directory containing `bigquery-client.ts` / `firestore-client.ts` / `config.ts` and business logic (e.g. `kpi-calculator.ts`). Execution is primarily Cloud Scheduler-driven. **Reference: `apps/generator/kpi/`**.

| severity | scope | Check |
|---|---|---|
| Major | `apps/generator` | LLM / BQ / Firestore clients are separated into individual files, structured for easy mocking in tests |
| Major | `apps/generator` | Re-running the batch does not cause duplicate generation or destructive overwrite (idempotency) |
| Minor | `apps/generator` | Business logic is extracted into pure functions and unit-testable |

### Transformer (`apps/transformer/`)

ETL / Embedding / BQ loading (biz-graph, image-processor, styling-pattern-watch, etc.). Responsibilities are split into phase files like `build-*.ts` / `generate-embeddings.ts`. **Reference: `apps/transformer/biz-graph/`**.

| severity | scope | Check |
|---|---|---|
| Major | `apps/transformer` | Phases (build / embed / similarity, etc.) are separated into individual files (one file per phase) |
| Major | `apps/transformer` | Intermediate data is handled as streams (not fully expanded in memory) |
| Minor | `apps/transformer` | Structured to allow partial re-execution (running only a specific phase) |

### MCP (`apps/mcp/`)

MCP servers (cortex-product-graph-server, code-graph-server, etc.). The canonical layout is `server.ts` + `tool-registry.ts` + `mcp/`. Tool definitions are aggregated at the registration layer. **Reference: `apps/mcp/code-graph-server/`**.

| severity | scope | Check |
|---|---|---|
| Critical | `apps/mcp` | Authentication and authorization are applied per tool (write-side tools are not exposed without authentication) |
| Major | `apps/mcp` | Tool implementations are aggregated in `tool-registry.ts` and invoked thinly from `server.ts` |
| Major | `apps/mcp` | MCP documentation (`docs/mcp/{name}.md`) is written for consumers (no implementation details — see [document-writing.md](./document-writing.md)) |

### Graph (`apps/graph/`)

Data-graph apps (code, product, cortex-db, db-dictionary, screen, service-product). Relatively large; the canonical layout splits responsibilities across `agent/`, `ai/`, `behavior/`, `cli/`, and `graph/` subdirectories. **Reference: `apps/graph/code/`**.

| severity | scope | Check |
|---|---|---|
| Major | `apps/graph` | Graph build / analysis / query responsibilities are separated into distinct layers |
| Major | `apps/graph` | Node and edge schemas loaded to BQ are explicitly type-defined |
| Minor | `apps/graph` | Both incremental builds (deltas) and full reindex are selectable |

### Web (`apps/web/`)

Frontend and admin UIs (cortex-admin, backoffice-console, mall, etc.). React + Vite + TanStack Router. The canonical split is `routes/`, `components/`, `hooks/`, `lib/`, `i18n/`. **Reference: `apps/web/cortex-admin/`**.

| severity | scope | Check |
|---|---|---|
| Critical | `apps/web` | Authentication gates (OAuth / auth middleware) cover every protected route |
| Critical | `apps/web` | Required configuration such as API base URL and secrets is validated at startup so that build or boot fails when missing (see "Required configuration and no-fallback policy" in [security.md](./security.md)) |
| Major | `apps/web` | Business logic is not embedded in routes or components but separated into `packages/domain-*` or pure functions in `lib/` |
| Major | `apps/web` | i18n keys are centralized; no hardcoded strings remain |

## Shared Packages (`packages/`)

| Naming pattern | Role |
|-------------|------|
| `domain-*` | Business-domain logic, constants, and types |
| `infra-*` | Cross-cutting infrastructure helpers |
| `core-*` | Common type definitions and shared configuration |
| `*-core` | Foundational utilities for a specific technology |

### Placement Rules

| Location | Purpose | Convention |
|------|------|------|
| `packages/core/` | **Truly app-wide cross-cutting concerns** (e.g. `@cortex/core-types`) | New additions are limited to SoT types and constants used across 30+ apps/stacks. Do not add new subpackages directly under `packages/core/` (contracts and utilities scoped to a specific group of apps belong in `packages/domain/<bc>/`) |
| `packages/domain/<bc>/` | Per-bounded-context domain logic and types | The `*-contract` naming is deprecated. Use bounded-context name + `src/types.ts` / `src/index.ts`. Even with a single caller, shared code scoped to a specific group of apps belongs here |
| `packages/domain/<bc>/<sub>/` | Subpackages within a bounded context (e.g. `domain/product/calendar-jp`) | Used when a bounded context bundles multiple utilities |
| `packages/infra/<layer>/<name>/` | Cross-stack infrastructure libraries | Provides framework-style helpers across stacks under the `@cortex/infra-*` namespace |

The blanket regex for `packages/core/` (in `.github/scripts/detect-changed-stacks.sh`) triggers every stack, so placing non-cross-cutting code here causes excessive deploys. When in doubt, choose `packages/domain/<bc>/` and explicitly state the trigger-target stacks in a new detect-script case.

| severity | scope | Check |
|---|---|---|
| Major | `packages/` | The `@cortex/` namespace and `workspace:*` protocol are used (see below for the externally-published exception) |
| Major | `packages/core/` | No new subpackages are being added (anything that isn't an SoT type or constant used across 30+ apps belongs in `packages/domain/<bc>/`) |
| Major | all apps | Domain logic is not embedded inside apps but separated into `packages/domain-*` |

### Namespace Exception: Externally-Published Packages

Packages that adopt the publish-pattern from [docs/guidelines/package-publish.md](./package-publish.md) and ship to GitHub Packages / npm registry use the **`@air-closet/`** namespace instead of `@cortex/`. `@cortex/` is reserved as the workspace-only namespace inside the monorepo; packages installable by external consumers are distinguished by the organization namespace. Concrete example: `@air-closet/cc-analyzer` (`packages/cc-analyzer/`).

## Role and Permission Design

`PortalRole` in `@cortex/core-types` is a small enum that expresses **organizational position only**.
App-specific access (whether a user can use a given app) is not added to the central `PORTAL_PERMISSIONS` — each app handles it internally.

- [ ] No "role that just names an app" has been added to `PortalRole` in `@cortex/core-types` (e.g. `*_resolver` / `*_manager`)
- [ ] No app gates (access control for specific apps) have been added to `PORTAL_PERMISSIONS` — they belong in each app's middleware and the Firestore `app_access` collection
- [ ] Per-email allowlists go through `AppAccessRepository` + `isAppAccessGranted(...)` from `@cortex/infra-api-app-access` (do not read Firestore directly)
- [ ] The Web side does not re-evaluate organizational roles; it consumes the `canAccess: boolean` returned by the API as-is

`@cortex/infra-api-app-access` is the shared boundary for extending the app_access pattern (introduced in `line-user-id-resolver`) to other per-app gates. Its purpose is to prevent each app from independently re-implementing email-allowlist caching, concurrent-load deduplication, and role fallback.

## Data Architecture and Event-Driven Patterns

| severity | scope | Check |
|---|---|---|
| Critical | apps using BQ | Parameterized queries are used (SQL injection prevention) |
| Critical | Pub/Sub / async processing | Message idempotency is guaranteed |
| Major | apps using BQ | `SELECT *` is avoided; embedding columns explicitly declare `mode: 'REPEATED'` |
| Major | `apps/api` | Long-running work is not executed inside a synchronous API (delegate to async) |
| Minor | apps using Firestore | Collection naming is consistent, and per-use-case role fields are distinguished |

## Infrastructure / Deploy

| severity | scope | Check |
|---|---|---|
| Critical | `infra/` | `Pulumi.prod.yaml` does not include `encryptionsalt` |
| Major | `infra/` | New `packages/` dependencies are added to the Dockerfile with COPY and build steps |
| Major | `infra/` | New stack patterns are added to the change-detection script |
| Major | `infra/pages-*` | When adding a new `pages-*`, both `add_pages` in `detect-changed-stacks.sh` and the `case "$PAGE" in` block in `deploy-stack.yml` are updated (also validated by the CI guard `verify-deploy-pages-integrity.sh`) |
