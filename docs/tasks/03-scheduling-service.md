# Package 3: Timefold scheduling service

## Objective

Implement a stateless local Timefold service and whole-plan proposal workflow for shoot blocks and post-production capacity.

## Source plan

`docs/superpowers/plans/2026-09-02-video-operations-scheduling-service.md`

## Dependency

Package 2 completed and domain snapshot contract approved.

## Completion evidence

- Java unit, constraint, and REST tests pass.
- Service has no database credentials, database dependency, or host port.
- Hard conflicts are rejected and locked assignments remain fixed.
- Soft scoring covers delivery, load, travel, and cost priorities.
- Stale proposals are rejected.
- Current whole-plan proposals apply atomically or roll back completely.

## Stop gate

Stop for user approval of the fixture outcome and before/after change summary before Package 4.

