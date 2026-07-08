# renovatebot/renovate#44426

## Current behavior

Renovate fails to create a replacement PR for `peerDependencies` when the target package uses either `"*"` or a satisfied range like `">=2.0.0"`.

## Expected behavior

Renovate should create a replacement PR in both cases.

## Link to the Renovate Discussion

[renovatebot/renovate#44426](https://github.com/renovatebot/renovate/discussions/44426)

## Raw Renovate logs

[2026-07-07 run log](logs/2026-07-07-renovate-run.log)
