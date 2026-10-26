# CLAUDE.md — hannibal

CLI to build, validate and audit PRDs and task plans, designed for AI agents. Rust, like `lql`.

## Rules

- **Public repo: everything in English** (code, comments, docs, commits, CLI output, and the content of hannibal's own PRD).
- **Run `opsec-scan --tree` before every push.** No internal hosts, local paths, names, emails or private
  data. The fixture PRD comes from a private project: see "Fixtures" in `docs/plans/roadmap.md`.
- Status: design. Do not write code until the item kinds and fields are agreed.

## Where things are

- `docs/research/2026-10-26-design-decisions.md`: what was decided and why.
- `docs/plans/audit-rules.md`: the rules hannibal verifies.
- `docs/plans/roadmap.md`: stages, next steps and fixtures.
