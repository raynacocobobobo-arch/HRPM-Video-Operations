# HRPM Video Operations Handoff

Updated: 2026-09-02 (Asia/Shanghai)

## 1. Why this handoff exists

The original Mac does not have enough free disk space for the approved OpenProject + PostgreSQL + Timefold stack. Planning and the visual prototype were completed there, but formal implementation was intentionally not started. Work will continue on another Apple Silicon Mac through this private GitHub repository.

## 2. Exact current state

Completed:

- Existing HRPM code was inspected without executing unknown business logic.
- Product needs were reverse-engineered from the legacy code.
- GitHub alternatives were evaluated; OpenProject + a dedicated plugin + Timefold was selected.
- The full design specification was confirmed.
- A program roadmap and five detailed implementation packages were written.
- A public eight-page interface demo was built and verified.
- GitHub Pages demo: <https://raynacocobobobo-arch.github.io/HR/>

Not completed:

- No Colima/Docker runtime was installed for the new system.
- OpenProject was not installed.
- No Video Operations plugin code was created.
- No Timefold service was created.
- No legacy/Excel data was imported.
- No production backup or restore drill exists yet.
- DeepSeek was discussed only as an optional future assistant; it has not been added to the approved first-version scope.

The next implementation action is Package 1, Task 1: run the read-only preflight on the new Mac. Do not skip to plugin, UI, migration, or AI work.

## 3. Approved architecture

```text
Browser
  |
  v
OpenProject 17.7.2 + Video Operations plugin
  |-- OpenProject projects, work packages, time entries, budgets, files
  |-- Plugin crew, skills, availability, rates, phase gates, assignments
  |-- One PostgreSQL 17 source of truth
  |
  | local versioned snapshot / proposal API
  v
Timefold Solver 2.6.0 / Quarkus 3.38.3 / Java 21
  |-- stateless
  |-- no database access
  |-- proposes schedules only
  `-- never writes official assignments
```

The plugin revalidates proposal versions and applies a confirmed whole-plan proposal in one database transaction. Manual scheduling remains available when Timefold is unavailable.

## 4. Confirmed product decisions

- OpenProject is the actual running base, not merely visual inspiration.
- The plugin and OpenProject share PostgreSQL; there is no shadow project table.
- One local Chinese website and one user in version 1.
- Navigation: 工作台 / 项目 / 人员 / 档期 / 工时 / 成本 / 报表 / 模板与设置.
- Workbench: exception-first.
- Project default: seven phase gates.
- Scheduling confirmation: current and proposed calendars side by side.
- Proposal action: apply the whole proposal or reject/re-solve; no partial cherry-picking.
- Scheduling: exact time blocks for shoots; daily capacity for post-production.
- Costs: planned, actual, and forecast remain separate; confirmed rates are snapshotted.
- Migration: dry-run and visible human resolution before one-time apply.
- Excel does not remain a bidirectional system of record.
- Original HRPM becomes a read-only archive after acceptance.
- Wedding CRM, hospital publishing ledgers, and lighting inventory are out of scope.
- ERPNext is deferred until formal finance, purchasing, invoicing, and collections are truly required.

See [DECISIONS.md](DECISIONS.md) for the complete decision ledger.

## 5. Pinned upstreams

| Component | Pin |
|---|---|
| OpenProject | 17.7.2 |
| OpenProject Compose | `1cf58dc832fb803ee44fa7632449ce8f8f2b928f` |
| PostgreSQL | 17 |
| Colima | 0.10.3 |
| Timefold Solver | 2.6.0 |
| Quarkus | 3.38.3 |
| Java | 21 |
| OpenProject prototype plugin reference | `48bd2359632d22c72836798dbea566f9544050fd` |
| Timefold quickstarts reference | `b9abb3bcd417d51cbd972a69744ba9fc81173b7f` |

Before implementation, verify that pinned artifacts remain downloadable. Do not silently replace a pin with `latest`; record and obtain approval for any required version change.

## 6. Execution order

Run the packages sequentially. Each package has its own plan, tests, commits, and user checkpoint.

1. [Local foundation](docs/superpowers/plans/2026-09-02-video-operations-local-foundation.md)
   - New-Mac preflight, private repository foundation, Colima, pinned OpenProject Compose, localhost smoke checks.
2. [Domain and workflow](docs/superpowers/plans/2026-09-02-video-operations-domain-workflow.md)
   - Dedicated plugin, migrations, seven stages, gates, crew, availability, costs, audit, manual scheduling.
3. [Scheduling service](docs/superpowers/plans/2026-09-02-video-operations-scheduling-service.md)
   - Stateless Timefold service, hard/soft constraints, proposal versioning, whole-plan transactional apply.
4. [Chinese operational interface](docs/superpowers/plans/2026-09-02-video-operations-chinese-ui.md)
   - Approved A1 + N2 + P1 + D2 interface inside OpenProject.
5. [Migration and go-live](docs/superpowers/plans/2026-09-02-video-operations-migration-go-live.md)
   - Dry-run import, human resolution, backups, second-volume copy, restore drill, local launch.

The controlling roadmap is [2026-09-02-video-operations-program-roadmap.md](docs/superpowers/plans/2026-09-02-video-operations-program-roadmap.md).

### GitHub task queue

Execute these issues in order:

1. [Package 1: local foundation](https://github.com/raynacocobobobo-arch/HRPM-Video-Operations/issues/1)
2. [Package 2: domain and workflow](https://github.com/raynacocobobobo-arch/HRPM-Video-Operations/issues/2)
3. [Package 3: Timefold scheduling service](https://github.com/raynacocobobobo-arch/HRPM-Video-Operations/issues/3)
4. [Package 4: Chinese operational interface](https://github.com/raynacocobobobo-arch/HRPM-Video-Operations/issues/4)
5. [Package 5: migration and local go-live](https://github.com/raynacocobobobo-arch/HRPM-Video-Operations/issues/5)
6. [Deferred: optional DeepSeek assistant](https://github.com/raynacocobobobo-arch/HRPM-Video-Operations/issues/6)

Do not start issue 6 as part of version 1. Each issue's stop gate must be satisfied before opening implementation work on the next package.

## 7. New Mac resume procedure

### 7.1 Clone and inspect

```bash
gh auth status
gh repo clone raynacocobobobo-arch/HRPM-Video-Operations
cd HRPM-Video-Operations
git status --short
git branch --show-current
```

Expected: clean status and branch `main`.

### 7.2 Give the new Codex this first prompt

```text
Read AGENTS.md and HANDOFF.md completely. Then read STATUS.md, DECISIONS.md,
the approved design spec, and the program roadmap. Do not install anything
or edit code yet. Verify the new Mac architecture, memory, logical CPUs,
free space on /System/Volumes/Data, Homebrew, GitHub authentication, and the
absence or presence of Docker/Colima. Report the preflight and ask for the
Package 1 execution confirmation required by the roadmap.
```

### 7.3 Hard preflight gate

The new Mac must have:

- Apple Silicon (`arm64`).
- At least 30 GiB free on `/System/Volumes/Data` before installing or pulling images.
- GitHub access to this private repository.
- A confirmed backup target on another mounted volume before go-live.

Do not delete user files to satisfy the disk gate. Stop and report if the gate fails.

## 8. Materials that must be transferred privately

The following are intentionally not in GitHub and must be copied directly to the new Mac or an encrypted external drive:

- The legacy HRPM source archive.
- The actual `video_schedule_data.json` or successor data file.
- The current Video Department scheduling Excel workbook.
- Any attachments required for migration verification.
- Any API token or future DeepSeek key, transferred through a password manager rather than a file in this repository.

Before migration, calculate and record source checksums. Keep the source read-only. Never search the entire home directory for legacy data; the user must select the exact source path.

See [TRANSFER_CHECKLIST.md](TRANSFER_CHECKLIST.md).

## 9. DeepSeek decision

DeepSeek is optional and deferred until the core project, scheduling, cost, backup, and restore workflow is stable.

If later approved:

- Prefer a provider adapter so local Ollama and the official DeepSeek API can be switched without changing business logic.
- Never place a DeepSeek API key in the browser or GitHub Pages.
- A local 7B/8B quantized model is the sensible starting point for a 16GB Mac, but it adds model storage and memory pressure.
- Use it only for summaries, explanations, natural-language queries, report drafts, and retrospective suggestions.
- It must not apply schedules, edit rates, approve gates, change budgets, or update templates without explicit human confirmation.

Timefold remains the deterministic scheduling engine. DeepSeek is a text assistant, not the source of truth.

## 10. Security and scope boundaries

- This private repository may contain internal analysis and plans but no credentials or real HR data.
- Bind the live website only to `127.0.0.1`.
- The Timefold service remains internal to Compose and receives no database credentials.
- Preserve manual scheduling during solver outages.
- Do not add multi-user permissions, ERPNext, wedding CRM, hospital ledgers, lighting inventory, payroll, attendance, or Excel bidirectional sync in version 1.
- Do not modify or delete the legacy HRPM during implementation or migration dry runs.

## 11. Definition of completion

The system is not complete until all nine completion conditions in the program roadmap pass, including clean-checkout tests, whole-plan stale-safe apply, manual scheduling fallback, idempotent migration, verified second-volume backup, and isolated restore drill.
