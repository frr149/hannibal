# hannibal

> "I love it when a plan comes together."

A CLI that builds, validates and audits product requirement documents (PRDs) and their task plans,
designed to be driven by AI agents.

**Status:** design. No code yet. See
[`docs/research/2026-10-26-design-decisions.md`](docs/research/2026-10-26-design-decisions.md).

## Why

When an agent writes a PRD as free text, mechanical mistakes go unnoticed: a task in the first
milestone that depends on a task in a later one, tasks with no owner, user stories with no test,
expensive reviews run twice or never. A careful reader catches some of them; a script catches all
of them, in milliseconds and for free.

hannibal makes the PRD a structured document that agents can only change through the CLI. Every
write is validated on the spot, so invalid plans cannot be stored, and the CLI derives the plan:
execution order, human work sessions and where each review goes.

## Principles

- **Impossible, not forbidden.** The agent has no way to touch the stored data except through
  `hannibal`.
- **Fail at write time.** Errors say what failed, where, which values are valid, and the exact
  command to fix it.
- **Declare facts, derive obligations.** A task declares what it touches; the reviews it needs are
  derived from a single table.
- **Agentic experience.** Tolerant of how agents invoke it, never of what they write; self-describing;
  structured output (TOON by default, `--json`).

Part of the Adversarial Programming toolset.

## License

MIT
