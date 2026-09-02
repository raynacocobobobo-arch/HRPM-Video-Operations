# Confirmed Decision Ledger

## Product and deployment

- Use the system only on one Mac in version 1.
- One user now; revisit multi-user access later.
- One Chinese website; the user does not operate multiple admin systems.
- Keep all browser-facing services on loopback.
- Use a new Mac because the original Mac lacks safe free space.

## Open-source composition

- OpenProject is the running project-management foundation.
- Create a dedicated Video Operations plugin; do not fork OpenProject core.
- Share one PostgreSQL instance; do not create a shadow project database.
- Timefold is a stateless local scheduling service with no direct database access.
- ERPNext is deferred.
- Plane, Leantime, Kimai, and ERPNext may inform management ideas but are not parallel systems of record.

## Workflow

- Seven stages: 项目准入 / 前期策划 / 拍摄准备 / 拍摄执行 / 后期制作 / 审片修改 / 交付关闭.
- Phase gates have hard requirements, soft warnings, and audited human overrides.
- Preserve the original baseline; changes produce current forecasts and change records.
- Project closeout generates a retrospective.
- Template improvement suggestions require explicit confirmation.

## Scheduling

- Shoots use exact time blocks, location, travel, working-time, and rest constraints.
- Post-production uses daily capacity, remaining effort, deadlines, and critical paths.
- Timefold proposes but never writes official assignments.
- D2 shows current and proposed weekly calendars side by side.
- Apply the entire current proposal or reject/re-solve; no partial application.
- Reject stale proposals and apply accepted proposals transactionally.
- Manual scheduling remains available when Timefold is down.

## Information architecture

- A1: exception-first workbench.
- N2: object navigation.
- P1: phase-gate project default page.
- D2: before/after schedule comparison.
- Navigation: 工作台 / 项目 / 人员 / 档期 / 工时 / 成本 / 报表 / 模板与设置.

## Cost and data

- Keep planned, actual, and forecast costs separate.
- Save confirmed rate snapshots so later standard-rate changes do not alter history.
- Excel is for one-time import and optional export, not bidirectional synchronization.
- Migration is dry-run-first, visible, idempotent, and never auto-deletes target records.
- Original HRPM becomes read-only after acceptance.

## Deferred or excluded

- Wedding CRM, hospital publishing ledgers, and lighting inventory.
- Multi-user roles, SSO, mobile app, payroll, attendance, and general HRIS functions.
- ERPNext finance/purchasing/invoicing/collections.
- DeepSeek in the core implementation sequence.

## Optional DeepSeek boundary

DeepSeek may later provide summaries, explanations, natural-language queries, report drafts, and retrospective suggestions. It must sit behind the local backend, never expose a key in browser code, and never perform authoritative writes. Timefold remains responsible for schedule optimization.

