# Observability and Notification-Message Guidelines

cortex **prohibits truncation and omission** in logs, Slack notifications, alert bodies, and operational messages, and **requires that errors always emit structured logs at an appropriate level**.
Observability is the foundation for detecting incidents, degradations, and threshold breaches, and for the first round of triage. Cutting off message bodies or silently swallowing errors makes it impossible to know "how many of these happened," forcing root-cause work to be redone via a different channel. The bar is that a recipient of a notification or log should be able to complete first response from that message alone.

This guideline covers two angles:

1. **No truncation of notification or log bodies** (emit everything, or split delivery)
2. **Error handling and log levels** (no silent catches; emit at the right level)

---

## 1. No Truncation of Notification or Log Bodies

### Prohibited Patterns

The following ways of reducing the information density of notifications or logs are prohibited.

- Cutting off arrays with `slice(0, N)` / `take(N)` / `head(N)`
- Appending an omission suffix like "...and N more" and truncating
- Shrinking content by limiting `JSON.stringify` depth or imposing a `length` cap
- Dropping the tail of content with `String#substring` / `String#slice`
- Consolidating `console.log` / logger output outside a loop into a single line that reports only counts

### Principles to Follow

1. **Emit everything**: Breaches, failures, and detected errors must be listed in full regardless of count. It's hard for humans to read at scale, but machines need it to grep and copy.
2. **Split delivery when necessary**: For notification destinations with a message-length cap (Slack, Discord, etc.), **split into multiple messages and deliver the full set** rather than omitting (loop `chat.postMessage`, thread replies, etc.).
3. **Prefer structured logs**: Emit bulk data as structured fields (`logger.info({ items: [...] }, 'message')`) rather than text. Observability tools aggregate and filter as needed.
4. **Summaries augment, never replace**: `count`, `min`, `max` summaries may be emitted alongside the full list, before or after it. A summary must not substitute for the full list.

### Exceptions

- **Masking PII / sensitive data**: Fields containing personal information, credentials, or PII must be masked. This is "secret protection," not "information reduction," and falls outside this guideline. Keep the structure intact and substitute the value with a placeholder (`***`, etc.).
- **Working around physical limits**: When something technically cannot fit (Slack's 40k character text limit, Cloud Logging's 256KB payload limit, etc.), split delivery. If you must truncate, mark the cut explicitly (e.g. `<<truncated: see ...>>`) and provide the full reference (GCS / BQ / Logs Explorer URL, etc.) so a human can still complete first response.

---

## 2. Error Handling and Log Levels

When an error happens, **do not paper over it with a hasty fallback or silent swallow — emit a structured log at the level that matches its meaning**.
AI-generated code has a strong tendency to "just make it run" by catching and returning empty values, null, or simply swallowing — and the result is production incidents that degrade unobserved.

### Prohibited Patterns

| severity | Check |
|---|---|
| **Critical** | Silently swallowing errors by returning success values without emitting error info, e.g. `catch (err) {}` / `catch { return null; }` / `catch { return []; }` / `.catch(() => undefined)` |
| **Critical** | Silently skipping with `if (!result) return;` when a missing value should actually be an error |
| **Critical** | Continuing past a missing required resource (DB record, configuration, dependency response) with only a warn-or-below log |
| **Major** | Dropping `Error.stack` / `cause` and keeping only a string, e.g. `logger.error(err.message)` |
| **Major** | Emitting fixed strings without context (target ID, parameters, stage), e.g. `logger.error('Failed')` |
| **Major** | Catching and printing via `console.error` / `console.log` (raw console instead of the Pino logger) |
| **Minor** | Marking the catch block with `// ignore` / `// best-effort` and similar comments without explaining in the PR description or a comment why ignoring is genuinely acceptable |

### How to Decide Log Level

Log level is not determined by "is it an `Error` type?" or "was it expected?" Decide based on **what this error means for this feature**.
Do not downgrade to warn just because the type name is `NotFoundError` or because the label says "expected."
For instance, "a record that must absolutely exist is missing" is error or fatal, not warn.

| Level | Criterion | Examples |
|---|---|---|
| **fatal** | The feature is expected to fail at **≥20%**. Service continuity is impossible, a critical configuration is missing, or a dependency is fully down | OTel initialization failure / required secret missing at boot / pipeline input data source fully down |
| **error** | **Data recovery or rerun will definitely be required**. Expected impact is under 20%. Loss of data that should exist, or failure of an external API call that should have succeeded | "A user record that should exist is missing" / BigQuery insert failure / per-record enrichment failure |
| **warn** | **Foreseeable in normal operation and not immediately problematic**. Transient errors expected to self-heal via retry, user-induced validation failures, anticipated empty results | Zero hits on a search query / optional field unset / short retries due to rate limiting |
| **info** | Normal operational events (start / completion / counts) | Batch completion summary / request handling logs |
| **debug** | Detailed traces for development. Do not emit continuously in production | Query bodies / intermediate state |

Decision flow:

1. Does this error require **later data recovery**? YES → error or higher
2. Could this error occur **broadly enough to breach the feature's SLO**? YES → fatal
3. Is it **anticipated and expected to self-heal via retry or next run**? YES → warn
4. Otherwise — no recovery needed and execution can continue → info / debug

Type-name-driven downgrades like "`NotFoundError`, so warn" or "`ValidationError`, so warn" are prohibited. The same `NotFoundError` can be error or fatal when "something that must exist is missing," and warn when "user search returned zero hits."

### Minimum Requirements for Structured Logs

When you catch and log, use the structured Pino logger from `@cortex/otel/logger` and meet the following requirements.

```typescript
import { createLogger, serializeError } from '@cortex/otel/logger';

const logger = createLogger('service-name');

try {
  await doSomething(targetId);
} catch (error) {
  logger.error(
    {
      err: serializeError(error),
      event: 'do_something_failed',
      targetId,
      stage: 'enrichment',
    },
    'Failed to enrich target',
  );
  throw error; // または業務上の代替値で継続する場合は理由を明記する
}
```

Required:

- **`err` field carrying `serializeError(error)`**: keeps `name` / `message` / `stack` as structured fields. Do not log only `err.message` and drop the stack.
- **Context fields**: attach the target ID, the stage, and related parameters as structured fields. String concatenation like `'Failed: ' + JSON.stringify(...)` is not acceptable (it cannot be aggregated via grep or Loki queries).
- **`event` field**: a fixed event name (snake_case recommended) so downstream Loki / Grafana queries can aggregate.
- **Fixed message string**: do not embed dynamic values into the message body (use `'Failed to enrich user'` + `{ userId }` instead of `Failed to enrich user ${userId}`).

### Rethrow and Delegating Logging Upstream

If the catch block **rethrows the same error, no log is needed** (the upstream handler or `process.on('uncaughtException')` at startup is expected to emit it). One of the following must hold:

1. **Rethrow the same error**: equivalent to not catching at all, so no extra log is required
2. **Wrap and throw a new error**: preserve the original error via `cause` (`throw new EnrichmentError('...', { cause: error })`) so the upstream does not lose the stack
3. **Complete handling at the catch site**: if you do not propagate upward, emit a log at the appropriate level right then

Doing nothing downstream on the assumption that "something upstream will catch it" is prohibited. The fact that an upstream log is guaranteed must be demonstrable in the code or the PR description.

### When Continuing with a Fallback Value

Returning an alternate value on error and continuing is permitted **only when all of the following are met**:

1. Business semantics explicitly require continuation via fallback (documented in the spec or issue)
2. The fallback occurrence **is logged at warn or higher** (not silent)
3. Fallback frequency is observable in Grafana / Slack, or the PR judges whether additional monitoring is needed
4. **Downstream data integrity** under the fallback value has been verified in the PR

"Just return an empty array" or "just return null" without a log is treated as a Critical violation.

### Review Checks (Error Handling)

- The `catch` block either emits a log or rethrows
- Log level is decided by "what this error means for the feature," not by type name
- The `err` field carries the result of `serializeError(error)` (not `.message` alone)
- `event` and context fields (target ID, stage) are structured
- A missing required resource is not downgraded to warn or below
- No swallowing like `.catch(() => undefined)` / `.catch(() => [])` / `.catch(() => null)`
- `if (!result) return;` is not silently skipping what should be an error- or fatal-level miss

---

## Review Checks (Truncation)

- Notification or log bodies do not contain "and N more," "...", "truncated," or "omitted."
- Loops do not impose `slice` / `take` / count caps.
- The PR verifies what happens when notification volume grows — does it split into multiple messages, or does it get cut off?
- When truncating is unavoidable, the PR description explicitly cites the "physical limit" exception.

## CI Guards

`scripts/check-observability-truncation.ts` detects unit-agnostic omission suffixes such as `...他${N}` / `…他${N}` / `(他 ${N})` / `他 ${N} 枚` and fails the guards job in `.github/workflows/test.yml`.

Silent catches in error handling are detected mechanically by the `graph/no-silent-catch` rule (`error` level) in `packages/eslint-plugin-graph/`. It flags patterns like `catch (...) {}` and `.catch(() => <literal>)` and enforces either a structured log via `@cortex/otel/logger` or a rethrow.

---

## Grafana Loki Alert Query Patterns

Patterns to follow when adding or modifying Loki alert rules in `infra/observability/grafana-alert-rules.ts`.

### Do Not Use `|= "error"` Text Filters

Using a `|= "error"` text filter in a Loki query causes **non-error logs that happen to contain the word "error"** to be falsely flagged.

**Prohibited example**:

```logql
count_over_time({service_name=~"..."} |= "error" [5m])
```

Real false-positive patterns observed:
- CSS class names (`ux-mirror`'s `.error-message`)
- TypeScript compiler output (`"0 errors"` / `"Found 1 error"`)
- Successful-completion messages that happen to contain "error" (`"total_errors: 0"`)

### Use the `detected_level` Stream Label

Across all services using `@cortex/otel` + Pino + OpenTelemetry SDK, Grafana Cloud's OTel ingest pipeline derives `detected_level` from `severity_number` and attaches it as an indexed **stream label** (not structured metadata).

**Recommended: use it inside the stream selector (index-efficient)**:

```logql
count_over_time({service_name=~"...", detected_level="error"} [5m])
```

**Alternative: pipeline filter also works**:

```logql
count_over_time({service_name=~"..."} | detected_level="error" [5m])
```

Because `detected_level` is a stream label, the stream-selector form `{..., detected_level="error"}` is valid and allows fast, index-backed filtering. The `| detected_level` pipeline-filter form also works against stream labels; `errorSeverityCount` uses it to catch "Logic GCF failed"-style errors whose log message doesn't contain the word "error" (both forms are valid).

### Tests: Pin the Entire LogQL Query with `toMatchInlineSnapshot`

For helper functions that generate alert queries, pin the expected query string in full with `toMatchInlineSnapshot` so unintended future changes are caught in CI. Partial-match-only assertions such as `toContain` / `not.toContain` are forbidden (see `docs/guidelines/testing.md`).

## References

- Severity criteria: [severity.md](./severity.md)
- Recurrence prevention: [recurrence-prevention.md](./recurrence-prevention.md)
- AI antipatterns (fallback overuse): [ai-antipattern.md](./ai-antipattern.md)
- Fail-fast at security boundaries: [security.md](./security.md)
