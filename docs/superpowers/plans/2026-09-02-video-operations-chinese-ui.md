# Video Operations Chinese Interface Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deliver the approved `A1 + N2 + P1 + D2` Chinese operating interface inside OpenProject so the user can run video projects without working directly in generic OpenProject administration screens.

**Architecture:** The plugin adds server-rendered Rails pages inside OpenProject's authenticated layout and uses one small Stimulus controller for the interactive schedule comparison dialog. Views call read-oriented query objects; they never reproduce workflow, cost, or apply rules. OpenProject's design tokens and accessible components provide the visual foundation, avoiding a separate SPA and a second login/navigation shell.

**Tech Stack:** OpenProject 17.7.2 plugin UI, Rails ERB, TypeScript/Stimulus, OpenProject design tokens/components, RSpec request/feature specs, Capybara, existing OpenProject JavaScript build.

**Spec:** `../specs/2026-09-01-video-operations-openproject-timefold-design.md`

## Package Constraints

- Packages 1–3 must pass before this plan starts.
- UI language in v1 is Simplified Chinese; internal codes remain stable English identifiers.
- A1 means the first screen prioritizes exceptions, decisions, and due work—not vanity totals.
- N2 navigation order is exactly: `工作台 / 项目 / 人员 / 档期 / 工时 / 成本 / 报表 / 模板与设置`.
- P1 means every project link inside Video Operations opens the phase-gate overview first.
- D2 means a schedule proposal opens a before/after calendar comparison and only offers `整案应用` or `返回调整/重新排程`.
- Never expose partial schedule-apply controls.
- Every mutation remains a call to the domain API/service from Packages 2–3.
- Reuse OpenProject authentication, layout, colors, spacing, forms, tables, dialogs, and keyboard behavior; do not create a parallel design system.
- Do not implement mobile-native screens. At 1024 px the full workbench must remain usable; at narrower widths tables may scroll horizontally.

---

## Planned File Map

```text
openproject-video-operations/
├── app/
│   ├── controllers/video_operations/
│   │   ├── base_controller.rb
│   │   ├── workbench_controller.rb
│   │   ├── projects_controller.rb
│   │   ├── people_controller.rb
│   │   ├── availability_controller.rb
│   │   ├── time_entries_controller.rb
│   │   ├── costs_controller.rb
│   │   ├── reports_controller.rb
│   │   ├── settings_controller.rb
│   │   └── schedules_controller.rb
│   ├── helpers/video_operations/
│   │   ├── application_helper.rb
│   │   └── schedule_helper.rb
│   ├── queries/video_operations/
│   │   ├── workbench_query.rb
│   │   ├── project_overview_query.rb
│   │   ├── people_capacity_query.rb
│   │   └── schedule_diff_query.rb
│   └── views/video_operations/
│       ├── shared/_navigation.html.erb
│       ├── shared/_empty_state.html.erb
│       ├── workbench/show.html.erb
│       ├── projects/index.html.erb
│       ├── projects/show.html.erb
│       ├── people/index.html.erb
│       ├── availability/index.html.erb
│       ├── time_entries/index.html.erb
│       ├── costs/index.html.erb
│       ├── reports/index.html.erb
│       ├── settings/show.html.erb
│       ├── schedules/show.html.erb
│       └── schedules/_diff_dialog.html.erb
├── config/locales/zh-CN.yml
├── config/routes.rb
├── frontend/module/
│   ├── stimulus/schedule_diff.controller.ts
│   └── global_styles.scss
└── spec/
    ├── features/video_operations/
    │   ├── navigation_spec.rb
    │   ├── workbench_spec.rb
    │   ├── project_phase_gate_spec.rb
    │   ├── people_and_costs_spec.rb
    │   └── schedule_diff_spec.rb
    ├── queries/video_operations/
    └── requests/video_operations/pages_spec.rb
```

### Task 1: Register the N2 module, routes, navigation, and Chinese copy

**Files:**
- Modify: `openproject-video-operations/lib/open_project/video_operations/engine.rb`
- Modify: `openproject-video-operations/config/routes.rb`
- Modify: `openproject-video-operations/config/locales/zh-CN.yml`
- Create: `openproject-video-operations/app/controllers/video_operations/base_controller.rb`
- Create: `openproject-video-operations/app/views/video_operations/shared/_navigation.html.erb`
- Create: `openproject-video-operations/spec/features/video_operations/navigation_spec.rb`
- Create: `openproject-video-operations/spec/requests/video_operations/pages_spec.rb`

**Routes:**

```text
/video_operations                         工作台
/video_operations/projects                项目
/video_operations/projects/:project_id    P1 项目阶段页
/video_operations/people                  人员
/video_operations/availability            档期
/video_operations/time_entries            工时
/video_operations/costs                   成本
/video_operations/reports                 报表
/video_operations/settings                模板与设置
/video_operations/projects/:project_id/schedule_proposals/:id  D2 排程比较
```

**Interfaces:**
- Engine registers one top-level OpenProject global menu item `视频项目运营`, linking to `/video_operations`.
- Every N2 item has visible Chinese text, stable route, active state, and authorization check.
- Project links in every plugin page resolve to `/video_operations/projects/:project_id`, making P1 the Video Operations project default.

- [ ] **Step 1: Write the failing navigation feature spec**

Log in through existing OpenProject test helpers and assert:

- top-level `视频项目运营` is visible;
- the eight N2 labels appear in the approved order;
- each link returns `200` for an authorized user;
- the active item has `aria-current="page"`;
- a user without project visibility cannot infer hidden project names;
- unauthenticated requests follow OpenProject's standard sign-in path.

Run:

```bash
./ops/compose.sh run --rm web bundle exec rspec \
  openproject-video-operations/spec/features/video_operations/navigation_spec.rb \
  openproject-video-operations/spec/requests/video_operations/pages_spec.rb
```

Expected: fail because routes and navigation are missing.

- [ ] **Step 2: Implement the module and semantic navigation**

Use OpenProject's layout and menu registration hooks. Render N2 as a `<nav aria-label="视频项目运营">` with an ordered list. Do not copy OpenProject's global sidebar. All text must come from `zh-CN.yml`; no visible Chinese string should be embedded in controllers or query objects.

- [ ] **Step 3: Add minimal authorized page endpoints**

Each page initially renders a localized title and intentional empty state. The settings page is local-admin only; normal object pages use OpenProject visibility scopes.

- [ ] **Step 4: Rerun tests and commit**

```bash
./ops/compose.sh run --rm web bundle exec rspec \
  openproject-video-operations/spec/features/video_operations/navigation_spec.rb \
  openproject-video-operations/spec/requests/video_operations/pages_spec.rb
git add openproject-video-operations
git commit -m "feat: add Chinese video operations navigation"
```

### Task 2: Build the A1 exception-first workbench

**Files:**
- Create: `openproject-video-operations/app/queries/video_operations/workbench_query.rb`
- Create: `openproject-video-operations/app/controllers/video_operations/workbench_controller.rb`
- Create: `openproject-video-operations/app/views/video_operations/workbench/show.html.erb`
- Create: `openproject-video-operations/app/views/video_operations/shared/_empty_state.html.erb`
- Create: `openproject-video-operations/spec/queries/video_operations/workbench_query_spec.rb`
- Create: `openproject-video-operations/spec/features/video_operations/workbench_spec.rb`

**Workbench sections, in order:**

1. `需要你决定` — stale/feasible schedule proposals awaiting action, phase-gate warnings requiring an override decision, and change requests awaiting a decision.
2. `今天有风险` — unavailable confirmed crew, schedule conflicts, hard gate blockers, overdue critical work packages, and cost forecast overruns.
3. `未来 7 天` — shoots, review deadlines, delivery dates, and capacity shortages.
4. `项目概览` — compact project rows with stage, next milestone, owner, and risk level.

Summary counts may appear beside these headings but may not displace the exception list.

**Interfaces:**
- `WorkbenchQuery.call(user:, now:) -> Result`.
- `Result` exposes ordered `decisions`, `risks`, `next_seven_days`, and `projects` arrays.
- Every item has `kind`, `severity`, `title_zh`, `context_zh`, `due_at`, `project_id`, and `action_path`.
- Sort decisions by oldest waiting first; risks by severity then due time; upcoming events chronologically.

- [ ] **Step 1: Write failing query tests using a fixed clock**

Cover one item of every kind, visibility scoping, exact ordering, and boundary times at `now`, end of today, and seven days. Prove a healthy project does not generate a false risk and a hidden project contributes neither rows nor counts.

- [ ] **Step 2: Implement the read-only query object**

Use bounded SQL relations/eager loading; the feature spec must assert the page remains within an agreed query budget for ten fixture projects. The query object must not persist anything or call Timefold.

- [ ] **Step 3: Write the failing A1 feature spec**

Assert headings in the required order, red/amber severity conveyed by both text/icon and color, visible project context, meaningful action links, and the empty state `当前没有需要处理的异常` when all four sections are empty.

- [ ] **Step 4: Render the workbench with OpenProject components**

Use semantic headings, lists/tables appropriate to density, OpenProject buttons and status badges, and one prominent primary action only when a decision is available. Avoid charts on this screen.

- [ ] **Step 5: Run tests and commit**

```bash
./ops/compose.sh run --rm web bundle exec rspec \
  openproject-video-operations/spec/queries/video_operations/workbench_query_spec.rb \
  openproject-video-operations/spec/features/video_operations/workbench_spec.rb
git add openproject-video-operations
git commit -m "feat: add exception-first operations workbench"
```

### Task 3: Build the P1 project phase-gate overview

**Files:**
- Create: `openproject-video-operations/app/queries/video_operations/project_overview_query.rb`
- Create: `openproject-video-operations/app/controllers/video_operations/projects_controller.rb`
- Create: `openproject-video-operations/app/views/video_operations/projects/index.html.erb`
- Create: `openproject-video-operations/app/views/video_operations/projects/show.html.erb`
- Create: `openproject-video-operations/spec/queries/video_operations/project_overview_query_spec.rb`
- Create: `openproject-video-operations/spec/features/video_operations/project_phase_gate_spec.rb`

**Project page order:**

1. Breadcrumb, project name, owner, current phase, delivery date.
2. `基线与当前预测` strip with baseline, actual, approved change, forecast-at-completion, variance, and margin.
3. Seven-stage horizontal/compact progress indicator using the approved Chinese names.
4. `进入下一阶段前` card with hard blockers first, soft warnings second, and exactly one advance/override action.
5. `本阶段关键工作` using OpenProject work packages.
6. `人员与档期` showing demand coverage and conflicts.
7. `变更与基线` audit timeline.

Keep secondary project tabs/links for `任务 / 时间线 / 资源 / 成本 / 变更 / 文件`; link to native OpenProject work-package, Gantt, cost, and file pages where they already own the data, and to plugin pages only for video-specific resource/change views.

**Interfaces:**
- `ProjectOverviewQuery.call(project:, user:) -> Result` composes `GateEvaluation`, `CostForecast`, OpenProject work-package relations, demand coverage, native OpenProject journals, and recent immutable plugin `AuditEvent` records.
- The page calls Package 2's phase-gate API for advancement; it never reimplements gate rules.

- [ ] **Step 1: Write failing query composition tests**

Stub the service results and prove the query preserves source values, uses only visible work packages, calculates demand coverage counts without mutation, and orders audit events newest first.

- [ ] **Step 2: Write the failing P1 feature spec**

Assert:

- all seven phase names appear in sequence;
- baseline and current forecast appear above the phase/gate detail and remain visually distinct;
- current/completed/future phases have text labels in addition to styling;
- hard blockers disable the advance control;
- soft warnings display a required override-reason field;
- a successful advance updates the phase and baseline timeline;
- a `409` stale response displays `数据已变化，请刷新后重新确认` without changing the page optimistically;
- project list links enter this page rather than generic project overview.

- [ ] **Step 3: Implement the project list and overview**

Use the existing phase-gate and cost service contracts. Render dates in `Asia/Shanghai`, money as `¥` with two decimal digits, and duration in hours/days while retaining minute precision in accessible labels.

- [ ] **Step 4: Run tests and commit**

```bash
./ops/compose.sh run --rm web bundle exec rspec \
  openproject-video-operations/spec/queries/video_operations/project_overview_query_spec.rb \
  openproject-video-operations/spec/features/video_operations/project_phase_gate_spec.rb
git add openproject-video-operations
git commit -m "feat: add phase-gate project overview"
```

### Task 4: Add people, availability, time, cost, report, and settings pages

**Files:**
- Create: `openproject-video-operations/app/queries/video_operations/people_capacity_query.rb`
- Create: `openproject-video-operations/app/controllers/video_operations/people_controller.rb`
- Create: `openproject-video-operations/app/controllers/video_operations/availability_controller.rb`
- Create: `openproject-video-operations/app/controllers/video_operations/time_entries_controller.rb`
- Create: `openproject-video-operations/app/controllers/video_operations/costs_controller.rb`
- Create: `openproject-video-operations/app/controllers/video_operations/reports_controller.rb`
- Create: `openproject-video-operations/app/controllers/video_operations/settings_controller.rb`
- Create: corresponding index/show ERB files from the planned map
- Create: `openproject-video-operations/spec/features/video_operations/people_and_costs_spec.rb`

**Page responsibilities:**

- `人员`: crew name, employment type, roles, skills, active state; no login/account implication.
- `档期`: weekly capacity matrix with confirmed, unavailable/leave, proposed, and remaining capacity states, plus a separate `手工安排` action that remains usable when Timefold is unavailable.
- `工时`: OpenProject time entries plus plugin assignment actual minutes; v1 supports review/filter, not a second time-entry store.
- `成本`: project rows with baseline, actual, forecast-at-completion, margin, and variance status.
- `报表`: stage throughput, on-time delivery, utilization, forecast accuracy, and change count from authoritative data.
- `模板与设置`: phase requirement mapping, roles, skills, locations/travel times, rates, and pending closeout-derived template adjustments; local-admin only. Every template proposal shows current vs suggested minutes/cost and requires `应用建议` or `拒绝建议`.

- [ ] **Step 1: Write failing people/capacity query tests**

Use a fixed week and prove half-open interval math, leave display, production fixed blocks, post allocated minutes, remaining daily capacity, inactive-person filtering, and no double-count of OpenProject time entries.

- [ ] **Step 2: Write failing page feature tests**

Assert the N2 active state, filters persisted in query parameters, Chinese empty states, CNY/time formatting, visible source labels (`OpenProject 工时` vs `排程分配`), and settings authorization. Stop/mock the scheduler and prove `手工安排` can still create a confirmed/locked assignment through the Package 2 API; named overlap/availability/skill conflicts remain visible and write nothing. Show a closeout-derived template proposal side by side, prove the underlying template remains unchanged before confirmation, and exercise apply/reject/stale responses through the Package 2 endpoint.

- [ ] **Step 3: Implement compact operational tables**

Use OpenProject form controls and tables. Prefer downloadable CSV links that call server-side report endpoints over client-side data export. Do not add editable spreadsheets or bidirectional Excel sync.

- [ ] **Step 4: Run tests and commit**

```bash
./ops/compose.sh run --rm web bundle exec rspec \
  openproject-video-operations/spec/queries/video_operations/people_capacity_query_spec.rb \
  openproject-video-operations/spec/features/video_operations/people_and_costs_spec.rb
git add openproject-video-operations
git commit -m "feat: add video operations object views"
```

### Task 5: Build the D2 before/after schedule comparison

**Files:**
- Create: `openproject-video-operations/app/queries/video_operations/schedule_diff_query.rb`
- Create: `openproject-video-operations/app/controllers/video_operations/schedules_controller.rb`
- Create: `openproject-video-operations/app/helpers/video_operations/schedule_helper.rb`
- Create: `openproject-video-operations/app/views/video_operations/schedules/show.html.erb`
- Create: `openproject-video-operations/app/views/video_operations/schedules/_diff_dialog.html.erb`
- Create: `openproject-video-operations/frontend/module/stimulus/schedule_diff.controller.ts`
- Create: `openproject-video-operations/spec/queries/video_operations/schedule_diff_query_spec.rb`
- Create: `openproject-video-operations/spec/features/video_operations/schedule_diff_spec.rb`

**D2 layout:**

- Header: project, proposal creation time, score, current/proposal data versions, feasibility status.
- Left calendar: `当前排程`.
- Right calendar: `建议排程`.
- Identical day/time scale and worker rows on both sides.
- Changed assignments visually connected by stable worker/demand labels; additions, removals, moves, and unchanged work have distinct text/icon states.
- Impact summary: changed people, moved work, uncovered demand, overtime, travel, fragmentation, and estimated labor-cost delta.
- Bottom actions: secondary `返回调整/重新排程`; primary `整案应用`.

**Interfaces:**
- `ScheduleDiffQuery.call(proposal:, current_version:) -> Result` returns normalized `before_rows`, `after_rows`, `changes`, `impact`, `stale`, and `applicable`.
- `schedule_diff.controller.ts` manages dialog focus, confirmation checkbox, apply request, and stale/error response rendering only.
- The controller sends `expected_data_version`; it does not choose or edit individual assignments.

- [ ] **Step 1: Write failing diff-query tests**

Cover added, removed, moved-time, changed-worker, unchanged, locked/manual, cross-midnight, production fixed block, post daily capacity, infeasible, and stale cases. Assert stable sort by date/start/worker/demand.

- [ ] **Step 2: Implement a pure normalized diff**

Compare the proposal's stored request/current-alternative against its stored response. Never reconstruct the original state from today's database because it may have changed; compare today's version separately to set `stale`.

- [ ] **Step 3: Write the failing JavaScript-enabled feature spec**

Assert:

- both calendars use identical visible dates and worker ordering;
- every change type has a Chinese label;
- a stale proposal disables `整案应用` and offers refresh/re-solve;
- an infeasible proposal shows violations and has no apply control;
- the confirmation dialog states that all changes apply together;
- apply stays disabled until `我已核对全部变更` is checked;
- a successful apply replaces the view with `排程已应用` and the new version;
- `409` and `422` responses leave the calendars visible with actionable Chinese errors;
- there is no assignment checkbox or partial-apply endpoint invocation.

- [ ] **Step 4: Implement the D2 view and focused Stimulus controller**

Use a native/accessibly wrapped dialog, trap focus via OpenProject's existing dialog component, restore focus on close, and submit through the Package 3 endpoint. Do not add a client state library.

- [ ] **Step 5: Run query, feature, and TypeScript build checks**

```bash
./ops/compose.sh run --rm web bundle exec rspec \
  openproject-video-operations/spec/queries/video_operations/schedule_diff_query_spec.rb \
  openproject-video-operations/spec/features/video_operations/schedule_diff_spec.rb
./ops/compose.sh build web
```

- [ ] **Step 6: Commit**

```bash
git add openproject-video-operations
git commit -m "feat: add whole-plan schedule comparison"
```

### Task 6: Verify accessibility, responsive behavior, and visual integration

**Files:**
- Create: `openproject-video-operations/frontend/module/global_styles.scss`
- Create: `openproject-video-operations/spec/features/video_operations/accessibility_spec.rb`
- Modify: all Video Operations ERB views as required by failing checks
- Modify: `README.md`

- [ ] **Step 1: Add failing semantic/accessibility assertions**

Verify unique page title/H1, heading order, landmark labels, keyboard-reachable actions, focus restoration, visible focus, table headers/scope, form labels/errors, non-color status labels, and no duplicate element IDs. Use existing OpenProject accessibility helpers if available; otherwise keep assertions at DOM/keyboard behavior level and do not introduce a new scanner dependency.

- [ ] **Step 2: Add the smallest plugin styles**

Only style the phase strip, exception list density, capacity/calendar grid, and diff change markers missing from OpenProject. All colors, spacing, radii, typography, and shadows must reference OpenProject variables/tokens. No global selector may alter pages outside `.video-operations`.

- [ ] **Step 3: Exercise the approved desktop breakpoints**

Run feature/system specs at `1440×900` and `1024×768`. At 1024 px:

- N2 remains accessible;
- exception actions remain visible without horizontal page scrolling;
- P1 cards stack predictably;
- D2 calendars use one contained horizontal scroller with synchronized headers.

- [ ] **Step 4: Run the complete UI and plugin suites**

```bash
./ops/compose.sh run --rm web bundle exec rspec openproject-video-operations/spec/features
./ops/compose.sh run --rm web bundle exec rspec openproject-video-operations/spec
./ops/compose.sh build web
./ops/up.sh
./ops/smoke.sh
```

Expected: all tests pass, assets compile, and the live local instance serves the plugin pages.

- [ ] **Step 5: Manual visual acceptance checklist**

Open `http://localhost:8080/video_operations` and check:

1. A1 decisions/risks are the first visible content.
2. N2 order and Chinese wording exactly match the approved labels.
3. Every project link opens P1.
4. D2 shows the same calendar scale before/after and only whole-plan actions.
5. No untranslated visible plugin string or duplicate project/work-package editor appears.

- [ ] **Step 6: Commit**

```bash
git add openproject-video-operations README.md
git commit -m "test: verify Chinese operations interface"
git status --short
```

Expected: clean working tree.

## Package Completion Criteria

- The UI is a single authenticated OpenProject website with one Chinese Video Operations module.
- A1/N2/P1/D2 behavior matches the approved visual decisions exactly.
- Workbench exceptions and decisions precede summary reporting.
- Project navigation lands on the seven-stage P1 page.
- People, availability, time, cost, report, and settings pages expose authoritative data without duplicating storage.
- D2 uses the proposal's stored before/after snapshots, detects staleness, and offers no partial apply.
- Server-side authorization, Chinese errors/empty states, keyboard behavior, and desktop breakpoints are tested.
- Full feature/plugin suites, asset build, and local smoke checks pass.
