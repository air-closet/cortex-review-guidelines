# AI-Generated Code Antipatterns

Criteria for detecting problem patterns that appear constantly in code produced by AI coding assistants but are rarely seen in human-written code. Most cortex PRs are either AI-generated or AI-assisted, so this lens must be applied during every review.

Reference: adapted from the [ai-antipattern-reviewer persona / policy in nrslib/takt](https://github.com/nrslib/takt/blob/main/builtins/ja/facets/personas/ai-antipattern-reviewer.md), restructured for the cortex context.

## Scope of this lens

**Covered here:**

- Sanity-checking the assumptions AI has made
- Detecting hallucinated APIs and methods that do not exist
- Consistency with existing patterns in the codebase
- Scope creep / scope shrinkage (missing parts of the task requirements)
- Dead and unused code
- Overuse of fallbacks and default arguments
- Unneeded backward-compatibility code

**Delegated to other guidelines:**

- Architecture → [architecture.md](./architecture.md)
- Security → [security.md](./security.md)
- Product Graph integrity → [graph-integrity.md](./graph-integrity.md)
- Impact analysis → [impact-analysis.md](./impact-analysis.md)

## Stance

- AI generates code faster than humans can review it. Closing that quality gap is the reason this lens exists.
- AI is confidently wrong. Catch the code that looks plausible but does not work, and the solutions that are technically correct but contextually wrong.
- Trust, but verify. Catch the subtle issues that slip through a first pass.

## Review criteria

Each item declares a **severity** and a **scope**. The auto-reviewer does not fire an item on PRs that fall outside its scope. For severity classification, defer to [severity.md](./severity.md).

### Verifying assumptions

AI makes assumptions. Verify them.

| severity | scope | check |
|---|---|---|
| Major | All PRs | Does the implementation actually match the requested feature / requirement (is it answering a different question)? |
| Major | All PRs | Does it use a one-off pattern not found elsewhere in the codebase? |
| Minor | All PRs | Does it bring in an overly generic solution for a specific problem? |
| Major | All PRs | Does it correctly understand the business rules / domain constraints? |

Verification approach:

1. Use Product Graph (`search_product_graph_nodes`) to find existing implementations in the same area and confirm the naming and structural conventions.
2. When domain rules are visible in docs, comments, or test names, check that the implementation does not contradict them.
3. Be suspicious when the implementation reads like a "plausible generic solution" without any cortex-specific domain vocabulary.

### Plausible-but-wrong detection

| severity | scope | check |
|---|---|---|
| Critical | All PRs | Does it call a hallucinated API (a method that does not exist in the library / version in use)? |
| Critical | All PRs | Is there code that is syntactically correct but semantically wrong (e.g. validation that checks the format but misses the business rule)? |
| Major | All PRs | Does it use a deprecated pattern or stale API learned from training data? |
| Critical | All PRs | When a new parameter is added, are call sites actually passing a value (wiring not forgotten)? |
| Major | All PRs | Does error handling cover the realistic scenarios (i.e. not under-engineered)? |
| Minor | All PRs | Does it add abstraction layers not needed by the task (i.e. over-engineered)? |

Verification approach:

1. Confirm the code actually builds and tests pass (`pnpm build`, `pnpm test`).
2. Confirm that the imported functions / types exist in `node_modules`.
3. When a newly added option / argument is consumed as `options.xxx ?? fallback`, grep call sites to confirm a value is actually passed.
4. For library API argument shapes, consult the OpenAI / GCP known traps in [external-api-clients.md](./external-api-clients.md).

### Copy-paste pattern detection

AI tends to repeat the same pattern — mistakes and all.

| severity | scope | check |
|---|---|---|
| Major | All PRs | Are dangerous patterns repeated (the same vulnerability / the same trap in multiple places)? |
| Minor | All PRs | Is the same logic implemented in different ways across files? |
| Minor | All PRs | Has unnecessary boilerplate exploded where abstraction would have been possible? |

Combine with the semantic search for similar implementations described in [impact-analysis.md](./impact-analysis.md).

### Redundant conditional branching

AI tends to generate code that branches between calls to the same function differing only in their arguments.

| severity | scope | check |
|---|---|---|
| Minor | All PRs | Branching only on whether an argument is present, like `if (x) f(a, b, c) else f(a, b)`? |
| Minor | All PRs | Calling the same function in both branches with only an options-presence difference, like `if (x) f(a, {opt: x}) else f(a)`? |
| Minor | All PRs | A redundant else that ignores the return value, like `if (x) { f(a, x); return; } f(a);`? |

```typescript
// REJECT - Both branches call the same function; only the presence of the 3rd argument differs.
if (options.format !== undefined) {
  await processFile(input, output, { format: options.format });
} else {
  await processFile(input, output);
}

// OK - Unified via the ternary operator.
const formatOpt = options.format !== undefined ? { format: options.format } : undefined;
await processFile(input, output, formatOpt);
```

### Overusing callbacks + external-variable capture

AI tends to implement data that could be returned via a return value using callbacks that capture and assign to outer variables.

| severity | scope | check |
|---|---|---|
| Minor | All PRs | Assigning to an outer variable from inside a callback, like `let result; await f(x => { result = x })`? |
| Minor | All PRs | Synchronously obtaining a value through an event handler? |
| Minor | All PRs | Building a result by pushing into an outer Map / array from inside `forEach` (when `map` / `reduce` would do)? |

```typescript
// REJECT - Captures an outer variable from a callback.
let selectedMode: string | undefined;
await promptUser(choices, (choice) => {
  selectedMode = choice;
});
return selectedMode;

// OK - Receive via the return value.
const selectedMode = await promptUser(choices);
return selectedMode;
```

### Misresponding to review feedback

Instead of *fixing* what review feedback pointed out, AI sometimes "addresses" it by adding tests or documentation that *verify the feedback itself* — and feels done.

| severity | scope | check |
|---|---|---|
| Critical | Post-review fix commits | Does the commit actually touch the file / line called out in the review (rather than masking the issue with unrelated refactors)? |
| Major | Post-review fix commits | Do the added tests verify "the correct behavior after the fix" (rather than substituting "verification of the feedback itself")? |
| Major | Post-review fix commits | For feedback like DRY violations, is the response trying to close it by writing "this is intentional" in documentation? |

Also read the "Root-cause fix principle" in [review-guidelines.md](../review-guidelines.md).

### Contextual fit

Does this code belong in this particular project?

| severity | scope | check |
|---|---|---|
| Minor | All PRs | Does it match the existing codebase's naming conventions? |
| Minor | All PRs | Are error handling, logging, and test style consistent with existing patterns in the project? |
| Minor | All PRs | Are there unexplained deviations from project conventions? |

Questions to ask:

- Would a developer familiar with this codebase write it this way?
- Does it feel like it belongs here?

### Integration pattern consistency

Check whether similar API calls / data fetches / type definitions are implemented in *different* ways within the project.

| severity | scope | check |
|---|---|---|
| Major | All PRs | For code with the same purpose (REST calls, data fetches, type definitions), does it introduce an approach that differs from existing implementations? |
| Major | `apps/`, `packages/` | Does it hit `fetch` / `axios` directly when a shared client exists? |

Verification approach:

1. Identify how API calls / data fetches / type definitions in the diff are implemented.
2. Use Product Graph / grep to find how existing code for the same purpose is written.
3. Check first whether a shared package exists (`packages/domain/*`, `packages/external/*`, etc.).
4. If there is an inconsistency, flag the divergence and require alignment with the project's standard pattern.

### Scope creep / scope shrinkage

AI tends to both over-deliver and miss parts of the requirements.

| severity | scope | check |
|---|---|---|
| Major | All PRs | Are unrequested features added? |
| Minor | All PRs | Is there premature abstraction, such as an interface or abstraction with only one implementation? |
| Minor | All PRs | Is something made configurable that does not need to be (over-configuration)? |
| Minor | All PRs | Are there "nice-to-have" additions that nobody asked for (gold plating)? |
| Major | All PRs | Has it added legacy value mapping / normalization logic without an explicit instruction to do so? |
| Critical | All PRs | Is any of the requirements written in the issue / PR description left unimplemented (scope shrinkage)? |

Criteria for legacy handling:

- Unless there is an explicit instruction to "support legacy values" or "maintain backward compatibility", legacy handling is not needed.
- Do not add `.transform()` normalizations, `LEGACY_*_MAP`-style maps, or `@deprecated` type definitions.
- Support only the new values and keep things simple.

### Premature caching strategies

AI tends to introduce caching mechanisms preemptively, to "improve" performance.

| severity | scope | check |
|---|---|---|
| Major | All PRs | Has it added a caching layer (stale-while-revalidate, in-memory, Redis, local persistence) without an explicit request or measurement? |
| Minor | All PRs | Is memoization used heavily without an identified bottleneck? |
| Major | All PRs | Has it added homegrown TTL / cache-key management / purge logic? |

Decision rule: is there an explicit requirement for caching, or measurement data showing it is needed?

- YES → implementing it is fine
- NO → do not implement it. A naive fetch is enough

### Dead code detection

AI adds new code but often forgets to delete the old code it has replaced.

| severity | scope | check |
|---|---|---|
| Major | All PRs | After a refactor, are there leftover unused functions / methods / variables / constants? |
| Major | All PRs | Is there unreachable code after an early return, or branches whose condition is always true / always false? |
| Major | All PRs | Are there defensive branches that are **logically unreachable** given the caller's constraints? |
| Major | All PRs | Are there leftover import statements or package dependencies for removed functionality? |
| Major | All PRs | Has the implementation been removed but a re-export / index registration left behind? |
| Minor | All PRs | Is there commented-out code left in place? |

Example of logically dead code:

```typescript
// REJECT - The caller assumes interactive input, so the !isInteractive branch is unreachable.
function displayResult(data: ResultData): void {
  const isInteractive = process.stdin.isTTY === true;
  const output = isInteractive ? formatRich(data) : formatPlain(data);
}

// OK - Understand the caller's constraints and remove the unnecessary branch.
function displayResult(data: ResultData): void {
  logger.info(formatRich(data)); // Use @cortex/otel/logger.
}
```

Verification approach:

1. When you find a defensive branch, use Product Graph / grep to inspect every caller and confirm whether the condition is already satisfied.
2. Grep for any references to code that was modified or deleted.
3. Confirm that the exports of public modules (index files, etc.) match the implementations that actually exist.

### Overuse of fallbacks and default arguments

AI overuses fallbacks and default arguments to paper over uncertainty. Align this with the "Security and boundaries" principle in cortex's [review-guidelines.md](../review-guidelines.md).

| severity | scope | check |
|---|---|---|
| Critical | All PRs | Is there a fallback on data that must exist (e.g. `user?.id ?? 'unknown'`)? |
| Major | All PRs | Has a default argument become a de-facto constant because every caller omits it? |
| Major | All PRs | Is there a nullish-coalescing fallback like `options?.cwd ?? process.cwd()` with no upstream path to pass a value? |
| Critical | All PRs | Does a try-catch hide a real error behind an empty-value return (e.g. `catch { return ''; }`)? |
| Major | All PRs | Are there multi-level fallback chains like `a ?? b ?? c ?? d`? |
| Critical | All PRs | Does `if (!x) return;` silently swallow a case that should have been an error? |
| Critical | Config / env vars | Is a required setting hidden behind a fallback such as `baseUrl ?? ''`, an `?: string` optional, an empty string, same-origin, or a `null` default? (Make it required in the type and pin it with a throw at startup.) |
| Critical | All PRs | Does a `catch` block continue without logging the error (`.catch(() => null)` / `.catch(() => [])` / `catch (e) { /* ignore */ }`, etc.)? See "Error handling and log levels" in [observability.md](./observability.md) for details. |
| Major | All PRs | Does `logger.error(err.message)` discard `Error.stack` / `cause` and keep only the message string? (Structure it as `{ err: serializeError(error) }`.) |
| Major | All PRs | Does it downgrade the log level to warn based on the type name (e.g. `NotFoundError`), swallowing the absence of something that should always exist in business terms? |

Verification approach:

1. Grep the diff for `??`, `||`, `= defaultValue`, and `catch`.
2. For each fallback or default argument, check:
   - Is it required data? → REJECT
   - Does every caller omit it? → REJECT
   - Is there an upstream path that could pass a value? If not, REJECT
3. A single unjustified fallback or default argument is enough to REJECT.
4. Grep for `catch` and confirm that every block satisfies one of "logs the error", "rethrows", or "uses a business-approved fallback". For error-level classification, follow "Log level criteria" in [observability.md](./observability.md).

### Unused-code detection

AI tends to generate unnecessary code "for future extensibility", "for symmetry", or "just in case". Delete anything that is not called today.

| severity | scope | check |
|---|---|---|
| Major | All PRs | Is there a public function / method that is currently called from nowhere? |
| Major | All PRs | Is there a setter / getter built "for symmetry" but never used? |
| Major | All PRs | Is there an interface / option prepared for some future extension? |
| Major | All PRs | Is there an exported symbol with no usage findable via grep? |

Framework lifecycle hooks that are invoked implicitly are an exception.

### Unneeded backward-compatibility code

AI tends to leave unnecessary code in place "for backward compatibility". cortex **forbids backward compatibility as a default consideration**, and that needs to be enforced.

Backward-compatibility code to delete:

| severity | scope | check |
|---|---|---|
| Major | All PRs | Is there a `@deprecated` implementation with no usages remaining? (Delete immediately.) |
| Major | All PRs | Do a new API and an old API both exist, with the old API having no usages? |
| Major | All PRs | Is there a compatibility wrapper that has outlived a completed migration? |
| Major | All PRs | Are there deferred-cleanup comments like `// TODO: remove after migration` left lying around? |
| Minor | All PRs | Is there a Proxy / adapter that exists, and adds complexity, purely for backward compatibility? |

Decision rule:

1. Are there usages? → Check via Product Graph / grep. If none, delete.
2. Are there usages of both old and new? → If both are actively used today, it may be a coexistence design rather than backward compatibility. Inspect the callers.
3. Is it exposed externally? → cortex does not publish to npm, so deletion is essentially always safe.
4. Is there a legitimate reason such as an in-progress data migration? → Keep it, but only after filing a follow-up issue to delete it once the migration is done.

When AI claims "for backward compatibility", be skeptical. Confirm that it is actually needed.

### Decision traceability

Verify that non-obvious choices AI made are explained.

| severity | scope | check |
|---|---|---|
| Minor | All PRs | Are non-obvious design choices (library selection, pattern choice, trade-offs) explained in the PR description / commit message? |
| Minor | All PRs | Are the reasons technically sound? |
| Minor | All PRs | Are alternatives considered (or at least acknowledged)? |
| Minor | All PRs | Are assumptions explicit and reasonable? |

## Auto Review operations

- This guideline is referenced by the `ai-antipattern-reviewer` sub agent.
- Detection runs **on the diff**, not on a full repository scan.
- Findings include the target file, line number, problem, fix rationale, and the verification method (grep command / Product Graph query).
- Duplicate findings of the same pattern are deduplicated via `family_tag`.

## Related guidelines

- [severity.md](./severity.md) - Severity classification
- [architecture.md](./architecture.md) - Classifying architecture violations
- [impact-analysis.md](./impact-analysis.md) - Cross-cutting impact and missed-fix detection
- [graph-integrity.md](./graph-integrity.md) - Product Graph integrity
- [recurrence-prevention.md](./recurrence-prevention.md) - When to promote to a lint rule
