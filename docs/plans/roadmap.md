# Roadmap

## Stage 1 — Build and audit PRDs

1. **Item kinds and fields (JSON Schema).** Derive them by importing the real fixture PRD; every
   section of it says which fields are needed. Recommended: Rust types are the source, the schema
   is generated with `schemars`, committed, and a test fails if they diverge.
2. **First version:** `schema`, `add`, `set`, `show`, `list`, `check`, generated Markdown, canonical
   JSON store.
3. **Acceptance test:** import the fixture PRD; hannibal rejects the milestone error (ORD-3) by
   itself.
4. **`plan`:** order, human sessions, ask-now list, review placement.
5. **Model-judged audit** (the *judgment* rules in [`audit-rules.md`](audit-rules.md)).
6. **Skill `/hannibal` and enforcement hooks**, installed by `hannibal init`.
7. Update the "PRDs" section of the global `CLAUDE.md` to point to hannibal.

## Stage 2 — Commands used while implementing a PRD

To be discussed. Execution state, stage gates, review runs, export to Linear.

## Dogfooding

hannibal's own PRD is built with hannibal, as far as possible.

- **Bootstrap:** until `add`, `set` and `check` exist, the design lives in the Markdown docs of
  this repo. The first thing the first version does is import them into `docs/plans/hannibal.prd.json`.
- **Written in English**, titles and prose included: the repo is public.
- **From then on**, every change to hannibal's plan goes through `hannibal`; stage 2 is planned
  with it.
- **Second fixture, public by construction:** hannibal's own PRD needs no sanitizing, so it is the
  natural fixture to commit while the real one is private.
- Every friction found while dogfooding (a failed invocation, a confusing error) becomes an AX test.

## Fixtures

The first fixture is the PRD of a real, private project (a coursework submission system, written
in Spanish), kept outside this repo. It must be converted into
correct hannibal JSON.

- **Golden fixture:** the converted PRD, after fixing its errors; `check --strict` passes.
- **Regression fixture:** the same PRD at its first committed version, which contains the ORD-3 error (milestone M1
  includes a command in T11, which depends on T10 of milestone M2).
- **Derived fixtures:** one per rule in `audit-rules.md`, each the golden fixture with exactly one
  violation and the expected finding. A rule without a failing fixture is not tested.

**Open: the source PRD is private and this repo is public.** It cannot be committed
as is (names, emails, internal hosts and processes). Options: a sanitized fixture (same structure,
neutral names and data), or keep the real fixture outside the repo and commit only sanitized
derivatives. Decide before committing any fixture; run `opsec-scan --tree` before every push.
