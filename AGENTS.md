# Instructions for Codex and other implementation agents

## Required reading order

Before any implementation action, read these files completely:

1. `HANDOFF.md`
2. `STATUS.md`
3. `DECISIONS.md`
4. `docs/superpowers/specs/2026-09-01-video-operations-openproject-timefold-design.md`
5. `docs/superpowers/plans/2026-09-02-video-operations-program-roadmap.md`
6. The package plan currently being executed

Treat the current clone root (`git rev-parse --show-toplevel`) as the repository root. Any historical absolute path in an analysis report is evidence only, never an implementation target.

## Execution rules

- Formal implementation has not started. Begin with Package 1 and the read-only preflight.
- Execute packages in order; do not overlap package implementation.
- Respect every user checkpoint in the roadmap.
- Use test-driven development for production behavior: failing test, minimal implementation, passing test, refactor.
- Prefer the Superpowers `subagent-driven-development` or `executing-plans` workflow when available.
- Make small commits matching the task boundaries in the package plans.
- Do not change pinned versions without recording the reason and obtaining user approval.
- Do not add unrequested infrastructure or features.
- Do not treat the public static demo as production code or a backend implementation.

## Data and security

- This repository must remain private.
- Never commit `.env`, tokens, passwords, API keys, real HR data, database dumps, attachments, WeCom files, legacy source archives, or generated secrets.
- Do not print secret values in tests, logs, screenshots, or migration reports.
- Never broaden a bind address beyond loopback.
- Do not expose the Timefold service to the host network.
- Do not let DeepSeek or any LLM perform authoritative writes.
- Never modify the legacy HRPM source or source data; migration reads a user-selected exact path.

## Scope

Version 1 is local, single-user, and Chinese. Wedding CRM, hospital ledgers, lighting inventory, ERPNext, payroll, attendance, multi-user roles, and Excel bidirectional sync are excluded.

## Completion claims

Do not claim a task or package is complete without fresh command output proving its tests and acceptance checks. Keep the repository clean at handoff boundaries and report the exact commit.

