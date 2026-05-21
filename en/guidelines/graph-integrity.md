# Product Graph Integrity

Product Graph is the core of cortex and the concrete realization of the **"externalization of knowledge"** principle laid out in VISION.md _(cortex internal reference)_. It captures not just the *what* of the code but the *why*, providing the structured backbone for deterministic Graph RAG routing, impact analysis, and cross-repository traceability. **The quality of Product Graph equals the quality of cortex's entire knowledge foundation.**

Each item below specifies a **severity** and **scope**. The auto-reviewer does not fire an item on PRs that fall outside its scope.

## `@graph-*` JSDoc tags (mandatory)

`@graph-*` tags are the Single Source of Truth for Product Graph. Within `@graph-*`-targeted files under `apps/`, `packages/`, and `infra/`, every top-level declaration (functions, classes, variables) is mechanically validated regardless of whether it is exported (enforced by ESLint `graph/require-graph-connects`).

| severity | scope | check |
|---|---|---|
| Critical | `apps/`, `packages/`, `infra/` (excluding cli) | New function / class / variable declarations carry a `@graph-connects` tag (use `@graph-connects none` when there is no connection) |
| Critical | `apps/`, `packages/`, `infra/` (excluding cli) | External connections are written as `@graph-connects {target} [{edgeType}] {description}` |
| Critical | `apps/`, `packages/`, `infra/` (excluding cli) | `@graph-business` records the business context (in Japanese) — this drives Embedding quality |
| Critical | `apps/`, `packages/`, `infra/` (excluding cli) | `@graph-stack` and `@graph-domain` are set at the file level or on the primary function |

**Notes:**
- `graph/require-graph-connects` also requires `@graph-connects` on top-level variables. In review, prefer the mechanically enforced rules over comment wording.
- `graph/suspicious-graph-connects-none` flags declarations marked `@graph-connects none` whose function bodies or variable initializers contain external call patterns.

## `apps/cli/*` exception

`apps/cli/*` packages are workspace packages intended for local execution only — they are not deployment targets like Cloud Run / Worker / Pages. They are currently excluded from Product Graph rules in `eslint.config.js`, and `@graph-*` tags are not required.

| severity | scope | check |
|---|---|---|
| Critical | `apps/cli` | If you bring `apps/cli/*` under the graph rules, you must update `eslint.config.js`, this guideline, [../review-guidelines.md](../review-guidelines.md), and [architecture.md](./architecture.md) **in the same PR** |
| Major | `apps/cli` | Do not mechanically require `@graph-*` tags on `apps/cli/*/src/**/*.ts` (avoid lint false positives) |
| Minor | `apps/cli` | The CLI package's design intent and operational rules are documented under `docs/` and shared among reviewers, with a clear note that they live outside Product Graph |

## Stack, domain, and edge consistency

| severity | scope | check |
|---|---|---|
| Critical | Adding `@graph-stack` | The value is registered in `STACKS` (adding a new value requires a rebuild) |
| Critical | Adding `@graph-domain` | The value matches one of the existing domains (unknown values are treated as `@graph-*` inconsistencies) |
| Critical | Declarations with `@graph-connects` | The edge type (`calls`, `queries`, `writes_to`, `reads_from`, `triggers`, `publishes`, `references`, etc.) faithfully reflects the actual connection. The allowlist comes from ../product-graph/README.md _(cortex internal reference)_ as the primary source |
| Major | Declarations with cross-repository connections | Boundary nodes are properly declared |

## Documentation consistency (`docs/`)

`.md` files under `docs/` are ingested into Product Graph as Document nodes and connected to code nodes via `documented_by` edges. A stack with no documentation has no Document node in the graph, which breaks the documentation-to-code path for semantic search.

**Documentation must follow [document-writing.md](./document-writing.md).** An update that falls under document-writing.md's "What not to write" section (e.g. prose descriptions of the current implementation) does not satisfy the documentation update requirements below (Critical). Such an update must additionally be flagged as Critical under the table below (changes to existing stacks).

> **Required:** Every PR that adds or modifies behavior must check whether the corresponding documentation (`docs/{category}/{name}.md`) exists and has been updated. PRs that only modify documentation (no code changes) are exempt from this check.

### Common to all code changes

| severity | scope | check |
|---|---|---|
| Critical | All PRs containing code changes | Documentation for the affected stack / app exists at `docs/{category}/{name}.md` (create it if missing) |
| Critical | All PRs containing code changes | Changes that affect the "Why", architectural decisions, or entry points (as defined by document-writing.md) are reflected in the documentation |

### New stacks / new apps

| severity | scope | check |
|---|---|---|
| Major | New app / stack | The document title (`#` heading) is appropriate as a node name |
| Major | New app / stack | A link has been added to the relevant category `README.md` |
| Major | New app / stack | The document follows `document-writing.md`'s "What to write" and "What not to write" |

### Changes to existing stacks / apps

| severity | scope | check |
|---|---|---|
| Critical | Changes to existing app / stack | The documentation does not diverge from the implementation (no stale information) |
| Critical | Changes to existing app / stack | The update is not a prose description of the current implementation (see [document-writing.md](./document-writing.md), "What not to write") |
