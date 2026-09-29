# Migrating from project-harness

For repos set up with `project-harness` (CLAUDE.md routing table + `architecture_docs/` +
`roadmap/`) that have turned into experiment-driven work — e.g. a trading-model project whose
"phases" became a series of evals, audits and retractions.

Migration is **additive and non-destructive**. Never delete the old docs; history is evidence.

## Detect

| Found | Meaning |
|---|---|
| `roadmap/PHASE_N.md` | Old planning layer — archive, don't delete |
| `architecture_docs/arch-*.md` | Keep; they become `docs/` equivalents (or stay in place, see below) |
| An eval log / model history doc (e.g. `experiments/MODEL_HISTORY.md`, `EVAL_*.md`) | Source for LOG, FINDINGS, REJECTED backfill |
| `security_check/`, `Design.md` | Keep as-is if they still apply; not part of this harness |

## Procedure

1. **Read everything first**: CLAUDE.md, the roadmap README, every arch doc header, and the full
   model-history/eval docs. Delegate to a subagent if they're long.
2. **Path choice (ask the human):** keep `architecture_docs/` in place and point the routing table at
   it, or move to `docs/`. Keeping in place is the default — fewer broken links.
3. **Build the anchor**: `task-spec.md` from the existing data/feature/label docs. Mark anything not
   verifiable from code or sources as UNKNOWN.
4. **Reconstruct the eval protocol** from what the project actually does now (walk-forward folds,
   embargo, holdout period, seeds). Present it to the human for approval — this is the first time it
   becomes frozen.
5. **Backfill the experiment system** from the history doc:
   - one `LOG.md` row per historical eval/stage, numbered `H01, H02…` (historical, no EXP files)
   - rejected directions → `REJECTED.md` with the stated reason
   - current beliefs → `FINDINGS.md`; any claim the history marks as retracted/corrected goes in as
     a ⚠ RETRACTED entry, keeping both numbers
   - future work listed in the history → `INBOX.md`, source = human
   New experiments start at `EXP-001`.
6. **Attribution pass:** where the history says "decided" but the evidence is an AI suggestion, tag
   the INBOX/FINDINGS source accordingly or ask the human.
7. **Archive the roadmap**: move `roadmap/` → `archive/roadmap/`, remove its routing rows, add a note
   in `arch-harness.md`.
8. **Rewrite CLAUDE.md** with the experiment template, preserving any project-specific coding rules
   and env var notes from the old file. Status block from the latest history section.
9. **Rules:** add `eval-freeze`, `experiments`, `src-model`, `task-spec-sync`, `compute`; update
   `harness.md` globs; keep existing domain rules if their globs still match.
10. Present the full plan (moves, backfilled rows, new rules) → confirm → execute → commit
    `[harness] migrate to experiment-harness`.

## Don't

- Don't rewrite history docs into new wording — link to them; they are the primary record.
- Don't silently promote numbers from a history doc that the doc itself later retracted.
