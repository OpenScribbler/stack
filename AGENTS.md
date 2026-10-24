# OpenScribbler stack

Decisions that bind ACIF, capmon, MOAT and syllago together. `README.md` holds them; `specs/` holds the specs for slices that cross projects.

## Rules

- No mutation testing while the stack's first implementation (syllago v2) is built, which overrides the global rule to mutation-test each stage.
- Merge a PR without asking once its CI checks pass and both second-opinion rounds are done, with the second round's fixes pushed. This repo runs no CI, so a PR here waits only on the two rounds.

## Agent skills

### Issue tracker

Issues live in this repo's GitHub Issues; specs are committed under `specs/` and linked from an issue. See `docs/agents/issue-tracker.md`.

### Triage labels

The five default triage labels, unchanged. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context, with each project's lexicon governing its own terms. See `docs/agents/domain.md`.
