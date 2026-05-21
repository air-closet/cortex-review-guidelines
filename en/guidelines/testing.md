# Test Quality

## Core Principles

### Tests Verify Behavior, Not Implementation Details

Pinning private-method call counts or internal data-structure shapes makes tests easy to break under refactors that did not change the specification. Verify externally observable results — return values, state changes, side effects, public API contracts.

Heuristics:

- If a refactor breaks it, the test is fragile
- If a bug introduction breaks it, the test is valuable

### Design the Test Portfolio in Layers

Take the Test Pyramid as the baseline: lots of small tests, few heavy ones.

- Small / Unit: fast, local, plentiful
- Medium / Integration: verify connection points and contracts
- Large / E2E: a small number of major user journeys only

### Do Not Tolerate Flakiness

"It passes on rerun" is not a successful test — it's a signal of eroded trust. Once you cannot trust red, neither CI nor review functions.

### Coverage Is a Supporting Metric, Not a Goal

What matters is not the number itself, but whether important business rules are guarded, whether failures lead you to the cause, and whether the false-positive rate stays low under change.

## Test Design

### Start From Failure Modes

Before implementing, make these explicit:

1. What breakage would harm users or the business
2. At which layer can we detect that breakage most cheaply
3. At which single layer do we catch a given defect exactly once

Examples:

- Tax-calculation errors → Unit
- Broken API request/response contract → Integration
- A break in the purchase-success flow → E2E

### Standardize on AAA or Given / When / Then

Writing every test in the same structure improves review speed and maintainability.

- Given / Arrange: preconditions and data
- When / Act: execute
- Then / Assert: expected outcome

### One Test, One Intent

Mixing different specifications into a single test makes it impossible to tell what broke when it fails. Test names should read as specification sentences, and assertions should be scoped to reinforce that one specification.

### Pin `index.ts` / Barrel Tests Mechanically

When `index.ts` is just an aggregation of re-exports, the goal of testing it is to cheaply pin "what is being exported." Per-export checks like `typeof ... === 'function'` or `toBeDefined` are weak and high-maintenance.

Default approach:

- Import the runtime exports in bulk via `import * as module from './index.js'`
- Pin `Object.keys(module).sort()` with `toMatchInlineSnapshot()`
- Treat the snapshot as the public contract of export names
- Type exports do not appear at runtime — verify them in type tests, not in `index.ts` tests

Example:

```ts
import { describe, expect, it } from 'vitest';
import * as domainModule from './index.js';

describe('barrel exports', () => {
  it('公開APIを集約している', () => {
    expect(Object.keys(domainModule).sort()).toMatchInlineSnapshot(`
      [
        "createItem",
        "deleteItem",
        "getItem",
      ]
    `);
  });
});
```

When it is appropriate to add further assertions:

- `index.ts` itself holds logic or constants
- Concrete values or behavior of public exports are part of the contract, not just the names
- `index.ts` wraps or composes — not just re-exports

Even then, start by pinning the export list with a snapshot, and add separate tests only for the specifications that matter. Do not blend implementation-detail checks into the barrel test.

## Per-Layer Guide

### Unit (Small)

The goal is fast verification of business rules and boundary conditions.

- Prioritize pure functions, domain logic, and transformations
- Make boundary inputs explicit (empty, minimum, maximum, anomalous)
- Make clock, random, and UUID injectable so tests are deterministic

Avoid:

- Re-testing framework internals
- Over-mocking that only verifies implementation details

### Integration (Medium)

The goal is to catch cases where each piece is correct but the composition breaks.

- Verify connection points like API handler + use case, repository + DB
- Use real implementations for everything except external I/O
- Centralize test-data factories to reduce setup cost

Avoid:

- Mocking everything so the test degrades to the same value as a Unit test

### E2E (Large)

The goal is to guarantee that, on a path close to production, the system actually works.

- Limit to critical paths: revenue, billing, authentication, key business flows
- Keep case count small
- On failure, retain observability info (logs, screenshots, traces)

## Mocking / Stubbing Policy

### Mock Only Uncontrollable or High-Cost Dependencies

Reasonable targets:

- External SaaS APIs
- Side-effect-heavy operations like billing or email
- Slow external systems

### Not What Mocks Are For

- Tests that only verify internal call counts
- Dummy contracts that ignore request/response or DB schemas

### Lock Boundaries With Contract Tests

For service-to-service communication and public APIs, test the consumer/provider contract so that a change on only one side is caught early.

## Rules to Prevent Flakiness

- Pin time (fake timers, fixed clock)
- Pin random seeds
- Hold no shared state
- Cut external network dependencies
- Do not `sleep`; watch for the condition to become true
- Use IDs and resources that do not collide under parallel execution

## Additional Techniques for Stronger Tests

### Property-Based Testing

Beyond example-based tests, verify invariants that always hold. The fuzzing-style search through input combinations that humans don't think to enumerate makes it easier to find missed boundary conditions.

Examples:

- The result of `sort` is monotonically non-decreasing
- The multiset of elements is preserved between input and output

### Mutation Testing

Inject small changes into the target code and observe whether tests fail. Limiting it to important modules makes spec gaps visible.

## Steps to Improve Weak Existing Tests

1. Identify candidates for removal: tests coupled to implementation details, redundant tests with no incremental spec value, tests whose failure messages convey nothing
2. Rewrite as Given / When / Then anchored to specifications
3. Redistribute responsibility across Unit / Integration / E2E
4. Eliminate the root cause of flakiness — do not paper over with retries
5. As needed, use supporting metrics like reproduction rate, mean time to repair, and mutation score

## Reaching Coverage in a New App

The canonical approach when 90% is hard to reach in the initial PR for a new app. **Lowering the threshold is forbidden** — reach it by separating the code into a testable shape.

### Typical Patterns for I/O-Heavy Apps

DB clients, external API clients, OAuth flows, and webhook handlers don't yield testable code in their naive forms, and tend to fall short of 90%. Decompose them as follows.

1. **Extract business logic into pure functions**: code that shapes BQ query results into outputs belongs in a separate file (e.g. `kpi-calculator.ts`) with unit tests. Keep the client as a thin fetch wrapper, and cover I/O via integration tests.
2. **Abstract external SDKs behind interfaces**: define `interface SecretManager { get(name: string): Promise<string> }` and swap implementations between production and tests. Do not exercise SDK calls directly under test.
3. **Keep webhook handlers thin**: limit them to three stages — receive → validate → call domain function. Unit-test validation and domain logic individually; cover the handler itself with a single integration test.

### `// istanbul ignore` Is Forbidden

Hiding coverage gaps with `istanbul ignore` is forbidden. Violations are sent back at Critical.
A line that appears to need `ignore` is a signal that the structure is not yet testable — refactor instead.

### Exception: bin / scripts Entrypoints

`bin/{cli-name}.ts`-style bin entrypoints (only CLI bootstrap and `process.argv` parsing) and one-shot operational scripts in `scripts/*.ts` are not required to reach 90% coverage. Exclude them explicitly via `coverage.exclude` in `vitest.config.ts` (per-line ignore comments are forbidden).

## Team Operating Rules

- Test failures are top priority — do not defer
- When you find flakiness, prioritize root-cause fixes over quarantining
- Include a regression test in the same PR as the bug fix
- Consolidate test helpers and fixtures once duplication appears
- Have a human review the value of AI-generated tests before adoption

## Avoid Weak Vitest Matchers

To prevent "assertions that just pass" from being used to inflate coverage, the following matchers are avoided as a rule.

- `toBeTruthy` / `toBeFalsy`
- `toBeDefined`
- `toBe(true|false)` / `toEqual(true|false)` / `toStrictEqual(true|false)`
- `toContain` / `toContainEqual`
- `expect.any` / `expect.anything` / `expect.objectContaining`

The top choice is `toStrictEqual`, which pins the expected value as an entire object. Do not settle for partial matches or existence checks — compare the entire output that is semantically meaningful.

Per-property tests like `expect(result.hoge)` cannot detect unintended changes that leak into other properties. The baseline is `expect(result).toStrictEqual(...)`, which verifies the invariance of the entire return value or output.

When the test target is large and the `toStrictEqual` expectation becomes hard to maintain by hand, you may use `toMatchInlineSnapshot` / `toMatchSnapshot` to catch changes in the entire output. On top of that, supplementing with individual assertions for business-critical properties is acceptable.

### Handling Dynamic Fields (Timestamps, IDs, etc.)

`toStrictEqual` is exact-match, so dynamic values like `Date.now()` / `crypto.randomUUID()` / DB auto-increment IDs cause flakiness when included. Escaping with `expect.any(Date)` falls into the weak-matcher list above and is not acceptable. Use one of the following to **pin** dynamic values before `toStrictEqual`.

1. **Make clock / random injectable**: pin time with `vi.useFakeTimers()` and mock `crypto.randomUUID` with `vi.spyOn`. Design the target to accept dependency injection such as `clock: () => Date` and `idGenerator: () => string`.
2. **Exclude dynamic fields and compare the rest**: pin via `const { createdAt, id, ...rest } = result; expect(rest).toStrictEqual(...)`. Verify the type or format of the excluded `createdAt` / `id` in a separate assertion (a type-only assertion like `expect(result.createdAt).toBeInstanceOf(Date)` is acceptable).
3. **Pin via snapshot**: use `toMatchInlineSnapshot` and normalize dynamic values via a serializer (`expect.addSnapshotSerializer` to replace UUID → `<UUID>`, Date → `<DATE>`).

Using `expect.any(Date)` / `expect.objectContaining` is sent back as Major in review. "Because it contains dynamic values" is not a sufficient justification.

Instead, use specific matchers that carry spec meaning. Prefer `toStrictEqual` when an exact match is expected. For large external payloads where only partial matching is maintainable, either assert individual important fields or map to the required shape and then pin with `toStrictEqual`.

- `toStrictEqual`
- `toMatchInlineSnapshot`
- `toMatchSnapshot`
- `toHaveLength`
- `toThrow`
- `toMatch`

### "Containment" Tests Are Prohibited as a Rule

Tests that settle for containment may only be used when **containment is genuinely the essence of the spec**. This applies not just to `toContain` / `toContainEqual` but to **any construction that uses "does this string appear?" as the basis for an assertion** — `String#includes` / `Array#includes` / regex `.match()` / `text.split(...).filter(line => line.includes(...))` and similar.

The reason is simple: "it's contained" tells you nothing about what else might be mixed in, whether ordering is correct, or whether an unrelated token of the same name happens to satisfy the condition. The assertion keeps passing even when the implementation truncates, duplicates, mangles encoding, or injects extra metadata, so spec drift goes undetected.

**Default**: pin the entire output with `toMatchInlineSnapshot` / `toStrictEqual`.

- Notification messages, log formatting, template output → pin the full text with `toMatchInlineSnapshot`
- Length or count invariants on arrays → `toHaveLength` + pin contents with `toStrictEqual`
- To express "a prohibited token must not appear," pin the entire output with a snapshot so the snapshot breaks the moment the prohibited token appears

**When partial matching is genuinely necessary**: place a `toMatchInlineSnapshot` next to it in the same test case so the full output at that point is captured and visible. If the snapshot alone reads as evidence of "contained / not contained," you may add a supporting `includes` check. Partial-match assertions without an accompanying snapshot are sent back as Major in review.

### Applying the ESLint Preset

`createBaseEslintConfig()` from `@cortex/core-ecosystem-config/eslint` is wired into the root `eslint.config.js`, so every workspace receives the `vitest-strong-matchers` preset automatically via spread. **New workspaces do not need to import the preset individually** (any leftover individual imports become Phase B cleanup targets).

### No Legacy Ignore

The staging-migration list `.lint-legacy/weak-matchers.json` has been retired. The `vitest-strong-matchers` preset is always active, including for existing tests, and CI rejects reintroduction of the legacy ignore list or its generator script. When a weak matcher appears necessary, do not ignore — make the expected shape concrete and fix the test itself.

### Visualizing Adoption Progress

```bash
pnpm lint:vitest-matcher-adoption
pnpm lint:vitest-matcher-adoption -- --ci
```

`--ci` returns exit 1 when any package that has tests is not adopted. As long as the root `eslint.config.js` wires in `createBaseEslintConfig` / the preset, every workspace is judged adopted via root inheritance.

## Review Checks

Each item carries an explicit **severity** and **scope**. The auto-reviewer does not raise an item on a PR that falls outside its scope.

### Test Presence

| severity | scope | Check |
|---|---|---|
| Critical | every workspace with a test target | Coverage of 90% (statements AND branches) is met. Lowering the threshold is forbidden (any change that lowers `coverage.thresholds` in `vitest.config.ts` is sent back as Critical) |
| Major | PRs containing code changes | New code has tests (`*.test.ts` placed in the same directory as the implementation) |
| Minor | workspaces with a barrel `index.ts` | Even re-export-only `index.ts` has a test file (otherwise the pre-commit 0%-check catches it) |

### Test Quality

| severity | scope | Check |
|---|---|---|
| Major | all tests | Verifies externally observable behavior, not implementation details |
| Major | all tests | No flakiness-inducing factors (time, randomness, shared state, order dependence) |
| Major | all tests | Covers boundary values and error cases, not just the happy path |
| Minor | all tests | Structured so intent reads cleanly under AAA (Arrange-Act-Assert) or Given / When / Then |
| Minor | all tests | Mocks are kept to the minimum (excessive mocking couples tests to implementation details) |
| Minor | all tests | Test names clearly describe the expected behavior |

### Error Handling

| severity | scope | Check |
|---|---|---|
| Critical | all apps | `catch` blocks are not empty — always emit a log (prevent silent failure) |
| Major | all apps | Error messages are concrete and actionable |

### File Size and Structure

| severity | scope | Check |
|---|---|---|
| Major | all apps | Function arguments are 3 or fewer (use an object argument beyond that) |
| Minor | all apps | Files are 500 code lines or fewer |
| Minor | all apps | Early-return pattern keeps nesting shallow |

### Naming and Style

| severity | scope | Check |
|---|---|---|
| Critical | entire repository | `eslint-disable` / `oxlint-disable` are forbidden (canonical definition in [security.md — Lint / Conventions](./security.md)) |
| Minor | entire repository | Filenames: kebab-case; constants: UPPER_SNAKE_CASE; booleans: is/has/should prefix |
| Nit | entire repository | Comments are in Japanese |
