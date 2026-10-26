# Design decisions — 2026-10-26

Decisions taken with Fernando while discussing how to audit PRD task plans. They are the starting
point for hannibal's own PRD. Nothing is implemented yet.

## Origin

While reviewing a real PRD (a coursework submission system), a planning error went unnoticed by
two adversarial reviewers: the first milestone included a command whose task depended on a task
of the second milestone. It is a mechanical error that a script detects for free. The audits we
wanted to run on every PRD task plan were:

| #   | Check                                                                                              | Who                         |
| --- | -------------------------------------------------------------------------------------------------- | --------------------------- |
| a   | Task order is correct: dependencies acyclic, listed in order, no task depends on a later milestone | script                      |
| b   | Task size: small but not fanatically so (see rule below)                                           | model, with script warnings |
| c   | Which tasks need expensive audits afterwards (`/types`, `/improve`, AP patterns)                   | script, derived             |
| d   | Which tasks need prior skills (`/safety-preflight`, `/groundedness`)                               | script, derived             |
| e   | Every task has an owner: agent or a named human                                                    | script                      |
| f   | Human tasks grouped and placed as late as possible                                                 | script                      |
| g   | Which tasks need a security review (`/owasp`, `/infra-sec`)                                        | script, derived             |
| h   | Consolidate expensive reviews so none runs more often than needed                                  | script                      |
| i   | Traceability: every user story has its e2e test; every incident has its guarantee or test          | script                      |

**Task size rule.** A task is too small if it has no test of its own that closes it, and too big if
its tests can fail for unrelated reasons. The script only warns (e.g. more than ~8 acceptance
criteria, or mixing domain, infrastructure and UI); a model other than the author judges.

**Human tasks.** Separate *asking* from *needing*: ask as early as possible (other people's latency
is unpredictable), need as late as possible (right before the first task that uses the result),
and group the owner's approvals into one or two sessions.

## Declare facts, derive obligations

A task declares one fact, `touches` (surfaces: domain, auth, email, personal data, infrastructure,
UI…). A single table maps surfaces to reviews. Declaring reviews by hand would create a second
source of truth.

- **Prior reviews** run once, before the **first** task that touches the surface.
- **Later reviews** run at the end of the stage (`/improve`, `/types`) or once per milestone before
  production (`/owasp`, `/infra-sec`), covering everything that touched the surface.

The model's role: detect a missing `touches` value (a task that sends email without declaring it).

## The CLI is the only interface

Options considered for writing the PRD:

| Option                                     | Rejected because                                                                                                                                                                                                                                       |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Markdown with typed fields parsed by regex | Fragile: a misplaced bold silently drops a field                                                                                                                                                                                                       |
| Separate YAML plan + Markdown prose        | Two sources of truth: prose can contradict the data                                                                                                                                                                                                    |
| Markdown with embedded YAML blocks         | The model still writes the format by hand; a bad edit lands and is fixed later                                                                                                                                                                         |
| Generated Markdown from a YAML source      | Prose inside YAML is awkward to write                                                                                                                                                                                                                  |
| TOON as the writing format                 | Tasks are not uniform rows (no savings), models write it less reliably than YAML/Markdown, and the spec is a working draft                                                                                                                             |
| Linear (via `lql`) as storage              | Saves only the cheap part (storage); loses write-time rejection (anyone can edit in Linear's UI), versioning with the code, offline work, and it fills the tracker with non-work items. Linear stays as a future **export target** for execution state |

**Decision:** the agent never reads or writes the data directly. It talks to the CLI, like `lql`
for Linear. Each consumer gets the form it handles best:

| Who                  | Does                                     | Format                                          |
| -------------------- | ---------------------------------------- | ----------------------------------------------- |
| Agent writing        | Runs commands                            | Fields as arguments; prose as Markdown on stdin |
| Script               | Validates and stores                     | Canonical JSON (below)                          |
| Agent reading        | Asks for slices (`show`, `list`, `plan`) | TOON, `--json`                                  |
| Humans and reviewers | Read the whole PRD                       | Generated Markdown                              |

Prose is also stored through the CLI (a `body` field per item; context sections are items too), so
nothing lives outside the single source. References in prose (`T11`, `HU-A3`) must resolve.

**Preventing bypass** (same strength as `lql`, whose API token could also be used directly):
`deny` rules in `.claude/settings.json` for Read/Edit/Write on the store, plus `check` in a git
pre-commit hook.

## Commands

Read (never modify): `schema [kind]`, `show <id>`, `list [kind] [filters]`, `refs <id>`,
`check [--strict]`, `plan`.

Write: `init <feature>`, `add <kind> [id] --field value … [--body-file -]`,
`set <id> --field value … [--body-file -]`, `rename <id> <new>`, `rm <id>` (refuses while
referenced), `validate <id> --by <who> [--note …]`.

Rules for every write:

1. **All or nothing.** An error stores nothing and answers with the cause and a fix.
2. **Immediate feedback.** A successful write also returns the remaining pending items.
3. **Every write regenerates the Markdown view.** No `render` command; `check` only has to detect
   hand edits.
4. **Changing a validated item returns it to draft**, and the answer says so.

Two levels of findings: **errors**, rejected at write time (invalid enum, unknown field, cycle,
dependency on a later milestone); **pending**, allowed while drafting and listed (story without
test, reference to an item not yet created, task without `touches`). `check --strict` requires
zero pending items before review.

IDs: free (given by the author) or assigned by the CLI (next number per kind). Both are allowed;
order comes from dependencies, not from the number.

Exit codes: 0 ok, 1 write rejected, 2 usage error. With a single PRD in the repo it is used by
default; with several, `--prd <feature>` is required.

Out of the first version: `apply` (batch, all-or-nothing; a shell script of `add` lines is enough
to import an existing PRD) and `export linear`.

## Storage format: canonical JSON

Nobody edits the file by hand, so editing comfort no longer matters. What matters:

1. **Reliable reading and writing in Rust.** `serde_json` is solid; `serde_yaml` was archived by
   its author in 2024 and the forks offer fewer guarantees. JSON has no `no` → `false` trap.
2. **Stable git diffs.** Fixed indentation, fields in struct order, items sorted by kind and id,
   trailing newline: the same content always produces the same bytes.
3. **Readable prose diffs** come from the committed generated Markdown, not from the JSON.
4. **Recoverable without the tool** (`jq`, any language).

Discarded: YAML (above), TOML (poor for nested data and lists of long texts), TOON (draft spec,
no gain as storage; kept for output), SQLite (binary in git; atomic write via temp file + rename
covers transactions here).

Details: `docs/plans/<feature>.prd.json` (source) and `docs/plans/<feature>-prd.md` (generated);
`format_version` from day one.

## Schema

A JSON Schema is part of the contract. Whether it is hand-written (source of truth, Rust types
derived) or generated from the Rust types with `schemars` is still open. Either way there is one
source, and cross-item invariants (cycles, milestone order, traceability) live in code because
JSON Schema cannot express them.

Starting item kinds, taken from the real PRD: context sections, user stories, rules, incidents,
open decisions, milestones, tasks. To be refined while importing that PRD.

## Language and stack

- Rust, like `lql`: `clap`, `serde`, `toon_format`; a single binary.
- Everything in English: kinds, fields, enum values, commands, output, error messages. Content
  (titles and prose) stays in the PRD's own language.

## Agentic experience requirements

From AP's agentic experience adapter (`framework.md`) and `lql`:

- **Impossible, not forbidden:** the only path to the data is the CLI.
- **Fail at write time**, never later; every error carries `help:` with the exact fix command.
- **Tolerant of invocation, strict on content:** aliases (`show`/`get`/`view`), positional or flag
  arguments, argument order, enum case. An unknown value is never guessed: error with
  "did you mean …?".
- **Self-describing:** `schema` teaches the interface without docs.
- **Structured output:** TOON by default, `--json`, fixed exit codes.
- **Tests from real sessions:** every real invocation that fails becomes a test (like `lql`'s AX-nn).

## Harness integration: a skill that helps, hooks that enforce

A skill is an instruction to the model; it can be ignored. Enforcement belongs to the harness
(AP: *impossible, not forbidden*). Both ship in this repo:

1. **Skill `/hannibal` (helps).** When to use it and the full flow: create, validate with the
   owner, `check --strict`, `plan`, audit. It does not restate rules or values; it points to
   `hannibal schema`, so it is not a second source of truth. Lives in `skills/hannibal/SKILL.md`,
   symlinked into `~/.claude/skills/`.
2. **Hooks (enforce).**
   - A PreToolUse hook rejects Edit/Write on any file carrying the "generated by hannibal" header,
     answering with the command to use instead.
   - The same hook rejects writing a `docs/plans/*-prd.md` without that header: no hand-written PRDs.
   - `hannibal init` installs the project's `deny` rules and the pre-commit `check`, so protection
     does not depend on remembering to configure it.

Once hannibal works, the "PRDs" section of the global `CLAUDE.md` must point to it instead of
describing the format by hand.

## Next

1. Define the item kinds and their fields (the JSON Schema).
2. First version: schema, `add`, `set`, `show`, `list`, `check`, generated Markdown.
3. Acceptance test: import the real PRD; hannibal must reject the milestone error by itself.
4. Then `plan` (human sessions, review placement) and the model-judged audit.
5. Skill and enforcement hooks (`hannibal init` installs them).
6. Later: extend the tool to the implementation phase (to be discussed).
