# Package 1: Local foundation

## Objective

Build a reproducible, pinned, localhost-only OpenProject foundation on the new Apple Silicon Mac without touching legacy HRPM data.

## Source plan

`docs/superpowers/plans/2026-09-02-video-operations-local-foundation.md`

## Prerequisites

- Repository clone is clean.
- New Mac preflight is recorded and approved.
- At least 30 GiB free space.

## Completion evidence

- All Package 1 Python tests pass.
- Pinned submodule and runtime versions match the plan.
- Only `127.0.0.1:8080` is browser-facing.
- `./ops/smoke.sh` passes.
- No secret or generated runtime file is tracked.

## Stop gate

Stop for user review of the vanilla OpenProject project, work-package, time-entry, and budget flow before Package 2.

