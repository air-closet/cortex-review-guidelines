# cortex Review Guidelines

Code review guidelines used by both human reviewers and the **Auto Review** AI agents on the [cortex](https://ryantsuji.dev/about) AI-first development platform at [airCloset, Inc.](https://corp.air-closet.com/).

> **Disclaimer**: "cortex" referenced here is the internal code name of airCloset's AI development platform. It is **unrelated** to existing commercial services such as Snowflake Cortex or Palo Alto Networks Cortex.

## What is this?

This repository is a **snapshot of the actual review guidelines** powering production PR reviews in cortex. Every PR — whether AI-generated or human-written — is reviewed against these guidelines first by AI (`Auto Review`), then optionally by humans for cases the AI cannot judge alone.

The guidelines are designed to be:

- **Used by AI reviewers as a hard contract** — each guideline file is loaded as context by a dedicated sub-agent (`arch-reviewer`, `graph-reviewer`, `security-reviewer`, etc.)
- **Severity-graded** — Critical / Major / Minor / Nit, with explicit **no-downgrade rules**
- **Anti-pattern aware** — explicit checks for AI-generated code traps (hallucinated APIs, fallback abuse, dead code, scope creep, premature abstraction)
- **Recurrence-driven** — every bug fix gets a verdict: lint rule, horizontal rollout, new guideline, or "do nothing" — recorded in the PR

This is the **source of the article series**: [The Heart of the AI Harness](https://ryantsuji.dev/series/building-ai-harness) — particularly **Part 3: Auto Review**, where the full review pipeline is dissected.

## Structure

```
.
├── ja/                            # Japanese (original)
│   ├── review-guidelines.md       # Entry point
│   └── guidelines/                # Per-domain guidelines
│       ├── README.md              # Guideline index
│       ├── architecture.md        # Composable Architecture
│       ├── graph-integrity.md     # Product Graph / @graph-* tags
│       ├── security.md            # Auth, input validation, secrets
│       ├── gcp-sdk-usage.md       # Cloud Run + OTel constraints
│       ├── testing.md             # Test design, quality, naming
│       ├── observability.md       # Logs, Slack, alert messages (no truncate)
│       ├── ai-antipattern.md      # ★ AI-generated code anti-patterns
│       ├── document-writing.md    # Document placement and management
│       ├── impact-analysis.md     # Impact range via Product Graph
│       ├── recurrence-prevention.md # Verdict matrix for bug fixes
│       ├── severity.md            # ★ Critical/Major/Minor/Nit + no-downgrade
│       ├── external-api-clients.md
│       ├── lint-rules.md          # Custom ESLint rule policy
│       ├── internal-member-identifier.md
│       ├── cloud-run-deploy.md    # Cloud Run known traps
│       ├── package-publish.md     # GitHub Packages workflow
│       └── frontend.md            # apps/web UI/UX baseline
└── en/                            # English translation
    └── (same structure)
```

★ Recommended starting points: [`severity.md`](./en/guidelines/severity.md) and [`ai-antipattern.md`](./en/guidelines/ai-antipattern.md).

## How AI uses these guidelines

Each guideline file has an associated `Auto Review` sub-agent. When a PR is opened:

1. The orchestrator agent identifies which guidelines are relevant based on the diff
2. Sub-agents are invoked in parallel, each loaded with one guideline file as context
3. Each sub-agent emits findings tagged with **severity** (Critical / Major / Minor / Nit)
4. Findings are aggregated; any Critical / Major blocks the merge (`REQUEST_CHANGES`)
5. For severity downgrades or guideline relaxations, **human approval is required**

The full pipeline (webhook → Product Graph context → sub-agents → findings → auto-fix → re-review → merge → parallel deploy) is described in the Part 3 article.

## How to use this in your team

1. **Read the entry point**: [`en/review-guidelines.md`](./en/review-guidelines.md)
2. **Start with two files**: [`severity.md`](./en/guidelines/severity.md) and [`ai-antipattern.md`](./en/guidelines/ai-antipattern.md) — these encode the most reusable policy
3. **Adapt to your stack**: this snapshot is opinionated for TypeScript / pnpm / Cloud Run / Pulumi — substitute where needed
4. **Wire up to your AI reviewer**: each `Auto Review` sub-agent loads one guideline file. If you use Claude Code, Codex CLI, or similar, you can point them at these files as system context

## Languages

- **Japanese** (original): [`ja/`](./ja/) — the source-of-truth used in production
- **English** (translation): [`en/`](./en/)

Both languages are kept in sync. When the production cortex platform updates a guideline, both versions are updated together.

## Related

- **Article series**: [The Heart of the AI Harness (Building AI Harness)](https://ryantsuji.dev/series/building-ai-harness)
- **Part 2 article (Product Graph)**: [The Heart of the AI Harness: A Knowledge Graph of the AI, by the AI, for the AI](https://ryantsuji.dev/posts/cortex-product-graph)
- **AI antipattern observation roots**: [nrslib/takt's ai-antipattern-reviewer persona](https://github.com/nrslib/takt/blob/main/builtins/ja/facets/personas/ai-antipattern-reviewer.md) — partially adapted for cortex's context

## License

MIT — feel free to fork, adapt, and use in your own organization.

---

This repository is maintained as a public snapshot. Production changes are mirrored periodically. The guidelines themselves are **opinionated and evolving** — they reflect real production trade-offs, not theoretical purity.
