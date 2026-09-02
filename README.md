# HRPM Video Operations

Private handoff and implementation repository for a local, single-user video-department project and resource management system.

The approved architecture combines:

- OpenProject as the running project/work-package system and source of project truth.
- A dedicated Video Operations plugin for crew, skills, availability, rates, phase gates, assignments, costs, audits, and Chinese operational pages.
- Timefold Solver as a stateless local scheduling service.
- PostgreSQL as the only production database.
- Optional DeepSeek assistance after the core system is stable; it is not part of the first implementation sequence.

## Start here

1. Read [HANDOFF.md](HANDOFF.md).
2. Read [AGENTS.md](AGENTS.md) before any Codex implementation turn.
3. Review [current status](STATUS.md) and [confirmed decisions](DECISIONS.md).
4. Read the [approved specification](docs/superpowers/specs/2026-09-01-video-operations-openproject-timefold-design.md).
5. Execute the [program roadmap](docs/superpowers/plans/2026-09-02-video-operations-program-roadmap.md) package by package.

## Public interface prototype

- Demo: <https://raynacocobobobo-arch.github.io/HR/>
- Source: <https://github.com/raynacocobobobo-arch/HR>

The demo is a static visual prototype. It is not connected to OpenProject, Timefold, DeepSeek, a database, or real company data.

## Privacy

This repository is private. Do not make it public. Never commit legacy source archives, real JSON/Excel data, WeCom exports, credentials, API keys, database dumps, attachments, or generated runtime secrets.

