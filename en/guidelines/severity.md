# Severity Classification and Decision Rules

## Severity Levels

| Severity | Criteria | Action |
|----------|----------|--------|
| **Critical** | Security vulnerabilities, risk of data destruction, risk of a production incident, OOM risk, **documentation drift** (missing supporting docs, docs not updated when behavior is added or changed, divergence between existing docs and implementation), **`@graph-*` JSDoc mismatch or absence**, **lowering of test coverage thresholds** (changes that reduce the coverage thresholds in `vitest.config.ts` are forbidden; add tests to meet the bar instead of lowering it), **quality-standard loosening** (any change that weakens guideline documents, lint rules, coverage thresholds, or similar quality standards; the AI reviewer must not approve a PR that contains such a change and must return `REQUEST_CHANGES`; approval by a human reviewer is required) | `REQUEST_CHANGES` |
| **Major** | Violations of the Composable Architecture principles, missing tests, severe performance issues | `REQUEST_CHANGES` |
| **Minor** | Naming improvements, missing comments, small refactor suggestions, suggestions to extract into a package | `REQUEST_CHANGES` |
| **Nit** | Style preferences, micro-improvements | `APPROVE` (raised as a comment) |

## No-Demotion Rule

A finding must not be demoted to Nit for any of the following reasons. The PR must address the root cause.

- **No demotion on the grounds of "following an existing pattern"**: If the existing code violates the guideline, new code that follows it must still be raised at the same severity. Comments that defer the fix ("we'll consider this in the next refactor") are not accepted.
- **No quality-standard loosening on the grounds that "the existing implementation already violates it, so let's align the bar"**: The fact that existing code already violates a rule is not justification for relaxing the guideline, the lint rule, or the threshold. The existing violation is a separate problem that must be fixed on its own; the standard itself must not be lowered to make it go away.
- **No demotion on the grounds of "to be handled in a separate PR", "next session", "out of scope", or "incrementally"**: Critical / Major / Minor findings must not be downgraded to Nit on the assumption that they will be deferred or only partially fixed. When the author tries to close a thread with such reasoning, the reviewer must return `REQUEST_CHANGES`.
- **No demotion on the grounds of leaving a TODO / FIXME**: Writing the outstanding work into a `TODO` / `FIXME` comment in the code does not reduce the severity. Review findings must be fixed at the root cause within the PR.

Example: When a function takes more than three positional arguments (it should accept an object argument), the finding is raised as **Minor** (`REQUEST_CHANGES`) even if existing functions exhibit the same violation. It is not a Nit. Likewise, if the root cause is an architecture violation, do not demote the finding even when the author proposes a surface-level patch with "we'll address the root cause in a separate PR" — keep it at `REQUEST_CHANGES`.

This document does not enumerate exception conditions for demotion. Exceptions are judged case by case, and any agreement reached must be made explicit on the PR.

## Merge Conditions

The PR is only mergeable when every review comment has been resolved. If the review is `APPROVE` but there are unresolved comments, the merge is held.

## Review Comment Policy

Report only problems and improvements. Positive-only feedback ("LGTM", "great design", and similar) is not needed.
