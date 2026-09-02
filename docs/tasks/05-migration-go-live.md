# Package 5: Migration, recovery, and local go-live

## Objective

Perform a dry-run-first, human-resolved, idempotent migration and prove backup/restore before local go-live.

## Source plan

`docs/superpowers/plans/2026-09-02-video-operations-migration-go-live.md`

## Dependency

Package 4 completed and the browser flow approved.

## Completion evidence

- User explicitly selects the legacy source path.
- Dry-run report shows created, updated, duplicate, unresolved, and rejected records.
- No source or target records are automatically deleted.
- Re-applying the same bundle is idempotent.
- Database/assets backup is copied to a second volume and checksummed.
- Isolated restore drill passes with matching business and attachment counts.
- Final website is available only at the approved loopback URL.

## Stop gates

Obtain explicit approval for the dry-run report and again for the restore drill before go-live.

