# Skills

A collection of reusable Agent Skills. Each skill lives in its own directory under `skills/` and can be installed independently.

## Available skills

### `temporal-date-api`

Safely design, review, and migrate JavaScript and TypeScript date/time code to the ECMAScript Temporal API while preserving observable behavior and external contracts.

The skill covers semantic type selection, runtime and polyfill strategy, DST behavior, serialization boundaries, regression testing, and migration from legacy `Date`, Moment, Day.js, Luxon, or date-fns code.

## Install

Install the skill with the package runner you already use:

```sh
# npm
npx skills add Sensei85/skills --skill temporal-date-api

# Bun
bunx skills add Sensei85/skills --skill temporal-date-api

# pnpm
pnpm dlx skills add Sensei85/skills --skill temporal-date-api
```

You can inspect the available skills without installing:

```sh
bunx skills add Sensei85/skills --list
```

## What it helps with

- Classifying values as instants, civil dates, wall-clock times, zoned date-times, or durations.
- Preserving API, database, UI, and serialization contracts during migration.
- Making DST gaps, overlaps, calendar arithmetic, and time-zone rules explicit.
- Choosing between native Temporal and a maintained polyfill based on actual runtime support.
- Designing regression coverage for leap days, month ends, ordering, invalid input, and precision.

## Repository structure

```text
skills/
└── temporal-date-api/
    ├── SKILL.md
    ├── agents/
    │   └── openai.yaml
    └── references/
        ├── migration-playbook.md
        ├── recipes.md
        └── runtime-and-polyfill.md
```

## License

MIT
