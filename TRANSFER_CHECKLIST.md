# New Mac Transfer Checklist

## GitHub handoff

- [ ] Log into GitHub as `raynacocobobobo-arch` on the new Mac.
- [ ] Clone the private `HRPM-Video-Operations` repository.
- [ ] Confirm `main` is clean and `HANDOFF.md` is present.
- [ ] Open the public demo and confirm the approved visual direction.

## Hardware and runtime

- [ ] Confirm `uname -m` returns `arm64`.
- [ ] Record memory and logical CPU count.
- [ ] Confirm at least 30 GiB free on `/System/Volumes/Data`.
- [ ] Confirm Homebrew status.
- [ ] Record whether Docker, Podman, or Colima already exists.
- [ ] Do not install until the Package 1 preflight is approved.

## Private legacy materials

- [ ] Transfer the legacy HRPM source archive privately.
- [ ] Transfer actual HRPM JSON data privately.
- [ ] Transfer the current scheduling Excel workbook privately.
- [ ] Transfer required attachments privately.
- [ ] Keep all sources read-only and calculate SHA-256 checksums.
- [ ] Do not place any of these materials in GitHub.

## Backup readiness

- [ ] Identify a second mounted volume or Time Machine destination.
- [ ] Record the future `VIDEO_OPS_BACKUP_ROOT` outside the repository.
- [ ] Do not go live until an isolated restore drill passes.

## First Codex turn

- [ ] Ask Codex to read `AGENTS.md` and `HANDOFF.md` completely.
- [ ] Ask for a read-only preflight report before implementation.
- [ ] Approve Package 1 only after reviewing the preflight.

