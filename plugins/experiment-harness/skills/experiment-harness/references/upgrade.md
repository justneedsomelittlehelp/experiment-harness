# Migrate and Upgrade

Two procedures: **Migrate** a project-harness repo into this harness, and **Self-upgrade** a repo
set up with an older experiment-harness version. Both are additive and non-destructive: never
overwrite user content, never delete history, present the plan and wait for confirmation.

---

## Migrate (from project-harness)

For repos set up with `project-harness` (CLAUDE.md routing table + `architecture_docs/` + `roadmap/`)
that have turned into experiment-driven work — e.g. a trading-model project whose "phases" became a
series of evals, audits and retractions.

### Detect

| Found | Meaning |
|---|---|
| `roadmap/PHASE_N.md` | Old planning layer — archive, don't delete |
| `architecture_docs/arch-foundations.md` | Keep — it is this harness's `arch-foundations.md` |
| `architecture_docs/arch-reference.md` | Becomes the code anchor (`code-map.md` role) |
| Other `architecture_docs/arch-*.md` | Keep; model docs become `arch-<component>.md` specs |
| An eval log / model history doc (e.g. `experiments/MODEL_HISTORY.md`, `EVAL_*.md`) | Source for LOG, FINDINGS, REJECTED backfill |
| `security_check/`, `Design.md` | Keep as-is if they still apply; not part of this harness |

### Procedure

1. **Read everything first**: CLAUDE.md, the roadmap README, every arch doc header, and the full
   model-history/eval docs. Delegate to a subagent if they're long.
2. **Path choice (ask the human):** keep `architecture_docs/` in place and point the routing table
   at it, or move to `docs/`. Keeping in place is the default — fewer broken links. If
   `arch-reference.md` is kept, it plays the `code-map.md` role; don't create both.
3. **Anchors**: build `task-spec.md` from the existing data/feature/label docs (UNKNOWN where not
   verifiable). Run the doc-rot check on `arch-reference.md` and fix it before relying on it.
4. **Reconstruct the eval protocol** from what the project actually does now (walk-forward folds,
   embargo, holdout period, seeds). Present it to the human for approval — this is the first time
   it becomes frozen.
5. **Backfill the experiment system** from the history doc:
   - one `LOG.md` row per historical eval/stage, numbered `H01, H02…` (historical, no EXP files)
   - rejected directions → `REJECTED.md` with the stated reason
   - current beliefs → `FINDINGS.md`; any claim the history marks as retracted/corrected goes in as
     a ⚠ RETRACTED entry, keeping both numbers
   - future work listed in the history → `INBOX.md`, source = human
   New experiments start at `EXP-001`.
6. **Attribution pass:** where the history says "decided" but the evidence is an AI suggestion, tag
   the INBOX/FINDINGS source accordingly or ask the human.
7. **Archive the roadmap**: move `roadmap/` → `archive/roadmap/`, remove its routing rows, add a
   note in `arch-harness.md`.
8. **Rewrite CLAUDE.md** with the experiment template, preserving any project-specific coding rules
   and env var notes from the old file. Status block from the latest history section.
9. **Rules:** add `eval-freeze`, `experiments`, `src-model`, `task-spec-sync`, `code-map-sync`,
   `compute`; keep the old `dependencies.md` and `reference-sync.md` if present (reference-sync
   covers code-map-sync — keep one); update `harness.md`; keep domain rules whose globs still match.
10. **Memory pass:** review auto memory (`/memory`) with the human. Entries that state facts now
    homed in the harness (scores, stage status, "current best strategy") are deleted; pointer
    entries are rewritten in the `When <situation> → read <path> (<what's there>)` format and
    repointed. Pay special attention to numbers the history later retracted.
11. Stamp `Harness version` in `arch-harness.md`, present the full plan (moves, backfilled rows, new
    rules) → confirm → execute → commit `[harness] migrate to experiment-harness v<X.Y.Z>`.

### Don't

- Don't rewrite history docs into new wording — link to them; they are the primary record.
- Don't silently promote numbers from a history doc that the doc itself later retracted.

---

## Self-upgrade (older experiment-harness → this version)

### Detect the installed version

Read `Harness version` in `docs/arch-harness.md`. If it's missing (repos from v0.1.0–v0.1.2 have no
stamp), infer it from the table below — the first row whose check fails is where the upgrade starts.

| Added in | Check | If missing, do |
|---|---|---|
| v0.1.1 | CLAUDE.md has a `## Memory` section | Add it from the template; run the memory pass below |
| v0.1.2 | Memory section uses the `When <situation> → read <path> (<what's there>)` format | Replace the pointer rule; rewrite existing memory entries in that format; drop entries duplicating a routing row |
| v0.1.3 | `docs/code-map.md` exists | Create it from the actual tree (entry points, directory map, key symbols) — only files that exist |
| v0.1.3 | `docs/arch-foundations.md` exists | Create it from `requirements`/`pyproject`/lockfiles and the scoring env in `task-spec.md`; ask the human for reasons you can't infer |
| v0.1.3 | Rules `dependencies.md` and `code-map-sync.md` exist | Create with globs matching the real tree |
| v0.1.3 | Every doc header has *Related docs*; arch-harness has the stamp, Audit Log and 7-step audit | Add the missing lines/sections without touching existing content |
| v0.1.3 | Rule files that guard a doc mirror its Invariants block | Copy the doc's Invariants into the rule (or ask which one is right if they differ) |
| v0.1.3 | CLAUDE.md routing table has rows for code-map and foundations; Delegation has the inline threshold | Add rows/lines; check the ≤120-line budget |

### Procedure

1. Run the detection table; list every missing item with the file it touches.
2. For each new doc, gather content from the repo first (no empty shells). What can't be inferred
   is asked, or recorded as `UNKNOWN — check <where>`.
3. Patch additively: insert sections, never rewrite existing prose, never renumber EXPs or findings.
   If an existing section conflicts with the new template, show both and ask.
4. **Memory pass** (for v0.1.1/v0.1.2 items): open `/memory`; delete entries stating facts with a
   repo home; rewrite pointers into the one-line format; remove pointers that duplicate routing rows.
5. Run the full harness audit (`arch-harness.md`) once, since new anchors make the doc-rot check
   possible for the first time.
6. Set `Harness version` to this skill's version; add an Audit Log row; present the diff summary →
   confirm → commit `[harness] upgrade to experiment-harness v<X.Y.Z>`.
