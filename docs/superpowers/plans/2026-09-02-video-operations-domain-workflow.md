# Video Operations Domain and Workflow Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the pinned OpenProject foundation into the authoritative video-project operations system by adding crew, capacity, resource demand, phase gates, change control, and three-view cost forecasting without duplicating OpenProject projects or work packages.

**Architecture:** A dedicated Rails engine named `openproject-video-operations` runs inside the official OpenProject image. OpenProject remains authoritative for projects, work packages, memberships, users, dates, and status; plugin tables reference those records and own only video-operations data. Domain behavior lives in small service objects, controllers expose versioned JSON contracts, and all workflow/cost mutations are transactional and audited.

**Tech Stack:** OpenProject 17.7.2, Ruby on Rails/OpenProject plugin API, PostgreSQL 17, RSpec, FactoryBot, Docker multi-stage builds.

**Spec:** `../specs/2026-09-01-video-operations-openproject-timefold-design.md`

## Package Constraints

- Repository root: the current Git clone, resolved with `git rev-parse --show-toplevel`.
- Package 1 must pass before this plan starts.
- Do not create a plugin `projects` or `work_packages` table.
- Crew records are operational resources, not OpenProject login accounts.
- Store money as integer cents and durations as integer minutes.
- Store all instants in UTC; render the configured local time zone only at the UI boundary.
- Rates are effective-dated. Assignments snapshot the applied rate so history never changes when a rate card is edited.
- Hard phase requirements block advancement. Soft requirements require an override reason and an audit record.
- Do not add Timefold or build the final Chinese workbench in this package.

---

## Planned File Map

```text
video-operations/
├── docker/openproject/
│   ├── Dockerfile
│   └── Gemfile.plugins
├── compose/
│   └── docker-compose.override.yml
└── openproject-video-operations/
    ├── openproject-video-operations.gemspec
    ├── app/
    │   ├── controllers/video_operations/api/v1/
    │   │   ├── base_controller.rb
    │   │   ├── health_controller.rb
    │   │   ├── crew_profiles_controller.rb
    │   │   ├── resource_demands_controller.rb
    │   │   ├── assignments_controller.rb
    │   │   ├── phase_gates_controller.rb
    │   │   └── cost_forecasts_controller.rb
    │   ├── models/video_operations/
    │   │   ├── crew_profile.rb
    │   │   ├── role.rb
    │   │   ├── skill.rb
    │   │   ├── crew_skill.rb
    │   │   ├── availability.rb
    │   │   ├── rate_card.rb
    │   │   ├── location.rb
    │   │   ├── travel_time.rb
    │   │   ├── resource_demand.rb
    │   │   ├── assignment.rb
    │   │   ├── project_state.rb
    │   │   ├── baseline_snapshot.rb
    │   │   ├── change_request.rb
    │   │   ├── override_record.rb
    │   │   ├── audit_event.rb
    │   │   ├── retrospective_snapshot.rb
    │   │   └── template_adjustment_proposal.rb
    │   └── services/video_operations/
    │       ├── gate_evaluation.rb
    │       ├── gate_advance.rb
    │       ├── scheduling_version_bump.rb
    │       ├── audit_recorder.rb
    │       ├── cost_forecast.rb
    │       ├── retrospective_generate.rb
    │       └── template_adjustment_apply.rb
    ├── config/
    │   ├── locales/zh-CN.yml
    │   └── routes.rb
    ├── db/migrate/
    ├── lib/open_project/video_operations/
    │   ├── engine.rb
    │   ├── patches/project_patch.rb
    │   ├── patches/work_package_patch.rb
    │   └── version.rb
    └── spec/
        ├── factories/video_operations.rb
        ├── models/video_operations/
        ├── requests/video_operations/api/v1/
        └── services/video_operations/
```

### Task 1: Scaffold the plugin and custom OpenProject image

**Files:**
- Create: `openproject-video-operations/openproject-video-operations.gemspec`
- Create: `openproject-video-operations/lib/open_project/video_operations/version.rb`
- Create: `openproject-video-operations/lib/open_project/video_operations/engine.rb`
- Create: `openproject-video-operations/config/routes.rb`
- Create: `openproject-video-operations/app/controllers/video_operations/api/v1/base_controller.rb`
- Create: `openproject-video-operations/app/controllers/video_operations/api/v1/health_controller.rb`
- Create: `openproject-video-operations/spec/requests/video_operations/api/v1/health_spec.rb`
- Create: `docker/openproject/Gemfile.plugins`
- Create: `docker/openproject/Dockerfile`
- Create: `compose/docker-compose.override.yml`
- Modify: `Makefile`

**Interfaces:**
- Adds `GET /video_operations/api/v1/health` returning `{"status":"ok","plugin_version":"0.1.0"}`.
- Produces image `video-operations/openproject:17.7.2-plugin` from the `web` service build and reuses it for `worker`, `cron`, and `seeder`.

- [ ] **Step 1: Write the failing health request spec**

```ruby
RSpec.describe "Video Operations health", type: :request do
  it "returns the plugin version" do
    get "/video_operations/api/v1/health"

    expect(response).to have_http_status(:ok)
    expect(response.parsed_body).to eq(
      "status" => "ok",
      "plugin_version" => OpenProject::VideoOperations::VERSION
    )
  end
end
```

Run:

```bash
./ops/compose.sh run --rm web bundle exec rspec openproject-video-operations/spec/requests/video_operations/api/v1/health_spec.rb
```

Expected: fail because the engine and route do not exist.

- [ ] **Step 2: Implement the smallest engine, route, and controller**

Register the engine with the OpenProject plugin API, set `OpenProject::VideoOperations::VERSION = "0.1.0"`, and scope JSON routes under `/video_operations/api/v1`. The base controller must inherit the OpenProject application controller, require an authenticated local user for domain endpoints, and let only the health action skip authentication.

- [ ] **Step 3: Add the custom image build**

`docker/openproject/Gemfile.plugins`:

```ruby
group :opf_plugins do
  gem "openproject-video-operations", path: "/app/vendor/plugins/openproject-video-operations"
end
```

`docker/openproject/Dockerfile` must:

1. Use `openproject/openproject:17.7.2` as the build stage.
2. Copy the plugin to `/app/vendor/plugins/openproject-video-operations`.
3. Copy `Gemfile.plugins`, run `bundle install`, and run OpenProject's production asset precompile script.
4. Copy the resulting bundle, plugin, and assets into `openproject/openproject:17.7.2-slim`.
5. Preserve upstream entrypoints and license files.

Extend the Package 1 Compose override with this exact ownership pattern:

```yaml
services:
  web:
    image: video-operations/openproject:17.7.2-plugin
    build:
      # Paths resolve from the first Compose file in vendor/openproject-docker-compose.
      context: ../..
      dockerfile: docker/openproject/Dockerfile
  worker:
    image: video-operations/openproject:17.7.2-plugin
  cron:
    image: video-operations/openproject:17.7.2-plugin
  seeder:
    image: video-operations/openproject:17.7.2-plugin
```

Retain the locale environment block already in the override. Do not replace the upstream PostgreSQL, cache, proxy, or Hocuspocus definitions. The Dockerfile's final slim stage must also copy `/app/vendor/plugins/openproject-video-operations` because the local-path gem must exist at runtime.

- [ ] **Step 4: Build, migrate, and rerun the health spec**

```bash
./ops/compose.sh build web
./ops/compose.sh run --rm web bundle exec rails db:migrate
./ops/compose.sh run --rm web bundle exec rspec openproject-video-operations/spec/requests/video_operations/api/v1/health_spec.rb
curl -fsS http://localhost:8080/video_operations/api/v1/health
```

Expected: spec passes and curl returns the exact JSON contract.

- [ ] **Step 5: Commit**

```bash
git add docker compose Makefile openproject-video-operations
git commit -m "feat: load video operations OpenProject plugin"
```

### Task 2: Model crew, skills, availability, effective rates, and travel facts

**Files:**
- Create: `openproject-video-operations/db/migrate/20260902090100_create_video_operations_people.rb`
- Create: `openproject-video-operations/app/models/video_operations/crew_profile.rb`
- Create: `openproject-video-operations/app/models/video_operations/role.rb`
- Create: `openproject-video-operations/app/models/video_operations/skill.rb`
- Create: `openproject-video-operations/app/models/video_operations/crew_skill.rb`
- Create: `openproject-video-operations/app/models/video_operations/availability.rb`
- Create: `openproject-video-operations/app/models/video_operations/rate_card.rb`
- Create: `openproject-video-operations/app/models/video_operations/location.rb`
- Create: `openproject-video-operations/app/models/video_operations/travel_time.rb`
- Create: `openproject-video-operations/spec/factories/video_operations.rb`
- Create: `openproject-video-operations/spec/models/video_operations/crew_profile_spec.rb`
- Create: `openproject-video-operations/spec/models/video_operations/rate_card_spec.rb`
- Create: `openproject-video-operations/spec/models/video_operations/travel_time_spec.rb`

**Schema:**

- `crew_profiles`: `display_name`, `employment_type`, `phone`, `active`, `preferred_daily_minutes` default `480`, `maximum_daily_minutes` default `720`, `minimum_rest_minutes` default `660`, `notes`, timestamps. These are explicit company scheduling policies, not legal claims.
- `roles`: unique `code`, `name_zh`, timestamps.
- `skills`: unique `code`, `name_zh`, timestamps.
- `crew_skills`: `crew_profile_id`, `skill_id`, `level` from 1–5; unique pair.
- `availabilities`: `crew_profile_id`, `starts_at`, `ends_at`, `kind` (`available`, `unavailable`, `leave`), `note`; validate `ends_at > starts_at`.
- `rate_cards`: `crew_profile_id`, `rate_type` (`hourly`, `daily`, `fixed`), `amount_cents`, `currency` fixed to `CNY`, `effective_from`, nullable `effective_to`; prohibit overlapping effective intervals for one person/rate type.
- `locations`: unique `code`, `name_zh`, `address`, `active`, timestamps.
- `travel_times`: `origin_location_id`, `destination_location_id`, positive `minutes`; unique directed pair, with no assumption that reverse travel takes the same time.

**Interfaces:**
- `CrewProfile#available_for?(starts_at:, ends_at:) -> Boolean`.
- `CrewProfile#effective_rate(rate_type:, occurred_at:) -> RateCard | nil`.
- `RateCard#cost_for(minutes:) -> Integer` in cents; hourly rounds to the nearest minute, daily rounds up per eight-hour day, fixed ignores minutes.
- `TravelTime.minutes_between(origin:, destination:) -> Integer`; same-location returns zero, while a missing directed pair is a validation error for production scheduling rather than guessed travel time.

- [ ] **Step 1: Write failing model examples**

Cover these exact cases:

- a crew member is unavailable if any `unavailable` or `leave` interval overlaps the requested half-open interval;
- touching endpoints do not overlap;
- an inactive crew member is unavailable;
- preferred/max daily capacity and minimum rest are positive, ordered (`preferred <= maximum`), and serialized into scheduling snapshots;
- a current effective rate is selected by date;
- overlapping rate periods are rejected;
- `3000` cents/hour for `90` minutes costs `4500` cents;
- `120000` cents/day for `481` minutes costs `240000` cents.
- directed A→B and B→A travel values may differ, same-location is zero, duplicate pairs and nonpositive minutes are rejected, and a missing pair reports an explicit configuration error.

Run:

```bash
./ops/compose.sh run --rm web bundle exec rspec \
  openproject-video-operations/spec/models/video_operations/crew_profile_spec.rb \
  openproject-video-operations/spec/models/video_operations/rate_card_spec.rb \
  openproject-video-operations/spec/models/video_operations/travel_time_spec.rb
```

Expected: fail because the tables and models do not exist.

- [ ] **Step 2: Add the migration and minimal model behavior**

Use namespaced tables with `video_operations_` prefixes, foreign keys, indexes on every foreign key and interval boundary, and database check constraints for positive amounts, valid skill levels, and ordered intervals. Add model validation for readable error messages and database constraints for integrity.

- [ ] **Step 3: Run migration and focused specs**

```bash
./ops/compose.sh run --rm web bundle exec rails db:migrate
./ops/compose.sh run --rm web bundle exec rspec openproject-video-operations/spec/models/video_operations
```

Expected: all examples pass.

- [ ] **Step 4: Commit**

```bash
git add openproject-video-operations
git commit -m "feat: model crew skills availability and rates"
```

### Task 3: Model project demand and auditable assignments

**Files:**
- Create: `openproject-video-operations/db/migrate/20260902090200_create_video_operations_resourcing.rb`
- Create: `openproject-video-operations/app/models/video_operations/resource_demand.rb`
- Create: `openproject-video-operations/app/models/video_operations/assignment.rb`
- Create: `openproject-video-operations/app/models/video_operations/project_state.rb`
- Create: `openproject-video-operations/app/services/video_operations/scheduling_version_bump.rb`
- Create: `openproject-video-operations/lib/open_project/video_operations/patches/project_patch.rb`
- Create: `openproject-video-operations/lib/open_project/video_operations/patches/work_package_patch.rb`
- Create: `openproject-video-operations/spec/models/video_operations/resource_demand_spec.rb`
- Create: `openproject-video-operations/spec/models/video_operations/assignment_spec.rb`
- Create: `openproject-video-operations/spec/services/video_operations/scheduling_version_bump_spec.rb`
- Create: `openproject-video-operations/spec/requests/video_operations/api/v1/resource_demands_spec.rb`
- Create: `openproject-video-operations/app/controllers/video_operations/api/v1/resource_demands_controller.rb`
- Create: `openproject-video-operations/spec/requests/video_operations/api/v1/assignments_spec.rb`
- Create: `openproject-video-operations/app/controllers/video_operations/api/v1/assignments_controller.rb`

**Schema:**

- `resource_demands`: OpenProject `project_id`, nullable `work_package_id`, `stage_code`, `role_id`, `required_skill_id`, `minimum_skill_level`, `mode` (`fixed_block`, `daily_capacity`), `starts_at`, `ends_at`, `required_minutes`, `headcount`, nullable `location_id`, `locked`, `data_version`, timestamps. Fixed production blocks require a location; post capacity demands do not.
- `assignments`: `resource_demand_id`, `crew_profile_id`, nullable `work_package_id`, `starts_at`, `ends_at`, `allocated_minutes`, `status` (`proposed`, `confirmed`, `completed`, `cancelled`), `rate_type_snapshot`, `rate_cents_snapshot`, `currency_snapshot`, nullable `actual_minutes`, `source` (`manual`, `solver`, `import`), timestamps.
- `project_states`: unique OpenProject `project_id`, `stage_code` default `project_intake`, `gate_data_version` default `1`, and `scheduling_data_version` default `1`; it is a one-to-one workflow/revision extension, not a shadow project record.

**Interfaces:**
- `ResourceDemand#bump_data_version!` increments the optimistic version inside the same transaction as a demand edit.
- `Assignment#confirm!(rate_card:)` snapshots rate fields and refuses a missing or ineffective rate.
- `POST /video_operations/api/v1/projects/:project_id/resource_demands` creates demand against an existing OpenProject project.
- `POST /video_operations/api/v1/projects/:project_id/assignments` creates a `source: manual` assignment, validates skill/availability/overlap, snapshots the effective rate, and may mark it locked.
- `PATCH /video_operations/api/v1/assignments/:id` adjusts, locks/unlocks, completes, or cancels a manual assignment with optimistic versioning; solver-owned assignments cannot be edited through this endpoint until they are explicitly converted to manual.
- `SchedulingVersionBump.for_project!(project:)` and `SchedulingVersionBump.for_crew!(crew_profile:)` atomically advance the affected projects' scheduling revisions.
- The API returns `409` when an update sends a stale `data_version`.

- [ ] **Step 1: Write failing integrity and API specs**

The examples must prove:

- a demand cannot reference a nonexistent project or work package;
- a work package must belong to the demand's project;
- an assignment cannot end before it starts or allocate non-positive minutes;
- confirming snapshots the effective rate;
- editing the future rate card does not change confirmed assignment cost;
- a stale demand update returns `409` and leaves the record unchanged;
- manual create/edit works while the scheduler endpoint is unavailable and increments the project scheduling revision;
- an overlapping, unavailable, or under-skilled manual assignment returns `422` with named conflicts and writes nothing.
- OpenProject project/work-package date or stage-readiness commits, plus crew skill/availability/rate commits, monotonically change every affected project revision; rolled-back and unrelated changes do not.

- [ ] **Step 2: Implement migration, models, and controller**

Use OpenProject's `Project` and `WorkPackage` associations directly. Keep controller logic limited to authorization, parameter coercion, service/model invocation, and serialization. Do not create crew login accounts or memberships. Register narrowly scoped `after_commit` concerns through engine `to_prepare` so ordinary OpenProject date/stage edits call the bump service without modifying OpenProject core files.

- [ ] **Step 3: Run focused tests**

```bash
./ops/compose.sh run --rm web bundle exec rspec \
  openproject-video-operations/spec/models/video_operations/resource_demand_spec.rb \
  openproject-video-operations/spec/models/video_operations/assignment_spec.rb \
  openproject-video-operations/spec/services/video_operations/scheduling_version_bump_spec.rb \
  openproject-video-operations/spec/requests/video_operations/api/v1/resource_demands_spec.rb \
  openproject-video-operations/spec/requests/video_operations/api/v1/assignments_spec.rb
```

- [ ] **Step 4: Commit**

```bash
git add openproject-video-operations
git commit -m "feat: add project resource demands and assignments"
```

### Task 4: Add seven-stage gates, baselines, changes, and overrides

**Files:**
- Create: `openproject-video-operations/db/migrate/20260902090300_create_video_operations_governance.rb`
- Create: `openproject-video-operations/app/models/video_operations/baseline_snapshot.rb`
- Create: `openproject-video-operations/app/models/video_operations/change_request.rb`
- Create: `openproject-video-operations/app/models/video_operations/override_record.rb`
- Create: `openproject-video-operations/app/models/video_operations/audit_event.rb`
- Create: `openproject-video-operations/app/services/video_operations/gate_evaluation.rb`
- Create: `openproject-video-operations/app/services/video_operations/gate_advance.rb`
- Create: `openproject-video-operations/app/services/video_operations/audit_recorder.rb`
- Create: `openproject-video-operations/app/controllers/video_operations/api/v1/phase_gates_controller.rb`
- Create: `openproject-video-operations/spec/services/video_operations/gate_evaluation_spec.rb`
- Create: `openproject-video-operations/spec/services/video_operations/gate_advance_spec.rb`
- Create: `openproject-video-operations/spec/services/video_operations/audit_recorder_spec.rb`
- Create: `openproject-video-operations/spec/requests/video_operations/api/v1/phase_gates_spec.rb`

**Stage sequence:**

```ruby
STAGES = %w[
  project_intake
  preproduction
  shoot_preparation
  shooting
  postproduction
  review_revision
  delivery_close
].freeze
```

**Schema:**

- `baseline_snapshots`: `project_id`, `kind` (`scope`, `schedule`, `cost`), `version`, `payload` JSONB, `created_by_id`, timestamps; immutable after insert.
- `change_requests`: `project_id`, `title`, `reason`, `impact_payload` JSONB, `status` (`draft`, `approved`, `rejected`, `applied`), `requested_by_id`, nullable `decided_by_id`, timestamps.
- `override_records`: `project_id`, `stage_code`, `requirement_code`, `reason`, `actor_id`, `context_payload` JSONB, timestamp; immutable.
- `audit_events`: nullable `project_id`, `actor_id`, `event_type`, `auditable_type`, `auditable_id`, `before_payload` JSONB, `after_payload` JSONB, `request_id`, timestamp; immutable and indexed by project/time and auditable object/time.
- `retrospective_snapshots`: unique `project_id`, `payload` JSONB, `created_by_id`, timestamp; immutable once the project closes.
- `template_adjustment_proposals`: `project_id`, native OpenProject `template_project_id`, `before_payload` JSONB, `proposed_payload` JSONB, `status` (`proposed`, `applied`, `rejected`), `created_by_id`, nullable `decided_by_id`, nullable `decided_at`, timestamps.
**Interfaces:**
- `GateEvaluation.call(project:) -> GateEvaluation::Result` with `stage_code`, `hard_blockers`, `soft_warnings`, `gate_data_version`.
- `GateAdvance.call(project:, actor:, expected_gate_data_version:, override_reason: nil) -> GateAdvance::Result`.
- Hard blockers always stop advancement.
- Soft warnings stop advancement unless `override_reason` is nonblank; a successful override creates one `OverrideRecord` per warning.
- Advancing captures a baseline appropriate to the stage and increments `ProjectState#gate_data_version` transactionally.
- `AuditRecorder.record!(actor:, event_type:, auditable:, before:, after:, request_id:)` records plugin mutations in the same transaction; core OpenProject project/work-package changes continue to use native journals.

- [ ] **Step 1: Write the failing evaluation matrix**

Test every boundary in the seven-stage sequence. At minimum, encode these concrete requirements:

| Leaving stage | Hard requirements | Soft warnings |
|---|---|---|
| 项目准入 | owner, client, target delivery date | budget not baselined |
| 前期策划 | approved scope, deliverable work packages | creative brief incomplete |
| 拍摄准备 | shoot dates, location, required resource demands | noncritical equipment note |
| 拍摄执行 | confirmed assignments, call-sheet work package | overtime risk |
| 后期制作 | footage handoff complete, edit work package dates | proxy/archive note |
| 审片修改 | review decision recorded, open revisions classified | low-priority revision remains |
| 交付关闭 | delivery accepted, actual time captured, final cost computed | retrospective missing |

Use existing OpenProject custom fields/statuses for facts that belong to projects/work packages and plugin records only for video-specific facts.

- [ ] **Step 2: Implement evaluation as a pure read service**

`GateEvaluation` may query models but may not mutate them. Return machine-readable blocker objects:

```ruby
Requirement = Data.define(:code, :severity, :message_zh, :object_type, :object_id)
Result = Data.define(:stage_code, :hard_blockers, :soft_warnings, :gate_data_version)
```

- [ ] **Step 3: Write failing advancement transaction specs**

Prove hard blockers leave every table unchanged, soft warnings require a reason, a valid reason creates immutable override/audit records, stale versions return a conflict, and a forced exception during baseline or audit creation rolls the entire transaction back. Add focused audit examples proving actor/request IDs, minimal before/after fields, append-only behavior, and rejection of secret/rate-authentication keys.

- [ ] **Step 4: Implement `GateAdvance` and the phase-gate API**

Use `Project.transaction` and lock the `ProjectState` row. Return `422` for blockers, `409` for a stale `expected_gate_data_version`, and `200` with the new stage/version after success. Record actor IDs through the authenticated OpenProject user.

- [ ] **Step 5: Run focused and aggregate specs**

```bash
./ops/compose.sh run --rm web bundle exec rspec \
  openproject-video-operations/spec/services/video_operations/gate_evaluation_spec.rb \
  openproject-video-operations/spec/services/video_operations/gate_advance_spec.rb \
  openproject-video-operations/spec/services/video_operations/audit_recorder_spec.rb \
  openproject-video-operations/spec/requests/video_operations/api/v1/phase_gates_spec.rb
```

- [ ] **Step 6: Commit**

```bash
git add openproject-video-operations
git commit -m "feat: enforce video project phase gates"
```

### Task 5: Add cost forecasts, closeout retrospective, and confirmed template learning

**Files:**
- Create: `openproject-video-operations/app/services/video_operations/cost_forecast.rb`
- Create: `openproject-video-operations/app/models/video_operations/retrospective_snapshot.rb`
- Create: `openproject-video-operations/app/models/video_operations/template_adjustment_proposal.rb`
- Create: `openproject-video-operations/app/services/video_operations/retrospective_generate.rb`
- Create: `openproject-video-operations/app/services/video_operations/template_adjustment_apply.rb`
- Create: `openproject-video-operations/app/controllers/video_operations/api/v1/cost_forecasts_controller.rb`
- Create: `openproject-video-operations/app/controllers/video_operations/api/v1/template_adjustments_controller.rb`
- Create: `openproject-video-operations/spec/services/video_operations/cost_forecast_spec.rb`
- Create: `openproject-video-operations/spec/services/video_operations/retrospective_generate_spec.rb`
- Create: `openproject-video-operations/spec/services/video_operations/template_adjustment_apply_spec.rb`
- Create: `openproject-video-operations/spec/requests/video_operations/api/v1/cost_forecasts_spec.rb`
- Create: `openproject-video-operations/spec/requests/video_operations/api/v1/template_adjustments_spec.rb`

**Interfaces:**

```ruby
VideoOperations::CostForecast.call(project:) # => Result

Result = Data.define(
  :baseline_cents,
  :actual_cents,
  :remaining_forecast_cents,
  :forecast_at_completion_cents,
  :approved_change_cents,
  :margin_cents,
  :currency,
  :calculated_at
)
```

Rules:

- baseline comes from the latest immutable cost baseline;
- actual labor cost uses confirmed/completed assignment rate snapshots and `actual_minutes` when present;
- remaining forecast uses confirmed future allocations plus open resource demand at current effective rates;
- approved changes are shown separately and included in the forecast-at-completion;
- margin is approved revenue minus forecast-at-completion;
- every result is in `CNY`; a non-CNY input is rejected before calculation.

**Closeout interfaces:**

- `RetrospectiveGenerate.call(project:, actor:) -> RetrospectiveSnapshot` requires delivery acceptance, actual time, and final cost, then freezes baseline/current/actual schedule, labor, cost, change-count, and on-time-delivery comparisons.
- If the project came from a native OpenProject template, closeout creates a `TemplateAdjustmentProposal`; it does not change the template.
- `TemplateAdjustmentApply.call(proposal:, actor:, decision:)` accepts only `apply` or `reject`. `apply` updates resource-demand minutes/cost assumptions on the native template project, snapshots before/after, increments the template scheduling revision, and writes an audit event transactionally.
- `POST /video_operations/api/v1/template_adjustments/:id/decision` is the only template-learning mutation endpoint.

- [ ] **Step 1: Write failing cost examples**

Cover zero-state, a completed assignment, future confirmed work, unsatisfied demand, approved change, historic rate preservation, and a project with no approved revenue. Assert exact integer-cent results.

- [ ] **Step 2: Implement the pure calculation service**

Do not persist derived totals. The JSON endpoint returns the result plus the newest source timestamp so the UI can state when it was calculated.

- [ ] **Step 3: Write failing closeout and template-learning examples**

Prove a closeout without accepted delivery, actual time, or final cost is rejected; a valid closeout creates one immutable plan-vs-actual snapshot; retry is idempotent; a template-based project creates a proposal but leaves template demands unchanged; reject writes only decision/audit state; apply updates only the fields shown in the proposal; stale template fingerprints return `409`; a late failure rolls back template, proposal, revision, and audit writes.

- [ ] **Step 4: Implement closeout and explicit template decisions**

Hook retrospective generation into successful passage of the final `delivery_close` gate, immediately before setting the native OpenProject project to its configured closed status. `closed` is terminal state, not an eighth video stage. Use native OpenProject template project/work-package IDs and plugin resource demands attached to that template—do not create a parallel template-project table. Require the same authenticated local user confirmation used by the settings page.

- [ ] **Step 5: Run tests**

```bash
./ops/compose.sh run --rm web bundle exec rspec \
  openproject-video-operations/spec/services/video_operations/cost_forecast_spec.rb \
  openproject-video-operations/spec/services/video_operations/retrospective_generate_spec.rb \
  openproject-video-operations/spec/services/video_operations/template_adjustment_apply_spec.rb \
  openproject-video-operations/spec/requests/video_operations/api/v1/cost_forecasts_spec.rb \
  openproject-video-operations/spec/requests/video_operations/api/v1/template_adjustments_spec.rb
```

- [ ] **Step 6: Commit**

```bash
git add openproject-video-operations
git commit -m "feat: forecast cost and learn from closeout"
```

### Task 6: Lock the plugin API contract and verify the package

**Files:**
- Create: `openproject-video-operations/spec/requests/video_operations/api/v1/contract_spec.rb`
- Create: `openproject-video-operations/spec/security/video_operations_authorization_spec.rb`
- Modify: `README.md`
- Modify: `UPSTREAMS.md`

- [ ] **Step 1: Write contract and authorization specs**

Verify:

- unauthenticated domain requests return `401`/OpenProject's standard unauthenticated response;
- a signed-in user without project visibility receives `403` or `404` according to OpenProject convention;
- JSON errors have `code`, `message_zh`, `details`, and `recovery_action_zh`; domain conflicts identify the affected object and never return only a generic failure;
- every mutation response contains the relevant optimistic revision: row `data_version`, `gate_data_version`, and/or project `scheduling_data_version`;
- health remains available locally without authentication;
- no route exposes a project-creation API outside OpenProject.

- [ ] **Step 2: Run the full plugin suite and migration check**

```bash
./ops/compose.sh run --rm web bundle exec rails db:migrate:status
./ops/compose.sh run --rm web bundle exec rspec openproject-video-operations/spec
./ops/compose.sh build --no-cache web
./ops/up.sh
./ops/smoke.sh
curl -fsS http://localhost:8080/video_operations/api/v1/health
```

Expected: migrations are up, all plugin specs pass, the image rebuilds from the pinned base, and only loopback ports are published.

- [ ] **Step 3: Inspect the schema for forbidden duplicate ownership**

```bash
./ops/compose.sh exec -T db psql -U postgres -d openproject -Atc \
  "SELECT tablename FROM pg_tables WHERE schemaname='public' AND tablename LIKE 'video_operations_%' ORDER BY 1"
```

Expected: plugin extension tables appear; no `video_operations_projects` or `video_operations_work_packages` table appears.

- [ ] **Step 4: Update operator documentation and commit**

Document the plugin image build, migrations, health URL, table ownership boundary, and the exact upstream plugin example/license references.

```bash
git add README.md UPSTREAMS.md openproject-video-operations
git commit -m "test: lock video operations domain contracts"
git status --short
```

Expected: working tree is clean.

## Package Completion Criteria

- The custom image is based on OpenProject `17.7.2` and reproducibly contains plugin version `0.1.0`.
- OpenProject owns all project/work-package records; plugin foreign keys reference them directly.
- Crew, skills, availability, effective rates, demands, assignments, baselines, changes, and overrides have model and database integrity tests.
- Seven phase gates reject hard blockers and audit reasoned soft overrides.
- Planned, actual, remaining, forecast-at-completion, change, and margin figures are deterministic integer-cent calculations.
- Project closeout freezes a plan-vs-actual retrospective; any template adjustment remains a separately confirmed, stale-checked, audited action.
- All mutations are authenticated, authorized, versioned, append-only audited, and transactionally tested.
- The full plugin suite, image rebuild, database migration, and localhost smoke checks pass.
