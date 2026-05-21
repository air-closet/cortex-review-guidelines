# Documentation Guideline

How to write and maintain documentation in the cortex repository.

---

## Principles

### Code is the single source of truth

Documentation is at best a derivative of the code. Documentation that contradicts the code is actively harmful. Before writing documentation, consider whether the information can be expressed in code, types, tests, or lint rules.

### Documentation value decays over time

A document's value peaks the moment it is written and degrades from there. Do not write documentation you cannot afford to maintain. If you write it, build the mechanism for maintaining it at the same time.

### Readers can read code

The primary audiences for `docs/` in cortex are developers and AI agents. There is no need to re-explain *how it works* in clearer terms than the code does. What readers actually need are two things: *why* it works that way, and *where to start reading*.

### Priority of expression

When you want to preserve a piece of information, work through the options below from the top down. If something can be expressed at a higher level, do not express it at a lower one.

> This is the order in which to *choose a means of expression* — it is not a ranking of each option's importance. For example, missing `@graph-*` annotations are defined as Critical in review-guidelines.md.

| Priority | Means | Reason |
|--------|------|------|
| 1 | Type definitions | Verified by the compiler. Zero runtime cost |
| 2 | Lint rules | Run every time. Reported with file:line. Supports `--fix` |
| 3 | Tests | Run in CI. Behavior expressed as assertions |
| 4 | Schema validation | Verified at the boundary at runtime |
| 5 | CI checks | Catch what lint and tests cannot |
| 6 | `@graph-*` JSDoc annotations | Annotate the code and a Product Graph node is generated automatically. Records Why, connections, and domain in a structured form |
| 7 | Generated documentation | Derived from code. Regeneration prevents drift |
| 8 | Hand-written documentation | The last resort. Use only when none of the above can express it |

---

## What to write

### Why documentation

Records *why it is the way it is* — the decisions behind the code that cannot be read off the code itself.

### How-to-get-started documentation

**Setup procedures** — anything that can be automated should be a script, and the documentation should explain only how to run the script.

### Application documentation

For each app, record its purpose, architecture, endpoints, configuration, and deployment information.

What to include:
- Purpose (one or two sentences describing what the app does)
- Architecture diagram (mermaid; the end-to-end data flow)
- Why this architecture (the rationale for this design and the alternatives that were rejected)
- Key design points (the non-obvious things you cannot pick up just by reading code: rate-limiting strategy, pagination approach, retry design, and so on)
- Infrastructure-mediated dependencies (apps and pipelines connected via EventArc, Cloud Scheduler, Pub/Sub, etc. — the implicit dependencies that `package.json` does not capture)
- Endpoint table
- Configuration (environment variables, `pipeline_config` fields, etc.)
- Deployment information (stack name, Pulumi path)

### Operational documentation

**review-guidelines.md** — the decision criteria for code review. Because it is also used as a prompt for the automated review system, avoid vague phrasing and write in terms that can be evaluated mechanically.

**observability.md** — the structure of the observability stack and the design principles for alerts and dashboards.

### Presentation and sharing material

Goes under `docs/sharing/` and `docs/presentation/`. Presentation material is not intended to reflect the accurate current state of the system. Do not cite it as justification for design decisions.

---

## What not to write

### Prose descriptions of the current implementation

Statements of the form "this system currently works like X". These become lies the moment the code changes.

Point at code entry points instead ("start reading from `apps/pipeline/shopify/src/index.ts`").

### Hand-written API references

Hand-maintained request/response specifications for endpoints. Use generation from an OpenAPI schema or derivation from the type definitions. If hand-written content is unavoidable, limit it to an endpoint list (three columns: name, method, role).

### Completed plans left in place

Plan documents that remain after implementation is finished, marked "implemented". AI agents read them as context indiscriminately. Move completed plans to `plans/archived/`.

### Information that belongs in code comments

The meaning of function arguments, the steps of an operation, the logic of a conditional branch. Write these as inline comments or JSDoc in the code. Separating them into `docs/` guarantees they will not stay in sync with the code.

### The same information in multiple places

Information has one canonical location. All other places link to it.

### Raw URLs and file paths

Write URLs as clickable Markdown links — `<https://example.com>` or `[label](https://example.com)`. Refer to repository files the same way: `docs/foo.md _(cortex internal reference)_`.

Put URLs and file paths in inline code or code blocks only when they need to appear as code samples or command examples.

### Subjective, feeling-based quality criteria

Phrases like "write clean code" or "design things properly". Write concrete, evaluable criteria instead. If a lint rule or test can verify it mechanically, prefer that.

### MCP documentation that exposes internal implementation

MCP server documentation is for the user. Do not include implementation details a user does not need. What belongs there is the tool list, use cases, and setup steps.

### Stale technology stack lists

Version-numbered enumerations of dependencies. `package.json`, `Pulumi.yaml`, and Dockerfile are the canonical sources of version information. Under `docs/`, record only the *why* of technology choices, as ADRs.

---

## Consistency

### Do not leave contradictions between documents

When the same concept, threshold, rule, configured value, or term definition appears in more than one document, confirm that every occurrence says the same thing. Updating one and leaving the other contradictory robs readers — humans and AI agents alike — of any way to tell which is correct, and the credibility of the entire documentation collapses.

Procedure:

1. Extract the subjects (terms, thresholds, decision criteria, rules) of the changed document
2. Use Product Graph MCP semantic search to find other Document nodes covering the same subject: `search_product_graph_nodes(query: '<subject>', search_mode: 'semantic')`
3. Verify that descriptions in the matching documents do not contradict the updated content
4. If they do, either align all documents in the same PR or pick one canonical location and replace the others with links to it (information normalization)

Typical contradictions:

- The same threshold or limit (90% coverage, Slack's 40,000-character limit, token caps, etc.) appears with different values
- The same term is defined differently or scoped differently across documents
- An item one document calls "forbidden" is allowed in another with caveats
- The same criterion is mapped to different severities (Critical / Major, etc.) across guidelines
- Procedures, commands, paths, or stack names appear with different spellings across documents

| severity | scope | check |
|---|---|---|
| Critical | All PRs containing documentation changes | The description in the changed document does not contradict the documents it references |
| Critical | All PRs containing documentation changes | Severity, prohibitions, thresholds, and other decision-affecting statements match the referenced documents |
| Major | All PRs | The same concept is not duplicated across documents (duplication breeds contradiction; consolidate to one canonical location and replace others with links) |

Contradictions correspond to "Documentation inconsistency" in [severity.md](./severity.md) and should `REQUEST_CHANGES` at Critical.

---

## Lifecycle

### Updates

Update documentation in the same PR as the corresponding code change. "I'll update it later" is functionally identical to "I won't update it".

### Markdown lint

`README.md` and `docs/**/*.md` are validated with `markdownlint-cli2`. Configuration lives in `.markdownlint-cli2.jsonc`, kept at `default: false` so that only the rules cortex actually needs are explicitly enabled.

Enabled rules:

| Rule | Purpose |
|--------|------|
| MD007 | Align list indentation |
| MD011 | Detect reversed-link typos |
| MD024 | Prevent duplicate headings at the same level |
| MD034 | Force URLs and email addresses into explicit link form |
| MD039 | Forbid stray whitespace around link text |
| MD041 | Require the document's first heading to be H1. `allow_preamble: true` to permit Marp's HTML wrapper |
| MD042 | Forbid empty links |
| MD051 | Validate the target heading of fragment links |
| MD052 | Detect undefined references in reference-style links |
| MD053 | Detect unused link references |

MD025, MD031, and MD040 are deferred to phased remediation because there are too many existing violations. Formatting rules whose responsibility overlaps with Prettier (MD003, MD012, MD013, MD030, etc.) are not enabled on the markdownlint side.

### Retirement

- Move completed plan documents to `plans/archived/`
- When an app is deleted, delete its corresponding documentation
- Delete documents whose content has diverged from reality if the cost of keeping them up to date is no longer justified — git history makes them recoverable

### When in doubt about whether to write

| Question | If yes |
|------|---------|
| Can it be expressed in code or types? | Don't write the doc. Write the code |
| Can a lint rule or test verify it? | Don't write the doc. Write the rule |
| Can it be auto-generated? | Don't hand-write it. Write the script |
| Can a single update keep it accurate? | Write it once, in the canonical location |
| Will it still be accurate in six months? | If no, don't write it |
| Will readers be stuck without this information? | If no, don't write it |
