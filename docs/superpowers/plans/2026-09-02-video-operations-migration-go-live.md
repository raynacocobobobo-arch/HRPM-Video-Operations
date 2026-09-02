# Video Operations Migration and Local Go-Live Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Safely move useful HRPM/Excel data into the new authoritative model, let the user resolve differences before any write, prove backup/restore, and publish the finished system only on this Mac at a stable localhost URL.

**Architecture:** A read-only source inventory feeds a canonical import bundle. The plugin computes a persisted preview containing creates, matches, differences, duplicates, unresolved references, and rejected rows. The user resolves real conflicts in the Chinese settings page; apply uses an idempotency key and one transaction per import batch without deleting existing records. Versioned database/assets backups and an isolated restore drill precede local go-live.

**Tech Stack:** Python 3.11, `unittest`, CSV/JSON/SQLite standard libraries, `openpyxl` 3.1.5 for `.xlsx`, OpenProject plugin JSON APIs and Rails transactions, PostgreSQL 17 `pg_dump`/`pg_restore`, POSIX shell, Docker Compose/Colima.

**Spec:** `../specs/2026-09-01-video-operations-openproject-timefold-design.md`

## Package Constraints

- Packages 1–4 must pass before go-live.
- Inventory and preview are always read-only. No import mutation is allowed until the user confirms the visible difference set.
- The legacy source path is supplied explicitly with `--source`; scripts must reject `/`, `/Users`, the home directory, and the repository root as a source target.
- Never rename, delete, normalize in place, or add files to the source directory.
- The canonical bundle contains no executable code and no copied attachment content; attachment metadata is inventoried for a later explicit decision.
- Create new records and update only explicitly approved differences. Never infer that absence from the import means deletion.
- Import is one-time in v1; do not build bidirectional Excel synchronization.
- Do not import wedding CRM, hospital publishing ledger, or lighting inventory data.
- Backup the current database, OpenProject assets, exact version lock, and secret-bearing runtime environment before import. Backups are untracked, local, and mode `0700`/`0600`.
- Restore drill uses separate Compose project names, volumes, and loopback port; it may not touch live volumes.
- Final published addresses bind only to `127.0.0.1`.

---

## Planned File Map

```text
video-operations/
├── requirements-migration.txt
├── ops/
│   ├── migration/
│   │   ├── source_inventory.py
│   │   ├── normalize_legacy.py
│   │   ├── validate_bundle.py
│   │   ├── upload_preview.py
│   │   ├── schemas/canonical-import.schema.json
│   │   └── mappings/video-operations-v1.json
│   ├── backup.sh
│   ├── backup-retention.sh
│   ├── restore.sh
│   ├── restore-drill.sh
│   ├── go-live-check.sh
│   ├── open.sh
│   └── tests/
│       ├── fixtures/migration/
│       ├── test_source_inventory.py
│       ├── test_normalize_legacy.py
│       ├── test_validate_bundle.py
│       ├── test_upload_preview.py
│       ├── test_backup_manifest.py
│       └── test_go_live_check.py
├── backups/                              # ignored local staging; never the only copy
└── openproject-video-operations/
    ├── db/migrate/20260902090500_create_video_operations_import_batches.rb
    ├── app/models/video_operations/import_batch.rb
    ├── app/models/video_operations/import_difference.rb
    ├── app/services/video_operations/imports/
    │   ├── preview.rb
    │   ├── resolve_difference.rb
    │   └── apply.rb
    ├── app/controllers/video_operations/api/v1/import_batches_controller.rb
    ├── app/controllers/video_operations/imports_controller.rb
    ├── app/views/video_operations/imports/show.html.erb
    └── spec/
        ├── services/video_operations/imports/
        ├── requests/video_operations/api/v1/import_batches_spec.rb
        └── features/video_operations/import_review_spec.rb
```

### Task 1: Inventory the selected legacy source without modifying it

**Files:**
- Create: `requirements-migration.txt`
- Create: `ops/migration/source_inventory.py`
- Create: `ops/tests/test_source_inventory.py`
- Create: `ops/tests/fixtures/migration/source/people.csv`
- Create: `ops/tests/fixtures/migration/source/projects.json`
- Create: `ops/tests/fixtures/migration/source/hrpm.sql`
- Modify: `.gitignore`

**Interfaces:**
- `python3 ops/migration/source_inventory.py --source ABSOLUTE_PATH --output work/migration/source-inventory.json`.
- Output contains `inventoryVersion`, `sourceRootHash`, `createdAt`, and ordered `files`.
- Each file entry contains relative path, byte size, UTC modification time, SHA-256, detected type, and structural metadata only: CSV headers/count, JSON top-level keys/count, SQLite table/column/count, XLSX sheet/header/count.
- Exit `2` for unsafe/broad paths, `3` for unsupported or unreadable files, and `0` only after writing a valid inventory.

- [ ] **Step 1: Pin the migration dependency lock**

`requirements-migration.txt` contains exactly:

```text
et_xmlfile==2.0.0
openpyxl==3.1.5
```

Create the migration virtual environment inside ignored `work/migration/.venv`; do not modify system Python.

- [ ] **Step 2: Write failing inventory tests**

Test deterministic ordering/hashes, CSV/JSON/SQLite/XLSX structural inspection, UTF-8 and GB18030 CSV detection, unreadable input, symbolic links escaping the source root, and rejection of `/`, `/Users`, the current home directory, and repository root. Record a pre/post recursive file hash and prove the source tree is unchanged.

- [ ] **Step 3: Implement the read-only scanner**

Open files read-only, never follow directory symlinks, skip known generated directories (`.git`, `node_modules`, `tmp`, `log`) with an explicit warning, and reject files larger than a configurable 2 GiB limit instead of loading them into memory. Do not print row contents, phone numbers, rates, or notes to stdout.

- [ ] **Step 4: Run tests and a real inventory**

```bash
python3 -m venv work/migration/.venv
work/migration/.venv/bin/pip install -r requirements-migration.txt
python3 -m unittest ops.tests.test_source_inventory
work/migration/.venv/bin/python ops/migration/source_inventory.py \
  --source "$video_ops_legacy_source" \
  --output work/migration/source-inventory.json
```

At execution time, set the task-specific shell variable `video_ops_legacy_source` to the exact absolute HRPM/export path explicitly selected by the user, then display and confirm the resolved path before running the scanner. Never guess it or search the entire home directory. Expected: tests pass and the inventory lists only in-scope HRPM/video-operations data. If mixed wedding/hospital/lighting sources are detected, stop before normalization and record them as excluded.

- [ ] **Step 5: Commit code and fixtures, not the real inventory**

```bash
git add requirements-migration.txt .gitignore ops/migration/source_inventory.py ops/tests
git commit -m "feat: inventory legacy sources read only"
```

### Task 2: Normalize supported source data into a canonical import bundle

**Files:**
- Create: `ops/migration/schemas/canonical-import.schema.json`
- Create: `ops/migration/mappings/video-operations-v1.json`
- Create: `ops/migration/normalize_legacy.py`
- Create: `ops/migration/validate_bundle.py`
- Create: `ops/tests/test_normalize_legacy.py`
- Create: `ops/tests/test_validate_bundle.py`
- Create: canonical/malformed fixtures under `ops/tests/fixtures/migration/`

**Canonical entity order:**

1. `roles`
2. `skills`
3. `crewProfiles`
4. `crewSkills`
5. `availabilities`
6. `rateCards`
7. `projects`
8. `workPackages`
9. `resourceDemands`
10. `assignments`
11. `timeEntries`

Every record contains `sourceType`, `sourceFile`, `sourceRow`, `sourceKey`, and `sourceFingerprint`. Cross-record references use canonical source keys until preview resolves them to OpenProject/plugin IDs.

**Interfaces:**
- `normalize_legacy.py --inventory ... --mapping ... --output work/migration/canonical-import.json`.
- `validate_bundle.py PATH` prints one JSON result and exits nonzero for schema or referential errors.
- Dates without time are interpreted in `Asia/Shanghai`; instants are converted to UTC.
- Money must parse to integer CNY cents without floating-point arithmetic.
- Empty values remain null; they are never converted to zero, today, or a guessed employee/project.

- [ ] **Step 1: Write the canonical JSON Schema and failing validator tests**

The schema must reject unknown top-level/entity fields, duplicate source keys, dangling references, invalid stage codes, invalid intervals, non-CNY money, negative durations/rates, and objects from excluded business domains. It must allow an empty array for any entity type and include a `warnings` array for lossy source conversions.

- [ ] **Step 2: Write failing normalization fixtures**

Cover:

- Chinese column aliases for name, role, skills, availability, dates, rates, project, stage, planned/actual time;
- Excel serial dates and text dates;
- duplicate rows with identical fingerprints collapsed with a warning;
- same source key with different values retained as an explicit normalization error;
- employee name collision kept unresolved rather than merged;
- project/work-package hierarchy;
- rate conversion such as `¥1,200/天` to `120000` cents and `daily`;
- exact exclusion of wedding CRM, hospital ledger, and lighting inventory sheets/tables.

- [ ] **Step 3: Implement an explicit mapping file, not heuristics**

`video-operations-v1.json` lists accepted file/table/sheet patterns and source-column aliases for each canonical field. A source structure not named in the mapping is reported and skipped; normalization must never guess by approximate column-name similarity.

After Task 1's real inventory, update this mapping with the exact detected HRPM export/table/sheet names, add a fixture that mirrors their structure without real personal data, make its test fail, and only then implement the adapter.

- [ ] **Step 4: Produce and validate the real canonical bundle**

```bash
work/migration/.venv/bin/python ops/migration/normalize_legacy.py \
  --inventory work/migration/source-inventory.json \
  --mapping ops/migration/mappings/video-operations-v1.json \
  --output work/migration/canonical-import.json
work/migration/.venv/bin/python ops/migration/validate_bundle.py \
  work/migration/canonical-import.json
```

Expected: valid JSON; warnings are reviewable; no excluded-domain entity or attachment content appears.

- [ ] **Step 5: Commit code/mapping/schema and sanitized fixtures only**

```bash
git add ops/migration ops/tests/fixtures/migration requirements-migration.txt
git commit -m "feat: normalize HRPM data into canonical bundle"
```

### Task 3: Create a persisted dry-run preview and visible difference resolution

**Files:**
- Create: `openproject-video-operations/db/migrate/20260902090500_create_video_operations_import_batches.rb`
- Create: `openproject-video-operations/app/models/video_operations/import_batch.rb`
- Create: `openproject-video-operations/app/models/video_operations/import_difference.rb`
- Create: `openproject-video-operations/app/services/video_operations/imports/preview.rb`
- Create: `openproject-video-operations/app/services/video_operations/imports/resolve_difference.rb`
- Create: `openproject-video-operations/app/controllers/video_operations/api/v1/import_batches_controller.rb`
- Create: `ops/migration/upload_preview.py`
- Create: `ops/tests/test_upload_preview.py`
- Create: `openproject-video-operations/spec/services/video_operations/imports/preview_spec.rb`
- Create: `openproject-video-operations/spec/services/video_operations/imports/resolve_difference_spec.rb`
- Create: `openproject-video-operations/spec/requests/video_operations/api/v1/import_batches_spec.rb`

**Schema:**

- `import_batches`: unique `bundle_sha256`, `status` (`previewed`, `resolving`, `ready`, `applied`, `failed`), `summary_payload` JSONB, `created_by_id`, nullable `applied_by_id`, nullable `applied_at`, timestamps.
- `import_differences`: `import_batch_id`, `entity_type`, `source_key`, `classification` (`create`, `exact_match`, `difference`, `duplicate`, `unresolved_reference`, `rejected`), nullable `target_type`, nullable `target_id`, `source_payload` JSONB, `target_payload` JSONB, nullable `resolution` (`keep_existing`, `use_import`, `skip`), nullable `resolved_by_id`, timestamps.
- Source/target payloads include only fields required for comparison; secret/authentication values are prohibited.

**Matching rules:**

- Project: exact imported legacy external ID stored in a plugin custom field; fallback to exact identifier only, never fuzzy name.
- Work package: external ID within resolved project; fallback exact subject/type/date tuple becomes an unresolved choice, not an automatic match.
- Crew: exact legacy external ID; fallback exact normalized phone only when unique; same display name alone is unresolved.
- Roles/skills: exact stable code.
- Other entities: exact parent source key plus natural interval/type key.

**Interfaces:**
- `POST /video_operations/api/v1/import_batches/preview` accepts the canonical bundle and returns batch ID/summary.
- Re-uploading the identical SHA-256 returns the existing batch.
- Preview makes zero domain-table writes.
- `PATCH /video_operations/api/v1/import_batches/:id/differences/:difference_id` accepts one of the three explicit resolutions.
- A batch is `ready` only when every `difference`, `duplicate`, and `unresolved_reference` row has a resolution and every `rejected` row is acknowledged as `skip`.

- [ ] **Step 1: Write failing preview purity and classification specs**

Build one fixture per classification and snapshot domain-table counts/checksums before/after preview. Assert exact summary counts, deterministic ordering, idempotent identical upload, and no model mutation.

- [ ] **Step 2: Implement preview and resolution services**

Perform matching through authoritative visibility-scoped models. Never accept client-supplied target IDs unless they are among server-calculated candidate targets. Record actor and resolution time.

- [ ] **Step 3: Implement and test the uploader**

The Python uploader reads the bundle, computes SHA-256, reads an OpenProject API token from an explicit mode-`0600` `--token-file`, sends one authenticated localhost request, validates the returned summary, and writes `work/migration/preview-result.json`. It must refuse non-loopback URLs and must never print the token.

- [ ] **Step 4: Run focused tests and create the real preview**

```bash
python3 -m unittest ops.tests.test_upload_preview
./ops/compose.sh run --rm web bundle exec rails db:migrate
./ops/compose.sh run --rm web bundle exec rspec \
  openproject-video-operations/spec/services/video_operations/imports \
  openproject-video-operations/spec/requests/video_operations/api/v1/import_batches_spec.rb
work/migration/.venv/bin/python ops/migration/upload_preview.py \
  --bundle work/migration/canonical-import.json \
  --url http://localhost:8080 \
  --token-file work/migration/openproject-token \
  --output work/migration/preview-result.json
```

- [ ] **Step 5: Commit**

```bash
git add openproject-video-operations ops/migration/upload_preview.py ops/tests/test_upload_preview.py
git commit -m "feat: preview import differences without writes"
```

### Task 4: Let the user resolve differences in the website and apply idempotently

**Files:**
- Create: `openproject-video-operations/app/services/video_operations/imports/apply.rb`
- Create: `openproject-video-operations/app/controllers/video_operations/imports_controller.rb`
- Create: `openproject-video-operations/app/views/video_operations/imports/show.html.erb`
- Create: `openproject-video-operations/spec/services/video_operations/imports/apply_spec.rb`
- Create: `openproject-video-operations/spec/features/video_operations/import_review_spec.rb`
- Modify: `openproject-video-operations/app/controllers/video_operations/api/v1/import_batches_controller.rb`
- Modify: `openproject-video-operations/spec/requests/video_operations/api/v1/import_batches_spec.rb`
- Modify: `openproject-video-operations/config/routes.rb`

**UI contract:**

- Entry is `模板与设置 → 数据迁移`.
- Header shows exact counts for new, matched, different, duplicate, unresolved, rejected.
- Default filter is `需要选择`.
- Each conflict shows `现有数据` and `导入数据` side by side, changed fields highlighted with text labels.
- Choices are exactly `保留现有`, `使用导入`, and `跳过此条`.
- `执行导入` stays disabled until all blocking rows are resolved and the user checks `我已核对差异，并已完成导入前备份`.

**Apply interface:**
- `POST /video_operations/api/v1/import_batches/:id/apply` requires `Idempotency-Key` equal to the batch UUID and a matching bundle SHA-256.
- `Imports::Apply.call(batch:, actor:, idempotency_key:) -> Result`.
- Creates/approved updates run in one database transaction in canonical entity order.
- Exact matches write nothing; `keep_existing` and `skip` write nothing; `use_import` updates only fields shown in the difference.
- Absence never deletes or deactivates a target record.
- Project/work-package creates use official OpenProject models/services; plugin entities use plugin services.

- [ ] **Step 1: Write failing human-review feature specs**

Assert side-by-side values, field-level difference markers, filters/counts, required resolutions, disabled apply, actor audit, Chinese validation errors, and no hidden auto-choice. Include a crew name collision and a work-package candidate requiring a visible target selection before choosing the resolution.

- [ ] **Step 2: Write failing transactional/idempotency specs**

Prove ordered creates, approved field-only updates, preserved existing-only rows, no deletes, same-key replay returning the original result, different-key replay rejection, and total rollback when a late time-entry row fails.

- [ ] **Step 3: Implement the UI and apply service**

Lock the batch row, revalidate target fingerprints so a changed target returns `409`, validate the entire execution plan, apply it transactionally, record before/after audit payloads, and mark the batch applied only after all rows succeed.

- [ ] **Step 4: Run tests, then stop for user confirmation**

```bash
./ops/compose.sh run --rm web bundle exec rspec \
  openproject-video-operations/spec/services/video_operations/imports/apply_spec.rb \
  openproject-video-operations/spec/features/video_operations/import_review_spec.rb \
  openproject-video-operations/spec/requests/video_operations/api/v1/import_batches_spec.rb
```

Open the real batch in the browser. Do not call the apply endpoint until the user has resolved the visible differences and explicitly confirms the checked review state.

- [ ] **Step 5: Commit the implementation before applying real data**

```bash
git add openproject-video-operations
git commit -m "feat: review and apply one-time HRPM import"
```

### Task 5: Create versioned backup and restore commands

**Files:**
- Create: `ops/backup.sh`
- Create: `ops/backup-retention.sh`
- Create: `ops/restore.sh`
- Create: `ops/tests/test_backup_manifest.py`
- Modify: `.gitignore`
- Modify: `Makefile`
- Modify: `README.md`

**Backup layout:**

```text
backups/<UTC_TIMESTAMP>-<GIT_SHORT_SHA>/
├── manifest.json
├── SHA256SUMS
├── database.dump
├── assets.tar
├── runtime.env
└── versions.env
```

`runtime.env` is mode `0600`; the backup directory is mode `0700`; `backups/` is ignored. `manifest.json` records backup ID, UTC time, Git commit, image digests, schema migration versions, database name, database dump size, asset count/bytes, replication targets, and file checksums. It never prints secret values.

**Interfaces:**
- `./ops/backup.sh` writes a new directory and prints its absolute path as the final line.
- `VIDEO_OPS_BACKUP_ROOT` must resolve to a user-approved directory on a different mounted volume; repository `backups/` may be staging but cannot be the only successful copy.
- `./ops/backup-retention.sh --dry-run` reports candidates while preserving at least seven daily and four weekly verified backups; deletion requires a separate explicit `--apply` invocation.
- `./ops/restore.sh --backup ABSOLUTE_BACKUP_PATH --compose-project EXPLICIT_NAME --host-port EXPLICIT_LOOPBACK_PORT` restores only into the named target environment.
- Restore refuses the live Compose project name `video-operations` unless `--replace-live` is passed; this plan never passes that flag during verification.

- [ ] **Step 1: Write failing manifest and target-safety tests**

Test required files, permissions, checksum validation, missing/corrupt input, explicit target name/port, rejection of broad paths, refusal to replace live volumes by default, refusal to accept a same-volume backup as the only copy, and the seven-daily/four-weekly retention selection. Retention must protect the newest valid backup, the last successfully restored backup, and every backup not yet verified on the second volume.

- [ ] **Step 2: Implement backup from running containers**

Quiesce writes by stopping `proxy`, `web`, `worker`, `cron`, and `hocuspocus` while leaving PostgreSQL running; install a shell trap that restarts those services on success or failure. Use PostgreSQL 17's `pg_dump --format=custom` through the live `db` service. Stream `/var/openproject/assets` from a one-off `web` container with `tar`; do not inspect or depend on Docker's internal volume path. Copy untracked runtime environment/version locks only into the protected backup directory. Write into a temporary directory beside the final target, rename only after all checksums pass, then copy and verify it under `VIDEO_OPS_BACKUP_ROOT` on the second mounted volume.

- [ ] **Step 3: Implement isolated restore**

Create only explicitly named target volumes/containers, restore schema/data with PostgreSQL 17 `pg_restore`, restore assets into that target application's asset mount, run migrations only if exact pinned application code requires them, and verify checksums before any restore step. Never delete source backups.

- [ ] **Step 4: Run tests and create the pre-import backup**

```bash
python3 -m unittest ops.tests.test_backup_manifest
./ops/backup.sh
./ops/backup-retention.sh --dry-run
```

Record the printed backup path in `work/migration/go-live-evidence.json`; do not commit it.

- [ ] **Step 5: Commit**

```bash
git add ops/backup.sh ops/backup-retention.sh ops/restore.sh ops/tests/test_backup_manifest.py Makefile README.md .gitignore
git commit -m "feat: add safe local backup and restore"
```

### Task 6: Prove recovery with an isolated restore drill

**Files:**
- Create: `ops/restore-drill.sh`
- Create: `ops/tests/test_restore_drill.py`
- Modify: `README.md`

**Interfaces:**
- `./ops/restore-drill.sh --backup ABSOLUTE_BACKUP_PATH` uses Compose project `video-operations-restore-drill` and `127.0.0.1:18080`.
- `./ops/restore-drill.sh --latest` resolves the newest checksum-valid backup already verified on the second volume, then uses the same isolated target.
- The script compares live/restore counts for OpenProject projects, work packages, plugin crew, assignments, baselines, changes, and import batches; compares asset count/bytes; and calls both health endpoints.
- On success it prints a JSON summary and asks/accepts an explicit `--cleanup` flag to remove only the named drill containers/volumes. Without the flag it leaves the isolated drill available for inspection.

- [ ] **Step 1: Write failing restore-drill orchestration tests**

Stub Compose and assert exact project name, loopback port, target volume scoping, count queries, health checks, and absence of live-project `down`, `rm`, or volume commands.

- [ ] **Step 2: Implement the drill and run against the pre-import backup**

```bash
python3 -m unittest ops.tests.test_restore_drill
./ops/restore-drill.sh --latest
curl -fsS http://localhost:18080/health_checks/default
curl -fsS http://localhost:18080/video_operations/api/v1/health
```

Expected: the newest valid second-volume backup is selected, counts/checksums match, and both health endpoints pass.

- [ ] **Step 3: Clean up only after the drill result is recorded**

```bash
./ops/restore-drill.sh --latest --cleanup
```

Expected: only `video-operations-restore-drill` containers/volumes are removed; live stack and backup remain.

- [ ] **Step 4: Commit**

```bash
git add ops/restore-drill.sh ops/tests/test_restore_drill.py README.md
git commit -m "test: prove isolated backup restoration"
```

### Task 7: Apply approved data and publish the localhost site

**Files:**
- Create: `ops/go-live-check.sh`
- Create: `ops/open.sh`
- Create: `ops/tests/test_go_live_check.py`
- Modify: `ops/smoke.sh`
- Modify: `Makefile`
- Modify: `README.md`

**Interfaces:**
- `./ops/go-live-check.sh --import-batch ID --backup ABSOLUTE_PATH` emits one JSON result and exits nonzero unless every gate passes. The safe selectors `--import-batch latest-ready` and `--backup latest` are accepted only when they resolve to exactly one ready batch and one checksum-valid second-volume backup.
- `./ops/open.sh` verifies health and opens `http://localhost:8080/video_operations` in the default Mac browser.
- `make start` starts Colima profile and the pinned Compose stack; `make stop` stops containers without deleting volumes; `make open` opens the site.

**Go-live gates:**

1. `/System/Volumes/Data` has at least 30 GiB free.
2. Git working tree is clean and at a named commit.
3. All pinned versions/digests match `versions.env`.
4. Ruby plugin, Java scheduler, operations, and UI test suites pass.
5. Latest protected backup checksums pass.
6. The backup has a checksum-matching copy on the user-approved second volume, retention preserves at least seven daily and four weekly recovery points, and restore-drill evidence matches that backup.
7. Import batch has no unresolved rows and the user has confirmed the visible review.
8. The live stack publishes exactly one host endpoint, `127.0.0.1:8080`; the scheduler has no host port.
9. OpenProject, plugin, scheduler, PostgreSQL, worker, and cron health checks pass.
10. Exactly one active local administrative user is documented; no public signup or remote access is enabled.

- [ ] **Step 1: Write failing go-live gate tests**

Stub each gate failing once and assert the script stops before import/app startup changes, prints the failed code, and never deletes data. Include a bind-address test that rejects `0.0.0.0`, `::`, LAN IPs, and Docker published ports without a host IP.

- [ ] **Step 2: Implement deterministic gate checks**

The script gathers evidence but performs no fixes. It must not install packages, free disk, resolve imports, create users, or alter firewall/network settings.

- [ ] **Step 3: Run the pre-apply go-live check**

```bash
python3 -m unittest discover -s ops/tests -p 'test_*.py'
./scheduler/mvnw -f scheduler/pom.xml verify
./ops/compose.sh run --rm web bundle exec rspec openproject-video-operations/spec
./ops/go-live-check.sh \
  --import-batch latest-ready \
  --backup latest
```

Expected: the selectors resolve unambiguously and all gates pass. If more than one ready batch exists, no verified second-volume backup exists, disk space is insufficient, or any source difference remains unresolved, stop and report it; do not bypass the gate.

- [ ] **Step 4: Apply the confirmed import exactly once**

Call the apply endpoint through the authenticated browser session after the user checks the visible confirmation. Record batch ID, response checksum, counts, data versions, actor, and time in the batch audit record. Rerun the same request once to prove idempotent response and unchanged row counts.

- [ ] **Step 5: Create a post-import backup and smoke the live site**

```bash
./ops/backup.sh
./ops/smoke.sh
make open
```

Expected browser URL: `http://localhost:8080/video_operations`.

Manually verify with the imported project/crew sample:

- A1 workbench has no unexplained blocker;
- N2 opens all eight pages;
- P1 opens from every project row;
- one nonproduction proposal can be generated and rejected without writes;
- D2 shows before/after and a deliberately stale fixture cannot apply;
- planned/actual/forecast values reconcile to the approved preview;
- restart with `make stop && make start` preserves all records.

- [ ] **Step 6: Finalize operator documentation and commit**

Document only these daily commands at the top of `README.md`:

```bash
make start
make open
make stop
make backup
```

Also document the local URL, backup location, restore drill, pinned version refresh process, and explicit v1 exclusions.

```bash
git add ops/go-live-check.sh ops/open.sh ops/tests/test_go_live_check.py \
  ops/smoke.sh Makefile README.md
git commit -m "docs: publish localhost video operations runbook"
git status --short
```

Expected: clean working tree and the site reachable only at the localhost URL.

## Package Completion Criteria

- The selected legacy source was inventoried read-only with a stable hash and explicit exclusions.
- Canonical normalization is schema-validated, deterministic, and has no guessed identities, money, dates, or references.
- Preview writes no domain data and classifies every row visibly.
- Every real difference is resolved by the user in the Chinese website before apply.
- Apply is field-scoped, non-deleting, transactional, audited, and idempotent.
- Protected backup contains database, assets, version lock, environment, manifest, and checksums, with a verified copy on a second mounted volume.
- Backup retention preserves at least seven daily and four weekly verified recovery points; pruning is dry-run-first and explicitly invoked.
- An isolated restore drill passes without touching the live stack.
- Go-live gates prove tests, pins, backup, restore, import readiness, loopback-only binding, and service health.
- The final site is available at `http://localhost:8080/video_operations` on this Mac only, with the listener bound to `127.0.0.1`.
