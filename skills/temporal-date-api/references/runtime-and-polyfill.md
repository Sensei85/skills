# Runtime and polyfill strategy

## Contents

- Decision process
- Adapter patterns
- TypeScript and build concerns
- Removal checklist
- Authoritative sources

## Decision process

Temporal is standardized, but runtime rollout is uneven. Never encode a permanent browser or Node support table in application logic. Inspect the project's declared engines, browser targets, edge/server runtimes, test runner, and embedded webviews, then verify current support against authoritative implementation data.

Choose one of three strategies:

1. **Native only**: use `globalThis.Temporal` when all supported runtimes implement the required API.
2. **Polyfill only**: import one implementation and use it consistently. This is predictable for mixed or old runtimes.
3. **Native with fallback**: expose native Temporal when present and a polyfill otherwise. This can avoid semantic drift between implementations but a static import may still add bundle cost.

Do not confuse standardization with availability. Do not remove a polyfill until the minimum supported runtime, tests, build tools, and deployment environments all pass without it.

## Adapter patterns

Prefer a project-owned module so call sites do not care where the implementation comes from.
For a one-file utility with one guaranteed implementation and no likely expansion,
a direct package import is sufficient. Introduce an adapter for applications,
multi-file migrations, libraries, or native/fallback runtime strategies.

### Polyfill-only module

Use this when any supported environment lacks native Temporal and deterministic behavior matters more than native-path testing:

```ts
export { Temporal } from "@js-temporal/polyfill";
```

Call sites:

```ts
import { Temporal } from "./temporal";
```

### Native with static fallback

Use only after checking the package's current API and project typings:

```ts
import { Temporal as PolyfilledTemporal } from "@js-temporal/polyfill";

export const TemporalApi =
  globalThis.Temporal ?? PolyfilledTemporal;
```

This is operationally simple. It does not guarantee that bundlers omit the polyfill from modern builds.

Avoid claiming `typeof Temporal` is always a portable annotation: it depends on the TypeScript version and configured library definitions. Prefer inferred types or a project type alias compatible with the chosen package.

### Conditional loading

Consider conditional dynamic import only at an async application entry point. Confirm that no module reads Temporal before initialization and that the bundler produces the intended chunks. Do not scatter asynchronous fallback logic across domain modules.

### Global installation

The `@js-temporal/polyfill` package historically exports `Temporal` without installing `globalThis.Temporal`. Treat global patching as an explicit compatibility choice, not a default. Do not mutate `Date.prototype` merely to migrate internal code.

## TypeScript and build concerns

- Verify the project's TypeScript version recognizes native Temporal types; runtime support and compile-time types are separate concerns.
- Check ESM/CJS compatibility, transpilation targets, tree shaking, server rendering, workers, tests, and bundle size.
- Lock and review the chosen package version using the project's normal dependency policy.
- Run representative parsing, DST, rounding, and serialization tests against every implementation path the app can execute.
- Avoid passing objects created by one Temporal implementation to another. Prefer strings or primitive boundary values if implementations must meet.
- Verify formatter support. Bridge an instant through epoch milliseconds or `Date` when an older `Intl` or third-party formatter cannot consume Temporal directly.

## Removal checklist

Before removing a polyfill:

- confirm every supported runtime exposes the required Temporal surface;
- confirm CI, tests, SSR, build tools, workers, and local development use supported runtimes;
- remove the dependency and adapter fallback in one controlled change;
- run type checking and the complete relevant date/time regression suite;
- compare serialization and zone behavior before and after;
- retain boundary conversions for third-party APIs that still require `Date`.

## Authoritative sources

- Temporal specification and documentation: https://tc39.es/proposal-temporal/
- TC39 proposal repository and implementation status: https://github.com/tc39/proposal-temporal
- `@js-temporal/polyfill` usage and compatibility: https://github.com/js-temporal/temporal-polyfill

Recheck these sources when runtime support or package selection affects the change. As of August 2026, Temporal is Stage 4, but support remains deployment-matrix dependent.
