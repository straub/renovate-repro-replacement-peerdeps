# replacementName silently fails for npm peerDependencies

Minimal reproduction for a bug where `replacementName`/`replacementVersion` rules in Renovate fail silently when the target package appears in `peerDependencies`, in two distinct ways depending on the declared version range.

## Current behavior

This repo has a `package.json` with `peerDependencies` declaring two packages, each exposing a different failure mode:

| Package | Range | What Renovate does |
|---|---|---|
| `istanbul-instrumenter-loader` | `"*"` | No replacement PR created, no [Dependency Dashboard](https://github.com/straub/renovate-repro-replacement-peerdeps/issues/2) entry, no log warning |
| `request` | `">=2.0.0"` | `package.json` is unchanged ("No package files need updating") |

## Expected behavior

In both cases, Renovate should create a PR that renames the package in `peerDependencies`:

- `istanbul-instrumenter-loader: "*"` → `coverage-istanbul-loader: "*"` (or `">=3.0.5"`)
- `request: ">=2.0.0"` → `got: ">=14.0.0"`

## Root cause

**Bug 1 (`"*"` — update silently dropped)**:
In `lib/workers/repository/process/lookup/utils.ts`, `determineNewReplacementValue()` calls `versioningApi.getNewValue()`. In `lib/modules/versioning/npm/range.ts`, `getNewValue` returns `null` for any X-range (`"*"`, `"x"`, `"X"`) when `rangeStrategy !== 'update-lockfile'`. The `null` value is then filtered out in `lookup/index.ts` before `updateDependency` is ever called.

**Bug 2 (`">=2.0.0"` — no changes)**:
`getNewValue` with `rangeStrategy: "widen"` (the default for `peerDependencies`) returns `">=2.0.0"` unchanged because `"14.0.0"` already satisfies `">=2.0.0"`. In `lib/modules/manager/npm/update/dependency/index.ts`, `updateDependency()` has an early-return guard:

```typescript
if (oldVersion === newValue) {
  logger.trace('Version is already updated');
  return fileContent; // exits before the upgrade.newName rename logic runs
}
```

`oldVersion (">=2.0.0") === newValue (">=2.0.0")` → early return → `request` is never renamed to `got` in `package.json`.

Note: Bug 2 also affects regular `dependencies` (not just `peerDependencies`) when the current pinned version equals the `replacementVersion`. See related discussion [#39481](https://github.com/renovatebot/renovate/discussions/39481).

## Suggested fix

The early return in `updateDependency` should not fire when the dependency name is changing:

```typescript
// Current:
if (oldVersion === newValue) {
  return fileContent;
}
// Suggested:
if (oldVersion === newValue && (!upgrade.newName || upgrade.newName === depName)) {
  return fileContent;
}
```

For Bug 1, `determineNewReplacementValue` would also need to return `currentValue` (rather than `null`) for X-ranges when a name change is configured.

## Renovate version

`43.252.1`

## Link to the Renovate Discussion

[renovatebot/renovate#44426](https://github.com/renovatebot/renovate/discussions/44426)

## Raw Renovate logs

[2026-07-07 run log](logs/2026-07-07-renovate-run.log)
