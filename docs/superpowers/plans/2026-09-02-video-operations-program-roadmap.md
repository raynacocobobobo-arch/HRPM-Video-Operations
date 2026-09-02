# Video Operations Program Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deliver a localhost-only Chinese Video Operations system that runs OpenProject 17.7.2 with a dedicated plugin and uses Timefold Solver 2.6.0 for human-approved crew scheduling.

**Architecture:** OpenProject and the Video Operations plugin share one PostgreSQL source of truth. A stateless Quarkus service based on Timefold receives versioned snapshots and returns proposals; only the plugin can apply a proposal transactionally after the user reviews the D2 before/after calendar.

**Tech Stack:** OpenProject 17.7.2, Ruby on Rails plugin APIs, PostgreSQL 17, TypeScript/Stimulus plugin frontend, Timefold Solver 2.6.0, Quarkus 3.38.3, Java 21, Docker Compose on Colima 0.10.3, RSpec, JUnit 5, RestAssured, Python `unittest` for operations scripts.

**Spec:** `../specs/2026-09-01-video-operations-openproject-timefold-design.md`

## Global Constraints

- Code repository root: the current private Git clone, resolved with `git rev-parse --show-toplevel`.
- Deployment is single-user and local-only; the live stack publishes only `127.0.0.1:8080`. The scheduler remains internal to Compose.
- Do not start installation until `/System/Volumes/Data` has at least 30 GiB free.
- Never delete user files to create disk space; stop at the preflight gate and ask the user to free space.
- Pin OpenProject to `17.7.2`, OpenProject Compose to commit `1cf58dc832fb803ee44fa7632449ce8f8f2b928f`, PostgreSQL to `17`, Timefold Solver to `2.6.0`, Quarkus to `3.38.3`, Java to `21`, and Colima to `0.10.3`.
- Use OpenProject as the only project/work-package source of truth; do not create a shadow project table.
- Timefold is stateless and may not write PostgreSQL.
- Schedule proposals require a matching data version and explicit D2 calendar confirmation.
- ERPNext, Plane, Kimai, Leantime, wedding CRM, hospital ledger, and lighting inventory are outside v1.
- Preserve upstream license and attribution files for GPL-3.0 and Apache-2.0 components.
- Use TDD for every behavior change and commit after every independently passing task.

---

## Package Order

The confirmed specification contains five independently reviewable subsystems. Execute the following plans in order; each plan ends with running software or a separately testable deliverable.

### Package 1: Local foundation

Plan: `2026-09-02-video-operations-local-foundation.md`

Produces:

- A new Git repository with pinned upstream metadata.
- A disk-space safety gate.
- Colima/Docker Compose prerequisites.
- An unmodified OpenProject 17.7.2 instance available at `http://localhost:8080`, with its listener bound only to `127.0.0.1`.

Exit check:

```bash
curl -fsS http://localhost:8080/health_checks/default
./ops/compose.sh ps --format json
```

### Package 2: OpenProject domain and workflow plugin

Plan: `2026-09-02-video-operations-domain-workflow.md`

Consumes Package 1 and produces:

- A custom OpenProject image containing `openproject-video-operations`.
- Crew, skills, availability, rates, travel facts, resource demand, manual assignment, baselines, changes, and overrides.
- Seven-stage gates, cost forecasts, closeout retrospectives, and separately confirmed template-learning proposals.

Exit check:

```bash
./ops/compose.sh run --rm web bundle exec rspec openproject-video-operations/spec
curl -fsS http://localhost:8080/video_operations/api/v1/health
```

### Package 3: Timefold scheduling and transactional apply

Plan: `2026-09-02-video-operations-scheduling-service.md`

Consumes Package 2 and produces:

- A Java 21/Quarkus 3.38.3 scheduling service based on Timefold 2.6.0.
- Tested hard and soft video-production constraints.
- Versioned proposal creation, staleness rejection, and all-or-nothing application.

Exit check:

```bash
./scheduler/mvnw verify
./ops/compose.sh run --rm web bundle exec rspec openproject-video-operations/spec/services/video_operations/scheduling
./ops/compose.sh exec -T scheduler curl -fsS http://localhost:8080/q/health
```

### Package 4: Chinese operational interface

Plan: `2026-09-02-video-operations-chinese-ui.md`

Consumes Package 3 and produces the approved `A1 + N2 + P1 + D2` interface:

- A1 exception-first workbench.
- N2 object-based navigation.
- P1 phase-gate project default page.
- D2 schedule before/after calendar confirmation.

Exit check:

```bash
./ops/compose.sh run --rm web bundle exec rspec openproject-video-operations/spec/features
./ops/compose.sh build web
```

### Package 5: Migration, backup, recovery, and go-live

Plan: `2026-09-02-video-operations-migration-go-live.md`

Consumes Package 4 and produces:

- A dry-run-first legacy/Excel import.
- Idempotent import apply and a migration report.
- Database/assets backup with a verified second-volume copy, seven-daily/four-weekly retention, and tested restoration.
- Final local smoke tests and a persistent local launch command.

Exit check:

```bash
python3 -m unittest discover -s ops/tests -p 'test_*.py'
./ops/smoke.sh
./ops/backup.sh
./ops/restore-drill.sh --latest
```

## Program Checkpoints

- [ ] **Checkpoint 1: Approve disk cleanup result before Package 1 installation**

Run:

```bash
df -h /System/Volumes/Data
```

Expected: `Avail` is at least `30Gi`.

- [ ] **Checkpoint 2: Review OpenProject upstream UI before Package 2 customization**

Confirm that project creation, work packages, time entries, and budgets operate correctly in the pinned vanilla instance.

- [ ] **Checkpoint 3: Review domain API before Package 3 solver integration**

Confirm that project truth remains in OpenProject and plugin tables contain only video-specific resource data.

- [ ] **Checkpoint 4: Review solver fixtures before enabling proposal apply**

Approve the expected outcome for a fixture containing a locked shoot, an unavailable camera operator, a post-production capacity overload, and a location travel conflict.

- [ ] **Checkpoint 5: Review the A1/N2/P1/D2 browser flow before migration**

Confirm the browser sequence: exception → project gate → D2 calendar → apply → audit record.

- [ ] **Checkpoint 6: Approve dry-run migration report before any legacy write**

The report must show created, updated, duplicate, unresolved, and rejected records. No source or target rows may be deleted.

- [ ] **Checkpoint 7: Approve restore drill before go-live**

The restored instance must contain the same project, crew, assignment, baseline, change, attachment, and audit counts as the backup source.

## Completion Definition

The program is complete only when:

1. The website opens from one local URL and no service is reachable through a non-loopback interface.
2. A video project can move through all seven phase gates with hard/soft requirements.
3. Timefold detects the agreed hard conflicts and produces a proposal without moving locked assignments.
4. Manual assignment remains available when Timefold is unavailable.
5. A stale proposal is rejected and a current D2 proposal applies atomically after confirmation.
6. Planned, actual, and forecast costs remain separate; historical rate snapshots do not change; closeout/template learning requires confirmation.
7. The one-time import is dry-run-first and idempotent.
8. A verified second-volume backup restores successfully in an isolated restore drill.
9. All Ruby, Java, operations, and browser feature tests pass from a clean checkout.
