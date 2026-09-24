# Temporal migration playbook

## Contents

- Inventory
- Behavior capture
- Boundaries
- High-risk semantics
- Library migration
- Review checklist

## Inventory

Search for legacy construction, parsing, mutation, formatting, serialization, and library calls. Include patterns such as:

```text
new Date(          Date.now(          Date.parse(
.getTime(          .toISOString(      .toLocale
.getFullYear(      .getMonth(         .getDate(
.getUTC            .set               setUTC
moment(            dayjs(             DateTime.from
parseISO(          addDays(            differenceIn
```

Also inspect schemas, database mappers, API types, HTML date/time inputs, cron/scheduler code, caches, logs, tests, JSON fixtures, sorting, and equality checks.

For each occurrence record:

- semantic meaning and selected Temporal type;
- source and destination;
- assumed calendar and time zone;
- calendar arithmetic versus elapsed-time arithmetic;
- invalid-input policy;
- precision and rounding;
- serialized/storage shape;
- consumers that still require `Date` or another library.

## Behavior capture

Test existing behavior before refactoring. A useful matrix includes:

- ordinary valid input;
- empty, malformed, and ambiguous input;
- leap day and a non-leap year;
- January 31 and other month ends;
- DST spring gap and fall overlap for each relevant zone;
- negative durations and ordering;
- sub-millisecond precision where relevant;
- JSON/API output and database round trips;
- locale output only when localized strings are contractual.

Use a fixed clock in tests when “now” participates. Avoid snapshots that merely approve changed date text without explaining the semantic expectation.

## Boundaries

### API and JSON

Parse into Temporal at ingress and serialize explicitly at egress. Preserve existing contracts by default:

| Meaning | Typical stable representation |
|---|---|
| Instant | ISO string with offset, commonly `Z` |
| Plain date | `YYYY-MM-DD` |
| Plain date-time | ISO local date-time without an implied zone |
| Zoned occurrence | RFC 9557-style string with bracketed zone, or separate instant and zone fields |
| Duration | ISO 8601 duration string |

Do not assume `toString()` options match an existing contract. Test exact output, including fractional seconds and annotations.

### Database

Classify columns semantically, not by their current SQL type. `created_at` is commonly an instant; `birth_date` and `due_date` are commonly plain dates. A date stored as midnight UTC may be a workaround, not the intended domain model.

Do not change database types, offsets, precision, or ORM mappings casually. Preserve storage during the internal refactor, then make schema migration a separate explicit change when needed.

### UI

- Map `<input type="date">` to `PlainDate`.
- Map `<input type="time">` to `PlainTime`.
- Map `<input type="datetime-local">` to `PlainDateTime`, then combine it with an explicitly chosen zone if it represents a real occurrence.
- Format for users with `Intl.DateTimeFormat`; retain the zone in the formatter options.

### Legacy consumers

Convert `Date` to an instant through `date.toISOString()` or epoch milliseconds. Convert an instant to `Date` with `new Date(instant.epochMilliseconds)`. Document loss of nanosecond precision. Never pass a date-only string through `new Date(...)` as a parsing shortcut.

When permissive legacy parsing is contractual but cannot yet be specified,
retain `Date` parsing at ingress and convert the validated result immediately:

```ts
const milliseconds = new Date(input).getTime();
if (Number.isNaN(milliseconds)) throw new RangeError("Invalid date/time");
const instant = Temporal.Instant.fromEpochMilliseconds(milliseconds);
```

Mark this as compatibility debt. Do not claim the parsing migration is complete,
because implementation-dependent or host-local input semantics still remain.

## High-risk semantics

### Calendar versus elapsed time

`zoned.add({ days: 1 })` means the next calendar day and may span 23 or 25 hours. `instant.add({ hours: 24 })` means exactly 24 elapsed hours. `Instant` rejects date units because they require calendar and zone context.

### Month ends and leap days

Temporal arithmetic commonly constrains overflow. Verify whether the product wants constrain, reject, “last day of month,” or another recurrence rule. Preserve accidental legacy overflow only if it is confirmed behavior.

### DST gaps and overlaps

When resolving a local date-time in a zone, choose a disambiguation policy deliberately: `compatible`, `earlier`, `later`, or `reject`. The default `compatible` often resembles legacy behavior, but defaulting is still a product decision for scheduling systems.

### Mutation

Temporal is immutable. Replace mutating setters with assigned return values, and inspect aliases/callers that relied on the original `Date` object being changed.

### Parsing

Reject ambiguous locale strings unless the project has an explicit parser and locale contract. An offset is not a named time zone. A plain date-time does not identify an instant.

## Library migration

Do not remove Moment, Day.js, Luxon, or date-fns by translating method names one by one. First map each library object and operation to domain semantics. Keep specialized formatting, recurrence, business-calendar, or interval functionality until Temporal plus project code genuinely replaces it.

Migrate in vertical slices:

1. shared Temporal adapter;
2. pure utilities with regression tests;
3. domain logic;
4. validation and mapping;
5. persistence;
6. API/UI boundaries;
7. dependency removal after usage and bundle verification.

## Review checklist

- Were all values classified by meaning?
- Did any date-only value become an instant, or vice versa?
- Is every behavior-sensitive zone explicit?
- Are DST disambiguation and offset rules deliberate?
- Are calendar and elapsed-time arithmetic distinguished?
- Were mutation side effects removed safely?
- Are leap day and month-end rules tested?
- Are API, JSON, database, and UI formats unchanged or intentionally versioned?
- Are fixed fractional-second digits and `Z`/offset formatting preserved exactly?
- Is the runtime/polyfill strategy compatible with every target?
- Does third-party `Date` interop remain only at boundaries?
- Do tests compare old and new behavior, including error behavior?
