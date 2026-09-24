# Temporal refactor recipes

Assume `Temporal` is imported from the project's central adapter.

## Contents

- Create values
- Convert legacy Date and Instant
- Date-only and exact arithmetic
- Differences, zones, comparison, and formatting
- Serialization
- Example migration

## Create values

```ts
const now = Temporal.Now.instant();
const today = Temporal.Now.plainDateISO("Africa/Accra");
const dueDate = Temporal.PlainDate.from("2026-06-25");
const localInput = Temporal.PlainDateTime.from("2026-06-25T09:00");
const meeting = Temporal.ZonedDateTime.from(
  "2026-06-25T09:00:00+00:00[Africa/Accra]",
);
```

Use a bracketed named zone for scheduled civil time. Include an offset when the contract needs to validate the intended occurrence.

## Convert legacy Date and Instant

```ts
export function dateToInstant(date: Date) {
  return Temporal.Instant.fromEpochMilliseconds(date.getTime());
}

export function instantToDate(instant: Temporal.Instant) {
  return new Date(instant.epochMilliseconds);
}
```

The second conversion loses precision below milliseconds.

## Date-only calendar arithmetic

```ts
const renewal = Temporal.PlainDate
  .from("2024-02-29")
  .add({ years: 1 }); // 2025-02-28 with constrain overflow

const nextBusinessDate = Temporal.PlainDate
  .from("2026-01-31")
  .add({ months: 1 }); // verify product's month-end rule
```

To reject invalid overflow where supported by the operation, pass an explicit overflow option and test it. Do not assume constrain behavior matches legacy setter overflow.

## Exact elapsed-time arithmetic

```ts
const expiresAt = Temporal.Now.instant().add({ hours: 24 });

const elapsed = Temporal.Instant
  .from("2026-06-25T10:00:00Z")
  .until("2026-06-25T12:30:00Z", {
    largestUnit: "hours",
    smallestUnit: "minutes",
  });
```

Do not add days, weeks, months, or years to an `Instant`; those units need a calendar and often a zone.

## Calendar difference

```ts
const days = Temporal.PlainDate
  .from("2026-06-25")
  .until("2026-07-05", { largestUnit: "days" })
  .days;
```

For a multi-unit duration, do not read only one field and assume it is the total. Choose `largestUnit`, `smallestUnit`, rounding, and `relativeTo` deliberately.

## Resolve a local time in a zone

```ts
const local = Temporal.PlainDateTime.from("2026-11-01T01:30");
const occurrence = local.toZonedDateTime("America/New_York", {
  disambiguation: "reject",
});
```

Choose `compatible`, `earlier`, `later`, or `reject` based on product behavior. Add tests using a zone whose transition actually occurs on the selected date.

## Convert an instant for local display or date extraction

```ts
const localDate = Temporal.Instant
  .from("2026-06-25T23:30:00Z")
  .toZonedDateTimeISO("Africa/Accra")
  .toPlainDate();
```

## Compare and sort

```ts
const isBefore = Temporal.PlainDate.compare(a, b) < 0;
const sorted = [...instants].sort(Temporal.Instant.compare);
```

Copy before sorting when callers expect immutability of the collection as well as its values.

## Format explicitly

```ts
const formatter = new Intl.DateTimeFormat("en-GH", {
  dateStyle: "medium",
  timeStyle: "short",
  timeZone: "Africa/Accra",
});

const text = formatter.format(
  new Date(instant.epochMilliseconds),
);
```

Use a `Date` bridge only when the target formatter/runtime cannot accept Temporal directly. Test locale output only when exact text is contractual; otherwise test semantic parts.

## Preserve common wire shapes

```ts
const timestamp: string = instant.toString();
const dateToISOStringCompatible: string = instant.toString({
  fractionalSecondDigits: 3,
});
const dateOnly: string = plainDate.toString();
const duration: string = temporalDuration.toString();
```

`Date#toISOString()` always emits three fractional-second digits. Pin exact
formatting options when compatibility, fractional precision, offsets, or
annotations matter. Never change a public wire format merely because another
Temporal string is richer.

## Example migration

Before:

```ts
export function getRenewalDate(startDate: string) {
  const date = new Date(`${startDate}T00:00:00Z`);
  date.setUTCFullYear(date.getUTCFullYear() + 1);
  return date.toISOString().slice(0, 10);
}
```

The legacy setter rolls `2024-02-29` to `2025-03-01`; the direct Temporal
replacement below constrains it to `2025-02-28`. Use this version only after
the product owner intentionally selects constrained anniversary behavior:

```ts
import { Temporal } from "./temporal";

export function getRenewalDate(startDate: string) {
  return Temporal.PlainDate.from(startDate).add({ years: 1 }).toString();
}
```

If preserving the setter's rollover is required, encode and name that rule
explicitly instead of relying on overflow. Test the ordinary case, February 29,
invalid input, and the agreed recurrence rule. Preserve the `YYYY-MM-DD` return
contract.
