# Package 4: Chinese operational interface

## Objective

Implement the approved A1 + N2 + P1 + D2 workflow inside OpenProject's authenticated interface.

## Source plan

`docs/superpowers/plans/2026-09-02-video-operations-chinese-ui.md`

## Dependency

Package 3 completed and proposal behavior approved.

## Completion evidence

- Feature tests cover all eight navigation objects.
- Workbench is exception-first.
- Project default shows baseline/current forecast and seven phase gates.
- Schedule page shows current/proposed calendars and only whole-plan apply/reject.
- Manual scheduling remains available during solver failure.
- Template recommendations require explicit apply/reject.

## Stop gate

Stop for browser review of exception → phase gate → schedule comparison → apply → audit before migration.

