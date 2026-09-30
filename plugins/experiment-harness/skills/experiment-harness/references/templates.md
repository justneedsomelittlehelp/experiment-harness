# Harness File Templates

Copy-paste templates for the files this skill creates. Each section is referenced from the step
that needs it. Fill every `<placeholder>` with real project content — a file committed with
angle-bracket placeholders still in it is worse than no file. Archetype-specific files (submission
gate, bot snapshots, opponent pool) live in the archetype references.

Every doc under `docs/` opens with the same header block — *Read this when* / *Does NOT cover* /
*Related docs* / *Invariants* — and the first 30 lines must stand alone.

---

## CLAUDE.md

*Step 7. ≤120 lines. Sections: task line / Role Contract / routing table / Rules / Status / Stop Points / Memory / Coding Rules / Delegation / Harness Maintenance.*

```markdown
# <Project name>

> <One line: what is predicted or played, how it's scored, the deadline.>

## Role Contract

- **Human** decides hypotheses, architecture, the eval protocol, and direction (promote/abandon).
- **Claude** implements specs exactly, runs experiments, logs results, proposes verdicts, keeps git clean.
- Claude never changes architecture silently. Open implementation details are Claude's; anything that
  changes what the model *is* goes back to the human.
- Claude's own ideas → `experiments/INBOX.md` tagged `[claude-suggestion]`, never straight into code.
- When recalling history, attribute: "you decided" only if the human said so; otherwise "I suggested".

## Where to Read

| When working on | Read |
|---|---|
| Data, metric, constraints, submission, deadlines | `docs/task-spec.md` |
| Finding a file, entry point, config or function | `docs/code-map.md` |
| Adding/upgrading a library, runtime or export format; "why this stack?" | `docs/arch-foundations.md` |
| Splits, seeds, promotion criterion, leakage checks | `docs/eval-protocol.md` |
| <component, e.g. sequence model> | `docs/arch-<component>.md` |
| Starting / running / logging an experiment | `experiments/LOG.md` (procedure: experiment-harness skill, `protocols.md`) |
| What is currently believed true (and retracted) | `experiments/FINDINGS.md` |
| Ideas already tried and rejected | `experiments/REJECTED.md` |
| Budget, spend, approval threshold | `experiments/COMPUTE.md` |
| Maintaining or auditing the harness | `docs/arch-harness.md` |

Anchors: `task-spec.md` (the task) and `code-map.md` (the repo). When docs disagree, fix the anchor first.

## Rules (`.claude/rules/`, auto-loaded)

| Rule | Applies to | Mirrors invariants of |
|---|---|---|
| `eval-freeze.md` | `<eval/**, data split files>` | `eval-protocol.md` |
| `experiments.md` | `experiments/**` | — |
| `src-model.md` | `<src/models/**>` | `arch-<component>.md` |
| `task-spec-sync.md` | `docs/task-spec.md`, `<engine/data version pin>` | `task-spec.md` |
| `dependencies.md` | `<requirements*.txt, pyproject.toml, lockfiles>` | `arch-foundations.md` |
| `code-map-sync.md` | `<src/**, eval/**, scripts/**, configs/**>` | `code-map.md` |
| `compute.md` | `<scripts/launch*.sh, infra/**>` | — |
| `harness.md` | `CLAUDE.md`, `docs/**`, `.claude/rules/**` | `arch-harness.md` |
| <archetype rules> | <globs> | <doc> |

## Status (snapshot — overwrite, don't append)

- **Baseline:** <EXP-id, commit, score ± seed spread>
- **Best candidate:** <EXP-id, score ± spread, verdict pending/ADOPT>
- **Active:** <EXP ids in flight>
- **Budget:** <spent> / <total> (<% used>)
- **Deadline:** <date> — <N> days. Freeze from <date>. Mode: <normal | FREEZE>

## Stop Points (wait for the human)

Architecture changes · eval protocol changes · promoting a baseline · retracting a headline ·
any run estimated above <threshold> (see `COMPUTE.md`).

## Memory

- Auto memory holds **pointers only**, one format:
  `When <situation> → read <path> (<what's there, one line>)`. Only for triggers not already a row
  in "Where to Read". Never save experiment status, scores, verdicts, budget, deadlines or
  architecture details there — their homes are listed above.
- Machine-specific paths and user preferences may go in auto memory; personal uncommitted notes →
  `CLAUDE.local.md`. No root `MEMORY.md`. No `@imports` of long docs — use routing rows.

## Coding Rules

- One variable per experiment; everything else pinned to the base commit.
- Every run writes a manifest (commit, config, seeds, data hash) — see eval protocol.
- <project-specific conventions>

## Delegation

- **Subagent** (fresh context): sweeps across many files, reading large logs, questions spanning
  3+ docs, and leakage audits (a context that hasn't seen the result is a fairer auditor). Name the
  docs to read in the prompt — a subagent may not inherit this file.
- **Inline**: work inside one doc's scope you already have context for, or needing < 3 file reads.

## Harness Maintenance

Full procedure and audit → `docs/arch-harness.md`. File-edit triggers fire from `.claude/rules/`;
these need a deliberate decision:

| Event | Update |
|---|---|
| Verdict reached | `LOG.md` row, EXP file, `REJECTED.md` if REJECT, `FINDINGS.md` if it changes a belief |
| Baseline promoted | Status block + `FINDINGS.md` |
| Claim retracted | Retraction procedure (`arch-harness.md`), incl. auto memory |
| New doc created | Routing row above (never before the file exists) |
| Deadline within freeze window | Mode → FREEZE in Status |
| Every ~10 EXPs or weekly | Harness audit (`arch-harness.md`) |
```

---

## Task spec (`docs/task-spec.md`)

*Step 2. Anchor for the task. Tables, not prose.*

```markdown
# Task Spec

> **Verified against:** <doc URL / engine version / data release> **on** <YYYY-MM-DD>
> **Read this when:** making any claim about the data, metric, constraints, submission or deadlines.
> **Does NOT cover:** how we evaluate internally → `eval-protocol.md`; where code lives → `code-map.md`.
> **Related docs:** `eval-protocol.md`, `arch-foundations.md` (runtime forced by the scoring env).
> **Anchor:** when another doc disagrees with this one about the task, re-verify here first.

## Invariants
- <e.g. Metric is weighted Pearson with weights |clip(y,-2,2)|, averaged per sequence then over sequences>
- <e.g. State must reset when seq_ix changes; every row is fed, including warm-up>
- <e.g. Inference: 1 vCPU, no GPU, 60 min total, deterministic>

## Objective & Metric
| Item | Value | Source |
|---|---|---|
| Target(s) | <t0, t1> | <link> |
| Metric | <exact formula> | <link> |
| Scored rows | <mask definition> | <link> |

## Data
| Split | Units | Rows | Size in memory (dtype) | Notes |
|---|---|---|---|---|
| train | <sequences> | <rows> | <GB, float32> | <arithmetic shown> |

| Column group | Columns | Meaning (as documented) |
|---|---|---|

## Constraints
| Constraint | Limit | Derived budget |
|---|---|---|
| Time | <60 min total> | <per-step µs = 3600 s / N steps, with N stated or UNKNOWN> |

## Submission
| Item | Value |
|---|---|

## Deadlines
| Event | Date (timezone) |
|---|---|

## Unknowns
| Question | Where to check |
|---|---|
```

---

## Code map (`docs/code-map.md`)

*Step 2 (created once the layout exists). Anchor for the repo. Lookup tables only, no prose. Lists
only files that exist.*

```markdown
# Code Map

> **Read this when:** looking for a file, entry point, config, script or function; before stating
> any path or function name in another doc.
> **Does NOT cover:** what the model is → `arch-<component>.md`; why a library → `arch-foundations.md`.
> **Related docs:** `eval-protocol.md` (what `eval/` must do), `task-spec.md`.
> **Anchor:** when another doc disagrees with this one about a path or symbol, check the code and fix here first.

## Invariants
- Every path and symbol named in any doc appears here and exists in the repo.
- One entry point per job (train, evaluate, export, submit).

## Entry Points
| Job | Command | File |
|---|---|---|
| Train | <python -m src.train --config configs/<x>.yaml> | <src/train.py> |

## Directory Map
| Path | Contents |
|---|---|

## Key Symbols
| Symbol | File | Purpose (one line) |
|---|---|---|

## Artifacts
| What | Where | Tracked in git? |
|---|---|---|
| Run manifests / metrics | `results/<EXP>/<seed>/` | no |
```

---

## Foundations (`docs/arch-foundations.md`)

*Step 2. Stack and dependency choices with reasons. Updated by the `dependencies.md` rule.*

```markdown
# Foundations

> **Read this when:** adding, removing or upgrading a library; choosing a runtime or export format;
> asking "why this stack?".
> **Does NOT cover:** model design → `arch-<component>.md`; scoring-env limits themselves → `task-spec.md`.
> **Related docs:** `task-spec.md` (constraints that force choices), `code-map.md`.

## Invariants
- <e.g. Inference code runs on Python 3.10 with onnxruntime <x.y>, 1 intra-op thread>
- <e.g. No training-only dependency is imported by the submission path>

## Stack
| Layer | Choice (pinned version) | Why | Forced by constraint? |
|---|---|---|---|
| Training | <PyTorch x.y> | <…> | no |
| Inference runtime | <onnxruntime x.y> | <…> | <yes: task-spec § Constraints> |
| Data format | <sharded .npy, float16> | <…> | <memory limit> |
| Compute | <provider, GPU type> | <…> | budget |

## Alternatives Considered
| Choice | Alternative | Why not |
|---|---|---|

## Evolution Log
| Date | Change | Reason | EXP |
|---|---|---|---|
```

---

## Eval protocol (`docs/eval-protocol.md`)

*Step 3. Human-approved, then frozen.*

```markdown
# Evaluation Protocol

> **Read this when:** designing, running or judging any experiment.
> **Does NOT cover:** the task's official metric definition → `task-spec.md`; scorer file paths → `code-map.md`.
> **Related docs:** `task-spec.md`, skill `references/protocols.md` (statistics, leakage audit).
> **Approved by the human on:** <date>. Changes require approval + a changelog row.

## Invariants
- Holdout `<name>` is never used for training, tuning, early stopping, threshold or filter selection.
- A comparison uses ≥ <3> independent seeds per arm; single-seed results are PRELIMINARY.
- ADOPT requires: <criterion, e.g. mean Δ > 0 on validation AND Δ > max(seed spread of either arm)
  AND not worse on holdout>.
- Metric code in `eval/` is the only scorer; analyzers may explore but never produce a headline.

## Splits
| Split | Definition | Used for |
|---|---|---|
| train | <…> | fitting |
| val | <…> | model selection, early stopping |
| holdout | <…> | promotion decisions only |

## Seeds & Variance
<independent runs via base-seed; how spread is reported (min/mean/max); compute cost per seed>

## Promotion Criterion
<exact rule, with numbers>

## Leakage Checklist (task-specific)
- [ ] <e.g. normalization statistics computed causally, never over future steps>
- [ ] <e.g. no label horizon crosses a split boundary (embargo = <N>)>
- [ ] <e.g. selection of thresholds/filters done on val, not holdout>

## Run Manifest
Every run writes `results/<EXP>/<seed>/manifest.json`: commit, dirty flag, config, seed, data
hash, start/end time, hardware, metric version.

## Changelog
| Date | Change | Approved by | Affected EXPs |
|---|---|---|---|
```

---

## Architecture spec (`docs/arch-<component>.md`)

*Step 4. Written from the human's description, read back and approved.*

```markdown
# <Component name> — v<N>

> **Read this when:** implementing or modifying `<path from code-map.md>`.
> **Does NOT cover:** training loop / eval → `eval-protocol.md`; libraries/runtime → `arch-foundations.md`.
> **Related docs:** `task-spec.md` (latency/size budget), <other arch docs>.
> **Author:** human. **Transcribed by:** Claude on <date>. **Approved:** <date>.

## Invariants
- <e.g. Stateful GRU; hidden state carried across steps within a sequence, reset between>
- <e.g. Per-step CPU latency ≤ <budget> µs single-thread>

## Spec
| Block | Input shape | Output shape | Notes |
|---|---|---|---|

Loss: <…> · Output: <…> · Param budget: <…> · Latency budget: <…>

## Left to Claude
<init, dtype, batching, padding — anything not affecting what the model is>

## Version History
| Version | EXP | Change | Decided by |
|---|---|---|---|
```

---

## Experiment file (`experiments/EXP-NNN.md`)

*≤80 lines. Sections above the line are written BEFORE running.*

```markdown
# EXP-<NNN>: <short title>

- **Status:** DRAFT | RUNNING | DONE | RETRACTED
- **Source:** human | strategist | [claude-suggestion] promoted by human on <date>
- **Branch / base commit:** exp/<NNN>-<slug> @ <sha>

## Hypothesis
<One falsifiable sentence.>

## Change (single variable)
<What differs from base. Everything else identical.>

## Eval Config
<splits, seeds, epochs/steps, opponents/maps if agent>

## Pre-registered Criterion
<ADOPT if …; REJECT if …; else INCONCLUSIVE>

## Cost Estimate
<GPU-hours × rate = $; above threshold? → approval required>

## Prior Art Check
<REJECTED.md / FINDINGS.md entries that overlap, or "none found (grep terms: …)">

---

## Result
| Arm | seed A | seed B | seed C | mean | spread |
|---|---|---|---|---|---|
Holdout: <…> · Manifest: `results/EXP-<NNN>/`

## Deviations
<anything run differently from the plan, and why>

## Verdict (proposed by Claude)
ADOPT | REJECT | INCONCLUSIVE — <reason> · **Human decision:** <pending | confirmed on date>

## Lesson
<One line that would stop someone repeating a mistake.>
```

---

## `experiments/LOG.md`

```markdown
# Experiment Log

| EXP | Date | Hypothesis (short) | Source | Result (mean ± spread) | Verdict | Commit / tag |
|---|---|---|---|---|---|---|
| 000 | <date> | Reproduce provided baseline | human | <score> | ADOPT (baseline) | <tag> |
```

Retracted rows are kept, marked `⚠ RETRACTED (see FINDINGS §n)`. Stale rows after a spec change
are marked `STALE?`.

---

## `experiments/INBOX.md`

```markdown
# Inbox

Plain-language ideas. Anyone may add. Claude converts them on "process inbox".

| # | Date | From | Idea | Status |
|---|---|---|---|---|
| 1 | <date> | <human / strategist / [claude-suggestion]> | <text> | open / → EXP-<n> / declined |
```

---

## `experiments/REJECTED.md`

```markdown
# Rejected Ideas

Grep this before every new experiment. One line each.

| Idea | EXP | Why rejected | Conditions under which it might be worth retrying |
|---|---|---|---|
```

---

## `experiments/FINDINGS.md`

```markdown
# Findings

What we currently believe, with evidence. Newest first. Never delete — retract.

## F<n>. <claim> — <date>
- **Evidence:** EXP-<…> (<numbers, seeds>)
- **Confidence:** high | medium | preliminary (single seed)
- **Implication:** <what this changes>

## ⚠ F<m>. ~~<old claim>~~ — RETRACTED <date>
- **Why:** <leak / non-replication / metric error>
- **Replaced by:** F<k>
```

---

## `experiments/COMPUTE.md`

```markdown
# Compute Ledger

- **Budget:** <amount, currency> · **Provider:** <…> · **Approval threshold per run:** <$ or GPU-h>
- **Shutdown rule:** every launch script stops the instance on exit; idle > <N> min → stop.

| Date | EXP | Hardware | Hours | Rate | Cost | Cumulative |
|---|---|---|---|---|---|---|
```

---

## Harness maintenance doc (`docs/arch-harness.md`)

```markdown
# Harness Maintenance

> **Harness version:** experiment-harness v<X.Y.Z> (set at setup, bumped by every upgrade)
> **Read this when:** closing an experiment, retracting a claim, adding a doc, or auditing the harness.
> **Does NOT cover:** the experiment lifecycle itself → skill `references/protocols.md`.
> **Related docs:** every doc in `docs/` (this one governs how they stay current).

## Invariants
- Every doc has a routing row in CLAUDE.md; no row points at a file that doesn't exist.
- Rule-file invariants match the Invariants block of the doc they guard, word for word.
- No fact lives in two places. Budgets: CLAUDE.md ≤120, rules ≤50, docs ≤500, EXP ≤80.
- Nothing is deleted from LOG, FINDINGS or REJECTED — only marked.
- Every rule file has `paths`; no `@import` of long docs into CLAUDE.md.

## Where Information Lives
| Fact | Home |
|---|---|
| Data / metric / constraints / deadlines | `task-spec.md` |
| Paths, entry points, key functions | `code-map.md` |
| Libraries, runtime, versions, export format — and why | `arch-foundations.md` |
| How we evaluate | `eval-protocol.md` |
| What the model is | `arch-<component>.md` |
| What was tried | `LOG.md` + EXP files |
| What is believed | `FINDINGS.md` |
| What failed | `REJECTED.md` |
| Money | `COMPUTE.md` |
| Current snapshot | CLAUDE.md Status |
| Personal, uncommitted notes | `CLAUDE.local.md` (gitignored) |
| Machine paths, user preferences, pointers | auto memory — `When <situation> → read <path> (<what's there>)`, never facts above, never a copy of a CLAUDE.md routing row |

## Harness Audit (weekly, every ~10 experiments, and before a freeze)
1. **Doc rot.** Extract every path, file and code symbol named in `docs/`, CLAUDE.md and rule files;
   `grep`/`ls` each one. Fix or remove any that no longer exist — a doc naming a deleted function
   gets believed. Start with `code-map.md`, then the docs that cite it.
2. **Anchors match reality.** `code-map.md` matches the tree; `task-spec.md` verification date is
   recent and the engine/data version still matches the pin.
3. **Rule globs** still match real paths; new directories are covered by some rule if they need one.
4. **Invariant sync.** Each rule file's invariants equal its doc's Invariants block.
5. **Budgets** within limits; routing rows = existing docs, one-to-one.
6. **Evidence.** Every ADOPT has ≥3 seeds; every FINDINGS claim cites an EXP that still stands.
7. **Auto memory** (`/memory`): delete any entry that states a fact with a repo home (status,
   scores, verdicts, budget, deadlines) or points at a moved/deleted file. Keep pointers and
   machine-specific notes only.
Record the audit date and anything fixed in the Audit Log below.

## Audit Log
| Date | Fixed |
|---|---|

## Deferred Components
Components intentionally postponed. **No routing row until the file exists.** Delete the row here
once created.

| Component | Create when |
|---|---|

## Retraction Procedure
See skill `references/protocols.md` § Retraction.
```

---

## Rules (`.claude/rules/`)

*Step 7. Adjust every glob to the real tree. A rule guarding a doc copies that doc's Invariants
block exactly, then points to the doc. Archetype references add more.*

### `eval-freeze.md`
```markdown
---
paths:
  - "eval/**"
  - "<data/splits/**>"
---
# Eval Freeze
Invariants (mirrored from `docs/eval-protocol.md` — keep identical):
- <copy the Invariants block of eval-protocol.md>

Procedure:
- Do not edit these files without the human's explicit approval in this conversation.
- If approved: add a Changelog row in `docs/eval-protocol.md` listing affected EXPs, mark those EXPs
  `STALE?` in `LOG.md`, commit as `[eval] <change>` separately from any experiment.
- Never move data between train/val/holdout. Never tune on holdout.
Full protocol → `docs/eval-protocol.md`
```

### `experiments.md`
```markdown
---
paths:
  - "experiments/**"
---
# Experiment Records
- EXP files follow the template; Hypothesis/Change/Criterion are written before the run.
- Every EXP has a LOG.md row. A REJECT appends to REJECTED.md in the same edit.
- Never delete rows or findings; retract with ⚠ and a pointer.
- Attribute sources honestly: human / strategist / [claude-suggestion].
- Result numbers must come from a run manifest in `results/`; never type numbers from memory.
Procedures → skill `references/protocols.md`
```

### `src-model.md`
```markdown
---
paths:
  - "<src/models/**>"
---
# Model Code
Invariants (mirrored from `docs/arch-<component>.md` — keep identical):
- <copy the Invariants block of the arch doc>

Procedure:
- Differences from the spec in inputs, layers, loss, pooling or capacity need the human's approval
  and a spec version bump.
- New tunables go to config, not literals in logic.
- Measure latency/size against `docs/task-spec.md` after structural edits.
Spec → `docs/arch-<component>.md`
```

### `task-spec-sync.md`
```markdown
---
paths:
  - "docs/task-spec.md"
  - "<engine/data version pin, e.g. requirements.txt>"
---
# Task Spec Sync
- Changing the engine/data version? Re-verify task-spec.md against sources, update the stamp,
  and run the Spec Change procedure (flag affected docs and EXPs as STALE?).
- Never write an unverified value; use `UNKNOWN — check <where>`.
- This is an anchor: when another doc disagrees about the task, re-verify here first.
Procedure → skill `references/protocols.md` § Spec change
```

### `dependencies.md`
```markdown
---
paths:
  - "<requirements*.txt>"
  - "<pyproject.toml>"
  - "<*.lock / environment.yml>"
---
# Dependencies
Invariants (mirrored from `docs/arch-foundations.md` — keep identical):
- <copy the Invariants block of arch-foundations.md>

Procedure:
- Added, removed or upgraded a dependency? Add an Evolution Log row in `arch-foundations.md` with
  the reason and EXP, and update the Stack table if a layer changed.
- Anything imported by the submission/inference path must satisfy the scoring environment in
  `task-spec.md` § Constraints — check before adding.
Rationale → `docs/arch-foundations.md`
```

### `code-map-sync.md`
```markdown
---
paths:
  - "<src/**>"
  - "<eval/**>"
  - "<scripts/**>"
  - "<configs/**>"
---
# Code Map Sync
- Created, moved, renamed or deleted an entry point, script, config family or a symbol listed in
  `docs/code-map.md`? Update code-map.md in the same commit.
- Before naming a path or function in any doc, confirm it's in code-map.md (and exists).
Lookup tables → `docs/code-map.md`
```

### `compute.md`
```markdown
---
paths:
  - "<scripts/launch*.sh>"
  - "<infra/**>"
---
# Compute
- Every launch estimates cost first; above the threshold in COMPUTE.md → ask the human.
- Every launch path stops the instance on completion or failure (trap/finally).
- Log each run in COMPUTE.md with actual hours.
- Checkpoint at least every <N> minutes; runs must resume from the last checkpoint.
```

### `harness.md`
```markdown
---
paths:
  - "CLAUDE.md"
  - "docs/**"
  - ".claude/rules/**"
---
# Harness
- New doc → routing row in CLAUDE.md, only once the file exists. Deferred docs live in
  `arch-harness.md` § Deferred Components.
- Changed a doc's Invariants? Update the rule file that mirrors it in the same edit (and vice versa).
- Every doc opens with Read this when / Does NOT cover / Related docs / Invariants; the first 30
  lines stand alone.
- CLAUDE.md ≤120 lines, Status is a snapshot (overwrite); docs ≤500; rules ≤50 and always with `paths`.
- No `@import` of long docs into CLAUDE.md.
Procedure and audit → `docs/arch-harness.md`
```

---

## Postmortem (`experiments/POSTMORTEM.md`)

```markdown
# Postmortem — <project>, <dates>

## Outcome
<final score/rank, final model lineage (EXP ids)>

## What Worked (top adopted experiments)
| EXP | Change | Effect (mean ± spread) |

## What Didn't (top rejected, with reasons)
| EXP | Idea | Why it failed |

## Retractions and What They Taught
<each retraction: what was claimed, what was wrong, how it was caught>

## Local Evaluation vs. Final/External Result
<divergence and its likely cause>

## Process Lessons
<what to change in the harness next time>
```
