# Harness File Templates

Copy-paste templates for the files this skill creates. Each section is referenced from the step
that needs it. Fill every `<placeholder>` with real project content — a file committed with
angle-bracket placeholders still in it is worse than no file. Archetype-specific files (submission
gate, bot snapshots, opponent pool) live in the archetype references.

---

## CLAUDE.md

*Step 7. ≤120 lines. Sections: task line / Role Contract / routing table / Rules / Status / Stop Points / Coding Rules / Delegation / Harness Maintenance.*

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
| Splits, seeds, promotion criterion, leakage checks | `docs/eval-protocol.md` |
| <component, e.g. sequence model> | `docs/arch-<component>.md` |
| Starting / running / logging an experiment | `experiments/LOG.md` (procedure: experiment-harness skill, `protocols.md`) |
| What is currently believed true (and retracted) | `experiments/FINDINGS.md` |
| Ideas already tried and rejected | `experiments/REJECTED.md` |
| Budget, spend, approval threshold | `experiments/COMPUTE.md` |
| Maintaining the harness | `docs/arch-harness.md` |

## Rules (`.claude/rules/`, auto-loaded)

| Rule | Applies to |
|---|---|
| `eval-freeze.md` | `<eval/**, data split files>` |
| `experiments.md` | `experiments/**` |
| `src-model.md` | `<src/models/**>` |
| `task-spec-sync.md` | `docs/task-spec.md`, `<lockfile pinning engine/data version>` |
| `compute.md` | `<scripts/launch*.sh, infra/**>` |
| `harness.md` | `CLAUDE.md`, `docs/**`, `.claude/rules/**` |
| <archetype rules> | <globs> |

## Status (snapshot — overwrite, don't append)

- **Baseline:** <EXP-id, commit, score ± seed spread>
- **Best candidate:** <EXP-id, score ± spread, verdict pending/ADOPT>
- **Active:** <EXP ids in flight>
- **Budget:** <spent> / <total> (<% used>)
- **Deadline:** <date> — <N> days. Freeze from <date>. Mode: <normal | FREEZE>

## Stop Points (wait for the human)

Architecture changes · eval protocol changes · promoting a baseline · retracting a headline ·
any run estimated above <threshold> (see `COMPUTE.md`).

## Coding Rules

- One variable per experiment; everything else pinned to the base commit.
- Every run writes a manifest (commit, config, seeds, data hash) — see eval protocol.
- <project-specific conventions>

## Delegation

Use a subagent for sweeps across many files, reading large logs, or leakage audits (a fresh context
that hasn't seen the result is a fairer auditor). Name the docs to read in the prompt.

## Harness Maintenance

Full procedure → `docs/arch-harness.md`. Conversation-triggered events:

| Event | Update |
|---|---|
| Verdict reached | `LOG.md` row, EXP file, `REJECTED.md` if REJECT, `FINDINGS.md` if it changes a belief |
| Baseline promoted | Status block + `FINDINGS.md` |
| Claim retracted | Retraction procedure (`arch-harness.md`) |
| Deadline within freeze window | Mode → FREEZE in Status |
```

---

## Task spec (`docs/task-spec.md`)

*Step 2. The anti-hallucination anchor. Tables, not prose.*

```markdown
# Task Spec

> **Verified against:** <doc URL / engine version / data release> **on** <YYYY-MM-DD>
> **Read this when:** making any claim about the data, metric, constraints or submission.
> **Does NOT cover:** how we evaluate internally → `eval-protocol.md`.

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

## Eval protocol (`docs/eval-protocol.md`)

*Step 3. Human-approved, then frozen.*

```markdown
# Evaluation Protocol

> **Read this when:** designing, running or judging any experiment.
> **Does NOT cover:** the task's official metric definition → `task-spec.md`.
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

> **Read this when:** implementing or modifying `<path>`.
> **Does NOT cover:** training loop / eval → `eval-protocol.md`.
> **Author:** human. **Transcribed by:** Claude on <date>. **Approved:** <date>.

## Invariants
- <e.g. Stateful GRU; hidden state carried across steps within a sequence, reset between>
- <e.g. Per-step CPU latency ≤ <budget> µs single-thread>

## Spec
| Block | Input shape | Output shape | Notes |
|---|---|---|---|

Loss: <…> · Output: <…> · Param budget: <…> · Latency budget: <…>

## Left to Claude
<init, dtype, batching, padding, export format — anything not affecting what the model is>

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

> **Read this when:** closing an experiment, retracting a claim, adding a doc, or auditing the harness.
> **Does NOT cover:** the experiment lifecycle itself → skill `references/protocols.md`.

## Invariants
- Every doc has a routing row in CLAUDE.md.
- Rule-file invariants match the Invariants block of their doc.
- No fact lives in two places. Budgets: CLAUDE.md ≤120, rules ≤50, docs ≤500, EXP ≤80.
- Nothing is deleted from LOG, FINDINGS or REJECTED — only marked.

## Where Information Lives
| Fact | Home |
|---|---|
| Data / metric / constraints | `task-spec.md` |
| How we evaluate | `eval-protocol.md` |
| What the model is | `arch-<component>.md` |
| What was tried | `LOG.md` + EXP files |
| What is believed | `FINDINGS.md` |
| What failed | `REJECTED.md` |
| Money | `COMPUTE.md` |
| Current snapshot | CLAUDE.md Status |

## Weekly Audit (or every 10 experiments)
1. Budgets within limits. 2. Rule globs still match real paths. 3. Every ADOPT has ≥3 seeds.
4. FINDINGS claims each cite an EXP that still stands. 5. `task-spec.md` verification date recent.
6. Auto memory (`/memory`): delete stale "currently running X" notes.

## Deferred Components
| Component | Create when |
|---|---|

## Retraction Procedure
See skill `references/protocols.md` § Retraction.
```

---

## Rules (`.claude/rules/`)

*Step 7. Adjust every glob to the real tree. Archetype references add more.*

### `eval-freeze.md`
```markdown
---
paths:
  - "eval/**"
  - "<data/splits/**>"
---
# Eval Freeze
- Do not edit these files without the human's explicit approval in this conversation.
- If approved: add a row to the Changelog in `docs/eval-protocol.md` listing affected EXPs, mark
  those EXPs `STALE?` in `LOG.md`, commit as `[eval] <change>` separately from any experiment.
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
- Implementation must match `docs/arch-<component>.md`. Differences in inputs, layers, loss,
  pooling or capacity need the human's approval and a spec version bump.
- New tunables go to config, not literals in logic.
- Respect latency/size budgets in `docs/task-spec.md`; measure after structural edits.
Spec → `docs/arch-<component>.md`
```

### `task-spec-sync.md`
```markdown
---
paths:
  - "docs/task-spec.md"
  - "<requirements*.txt / pyproject.toml / engine lockfile>"
---
# Task Spec Sync
- Changing the engine/data version? Re-verify task-spec.md against sources, update the stamp,
  and run the Spec Change procedure (flag affected docs and EXPs as STALE?).
- Never write an unverified value; use `UNKNOWN — check <where>`.
Procedure → skill `references/protocols.md` § Spec change
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
- New doc → routing row in CLAUDE.md. Changed invariants → update doc AND rule file.
- CLAUDE.md ≤120 lines; Status is a snapshot, overwrite it.
- Docs ≤500 lines; rules ≤50.
Procedure → `docs/arch-harness.md`
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
