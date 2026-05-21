# Lint Rule Authoring and Operations Guidelines

In cortex, inline `no-restricted-syntax` / `no-restricted-imports` rules inside
`eslint.config.*` files and presets are forbidden. The same intent must instead be
implemented as a custom ESLint rule and exposed from `@cortex/eslint-plugin-graph`.

## Why inline rules are forbidden

1. **AST selector strings cannot be type-checked** — selector strings like
   `CallExpression[callee.property.name='toContain']` are not validated by your
   editor, the type system, or the compiler. When the AST specification changes,
   they silently stop matching.
2. **No DRY** — the same intent gets copy-pasted across multiple workspaces, and
   the messages and selectors drift apart over time.
3. **They invite `eslint-disable`** — cortex forbids `eslint-disable` entirely
   (AGENTS.md _(cortex internal reference)_). Loose configuration makes it easier
   for that policy to slip in practice.
4. **Not testable** — selector-string logic cannot be covered by vitest, and the
   reviewer has no clear bar to apply.

## Recommended flow

### Step 1: Check whether an existing rule already covers it

Review
[`packages/eslint-plugin-graph/src/rules/`](../../packages/eslint-plugin-graph/src/rules/).
Common replacements:

| Intent | Rule to use |
|---|---|
| Forbid weak Vitest matchers (`toContain`, `toBeTruthy`, etc.) | `graph/vitest-strong-matchers` |
| Enforce that barrel `index.ts` files re-export only | `graph/reexport-only-index` |
| Validate `@graph-*` JSDoc tags | `graph/valid-graph-*`, `graph/require-graph-*` |
| Detect inline-restriction rules in `eslint.config.*` | `graph/no-restricted-syntax-direct-usage` |

### Step 2: Add a custom rule

Implement any new restriction as a custom ESLint rule.

1. Add the rule body at `packages/eslint-plugin-graph/src/rules/<rule-name>.ts`
   - Create it with `ESLintUtils.RuleCreator`
   - Define fixed messages in `meta.messages` and reference them by `messageId`
   - Write AST visitors against `AST_NODE_TYPES` from `@typescript-eslint/utils`
     for type safety
2. At `packages/eslint-plugin-graph/src/rules/<rule-name>.test.ts`, cover both
   valid and invalid cases with `RuleTester` from `@typescript-eslint/rule-tester`
   (coverage 90% or higher)
3. Register the rule in the `rules` map in
   `packages/eslint-plugin-graph/src/index.ts`
4. Reference it from the presets at `packages/core-ecosystem-config/eslint/` as
   `'graph/<rule-name>': 'error'`
5. Regenerate the inline snapshot in
   `packages/eslint-plugin-graph/src/index.test.ts`

### Step 3: Enforce the no-inline policy with lint

The `graph/no-restricted-syntax-direct-usage` rule raises an error whenever an
object-literal key for `no-restricted-syntax` / `no-restricted-imports` appears
inside an ESLint config or preset. Any new `eslint.config.*` that uses an inline
restriction will fail `pnpm lint:eslint`.

## Allow-list policy

Exceptions to this guideline are not permitted. When new syntax or imports need
to be restricted:

1. Check whether an existing custom rule can express the restriction
2. If not, add a new custom rule
3. If implementing a custom rule is not justified, reconsider the design that
   created the need to restrict the syntax in the first place

Bypassing rules via disable comments is not permitted, in accordance with the
prohibitions listed in AGENTS.md.

## Related

- [packages/eslint-plugin-graph](../../packages/eslint-plugin-graph/) — the custom rules themselves
- [packages/core-ecosystem-config/eslint](../../packages/core-ecosystem-config/eslint/) — preset bundles
- [docs/guidelines/testing.md](./testing.md) — test quality standards
- AGENTS.md _(cortex internal reference)_ — the philosophy behind the blanket ban on rule-disable comments
