---
name: temporal-date-api
description: Safely design, review, or refactor JavaScript and TypeScript date/time code to the ECMAScript Temporal API while preserving product behavior and external contracts. Use for legacy Date migration; Moment, Day.js, Luxon, or date-fns replacement; Temporal type selection; time-zone or DST bugs; date arithmetic, parsing, formatting, serialization, database/API/UI boundaries; runtime support assessment; and native-versus-polyfill setup with @js-temporal/polyfill or another maintained implementation.
license: MIT
metadata:
  version: "1.0.0"
---

# Temporal Date API

Refactor date/time handling by meaning, not syntax. Preserve observable behavior unless the user explicitly authorizes a contract or business-rule change.

## Start with semantics

For every value, write down what it represents before editing code:

| Meaning | Temporal type |
|---|---|
| Exact point on the timeline | `Temporal.Instant` |
| Date and time in a named zone | `Temporal.ZonedDateTime` |
| Calendar date only | `Temporal.PlainDate` |
| Wall-clock date and time without a zone | `Temporal.PlainDateTime` |
| Time of day only | `Temporal.PlainTime` |
| Year and month only | `Temporal.PlainYearMonth` |
| Month and day only | `Temporal.PlainMonthDay` |
| Amount of time | `Temporal.Duration` |

Do not infer semantics solely from a variable name, a database type, or a midnight timestamp. Trace producers, consumers, storage, and display behavior.

## Follow the migration workflow

1. Inspect the target runtimes, package manager, TypeScript configuration, date libraries, storage schema, API contracts, and tests.
2. Inventory date/time creation, parsing, getters/setters, arithmetic, comparison, formatting, serialization, persistence, and test fixtures.
3. Classify each value using the table above. Flag unresolved product rules instead of guessing.
4. Capture current behavior with regression tests before changing implementation. Include invalid input, leap day, month end, serialization, ordering, and DST transitions when zones are involved.
5. Choose a runtime strategy using [runtime-and-polyfill.md](references/runtime-and-polyfill.md). Never assume native support merely because Temporal is standardized.
6. Introduce one project-owned Temporal adapter or import policy. Avoid ad hoc imports and multiple Temporal implementations within one execution path.
7. Migrate the smallest pure internal path first: parsing/creation, arithmetic, comparison, then formatting.
8. Keep database, API, UI, and third-party boundaries stable. Convert at the edge when a consumer still requires `Date`.
9. Run focused tests after each slice, then the broader relevant suite and type checker.
10. Report selected semantics, edge cases, intentional differences, polyfill/runtime decisions, and remaining legacy interop.

If a tiny snippet has no test harness, do not build a large scaffold by default.
Create the smallest runnable characterization check supported by the existing
runtime, or record an explicit input/output behavior table and state what could
not be executed.

For detailed inventory patterns, sequencing, boundary rules, and review checks, read [migration-playbook.md](references/migration-playbook.md). For concrete conversions and arithmetic examples, read [recipes.md](references/recipes.md).

## Enforce semantic guardrails

- Distinguish a calendar day from 24 elapsed hours.
- Require an explicit IANA time-zone identifier when product behavior depends on place. Use the host/device zone only when that is the stated rule.
- Treat `Temporal` values as immutable; assign returned values and find callers that depended on `Date` mutation.
- Preserve ISO/wire/database shapes unless an intentional migration changes them, including fixed fractional-second digits such as the three milliseconds emitted by `Date#toISOString()`.
- Parse date-only, local date-time, offset date-time, and zoned date-time strings with the matching type.
- Do not use `Instant` for birthdays, due dates, or other date-only values.
- Do not use `PlainDateTime` alone for a real scheduled occurrence; retain the named zone or resolve it deliberately to an instant.
- Make DST disambiguation policy explicit when local times can be skipped or repeated.
- Recognize that `Instant` arithmetic does not accept calendar units such as days, weeks, months, or years; use exact time units or convert through a zoned/calendar-aware type.
- Keep legacy `Date` interop where a platform or library requires it, and document millisecond precision loss.

## Challenge ambiguous requirements

Pause before implementing when any answer changes behavior. Ask focused questions such as:

- Is this value an exact moment, a civil date, or a wall-clock entry waiting for a zone?
- For February 29 or month-end recurrence, should the result constrain, reject, or follow a custom rule?
- During a DST gap or overlap, should scheduling choose the earlier instant, later instant, legacy-compatible behavior, or reject?
- Does “one day” mean the next calendar day or exactly 24 hours?
- Which serialized and database formats are contractual?

Do not let a mechanical migration silently decide these product rules.

## Choose polyfill behavior deliberately

Prefer native `globalThis.Temporal` only when every supported runtime supplies the required API and tests exercise that path. Otherwise, use a maintained polyfill consistently through a project adapter. Static fallback imports are simple but usually keep the polyfill in the bundle; dynamic loading can reduce modern bundles but introduces asynchronous startup and build complexity. Choose based on the actual deployment matrix, not fashion.

Do not patch `globalThis` or `Date.prototype` unless the project explicitly needs global compatibility and the selected package documents that setup. Prefer module-local exports.

## Complete with evidence

Return a concise summary containing:

- values classified and Temporal types selected;
- runtime/polyfill strategy and why;
- unchanged external contracts;
- tests and edge cases exercised;
- intentional behavior differences;
- remaining `Date` or library interop and why it remains.
