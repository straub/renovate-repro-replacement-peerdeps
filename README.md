# replacementName silently fails for npm peerDependencies

Minimal reproduction for a bug where `replacementName`/`replacementVersion` rules in Renovate fail silently when the target package appears in `peerDependencies`, in two distinct ways depending on the declared version range.

## Current behavior

This repo has a `package.json` with `peerDependencies` declaring two packages, each exposing a different failure mode:

| Package | Range | What Renovate does |
|---|---|---|
| `old-package-wildcard` | `"*"` | No replacement PR created, no Dependency Dashboard entry, no log warning |
| `old-package-gte` | `">=1.0.0"` | Branch `renovate/old-package-gte-replacement` is created but `package.json` is unchanged ("No package files need updating") |

## Expected behavior

In both cases, Renovate should create a PR that renames the package in `peerDependencies` and sets the version to `>=2.0.0` (or preserves `"*"` with the new name for the wildcard case).

## Root cause

**Bug 1 (`"*"` — update silently dropped)**:
In `lib/workers/repository/process/lookup/utils.ts`, `determineNewReplacementValue()` calls `versioningApi.getNewValue()`. In `lib/modules/versioning/npm/range.ts`, `getNewValue` returns `null` for any X-range (`"*"`, `"x"`, `"X"`) when `rangeStrategy !== 'update-lockfile'`. The `null` value is then filtered out in `lookup/index.ts` before `updateDependency` is ever called.

**Bug 2 (`">=1.0.0"` — branch created, no changes)**:
`getNewValue` with `rangeStrategy: "widen"` (the default for `peerDependencies`) returns `">=1.0.0"` unchanged because `"2.0.0"` already satisfies `">=1.0.0"`. In `lib/modules/manager/npm/update/dependency/index.ts`, `updateDependency()` has an early-return guard:

```typescript
if (oldVersion === newValue) {
  logger.trace('Version is already updated');
  return fileContent; // exits before the upgrade.newName rename logic runs
}
```

`oldVersion (">=1.0.0") === newValue (">=1.0.0")` → early return → `old-package-gte` is never renamed in `package.json`.

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

[Link TBD — to be added when discussion is filed]
