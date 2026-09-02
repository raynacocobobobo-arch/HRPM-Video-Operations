# Package 2: Domain and workflow plugin

## Objective

Add the dedicated Video Operations plugin, seven-stage workflow, crew/resource domain, cost snapshots, audit records, and manual scheduling while keeping OpenProject as project truth.

## Source plan

`docs/superpowers/plans/2026-09-02-video-operations-domain-workflow.md`

## Dependency

Package 1 completed and upstream OpenProject behavior approved.

## Completion evidence

- Plugin migrations and models pass focused tests.
- No shadow project table exists.
- Phase gates, hard/soft rules, audited overrides, baselines, change requests, and cost snapshots work.
- Manual assignment works without Timefold.
- Audit records are append-only and transaction rollback is proven.

## Stop gate

Stop for user review of the domain API and confirm the solver snapshot boundary before Package 3.

