---
name: experiment-harness
version: 0.1.0
description: >
  Set up and run a research harness for experiment-driven projects — ML/prediction competitions,
  bot/agent competitions, and research playgrounds — where the human owns ideas and architecture
  and Claude owns implementation, experiment execution, logging and version control. Use this skill
  when the user says "set up an experiment harness", "research harness", "set up this competition",
  "start a new experiment project", or hands over a task plus data/engine and wants Claude to run
  the experiment loop. Also use it for the harness's recurring operations: "new experiment",
  "process inbox", "run EXP-N", "log results", "retract", "status report", "freeze", "submit",
  "task/engine updated", "postmortem", and "migrate from project-harness".
---

# Experiment Harness

You are setting up (or operating) a harness for a project whose work is **testing hypotheses**, not
building features. A task and its data (or a game engine) are given; the unit of progress is a
logged experiment with a verdict, not a completed phase.

This skill descends from `project-harness` (v1.5.0). It keeps that skill's context-engineering core
— budgets, routing table, path-scoped rules, one home per fact, an anti-hallucination anchor — and
replaces its phase roadmap with an experiment system, a frozen evaluation protocol, and a strict
division of labor between the human and Claude.

Every file you create must contain real, project-specific content. A template with placeholders
left in it is worse than no file. If something isn't known yet (e.g. a game that hasn't been
revealed), defer the file and record the trigger — never create a shell.

---

## The Role Contract (the core of this skill)

| Human owns | Claude owns |
|---|---|
| Hypotheses and research direction | Turning a hypothesis into a falsifiable experiment spec |
| Model / agent **architecture** and its spec | Implementing the spec exactly, flagging ambiguities |
| The evaluation protocol and promotion criteria | Running experiments, seeds, sweeps, audits |
| Direction decisions (promote a baseline, abandon a line) | Proposing a verdict with the evidence behind it |
| Spend above the approval threshold | Compute ledger, cost estimates, shutting down idle machines |
| — | Experiment log, findings, retractions, git history |

Rules that make the contract hold:

1. **Claude never silently changes architecture.** Choices the spec leaves open (init, padding,
   dtype, batching) are Claude's; anything that changes what the model *is* (inputs, layers, loss,
   pooling, capacity) goes back to the human. When unsure, ask.
2. **Claude's own ideas go to `experiments/INBOX.md` tagged `[claude-suggestion]`**, never straight
   into code. The human promotes or discards them. A later reader must be able to tell the human's
   decisions from Claude's suggestions.
3. **Claude proposes verdicts; the human decides direction.** Claude may mark an EXP `REJECT` or
   `INCONCLUSIVE` against its pre-registered criterion. Promoting a new baseline or abandoning a
   line of work needs the human's explicit OK.
4. **Stop points.** Claude pauses for the human at: architecture choices, any change to the
   evaluation protocol, promoting a baseline, retracting a headline finding, and spend above the
   threshold in `COMPUTE.md`. Everything else Claude does autonomously and reports afterwards.

---

## Harness Components

| Layer | File(s) | Purpose |
|---|---|---|
| Navigation hub | `CLAUDE.md` | Role contract, routing table, status, stop points |
| Task anchor | `docs/task-spec.md` | Anti-hallucination anchor: data schema, metric, constraints, submission, deadlines |
| Eval protocol | `docs/eval-protocol.md` | Splits, holdout, seeds, promotion criterion, leakage checklist — **frozen** |
| Architecture specs | `docs/arch-{component}.md` | Human-authored model/agent specs; the implementation must match |
| Experiment system | `experiments/` | `LOG.md`, `EXP-NNN.md`, `INBOX.md`, `REJECTED.md`, `FINDINGS.md`, `COMPUTE.md` |
| Harness procedure | `docs/arch-harness.md` | Maintenance events, audits, retraction procedure pointer |
| Rules | `.claude/rules/*.md` | Invariants that auto-load on matching file reads |

### Context budgets

| File | Budget | When exceeded |
|---|---|---|
| `CLAUDE.md` | ≤ 120 lines | Move detail to a doc, leave a routing row |
| `.claude/rules/*.md` | ≤ 50 lines | It's carrying reasoning → move to the doc |
| `docs/*.md` | ≤ 500 lines | Split; the original becomes an index |
| `experiments/EXP-NNN.md` | ≤ 80 lines | Raw numbers belong in `results/`, not markdown |
| `experiments/FINDINGS.md` | ≤ 300 lines | Archive superseded sections to `experiments/archive/` |

CLAUDE.md is tighter than project-harness's 150 because experiment projects accumulate status fast;
the status block must stay a snapshot, never a diary.

### Archetypes

Detect which applies (ask if unclear) and read its reference before planning:

- **Prediction task** — fixed dataset + metric (+ usually a submission), e.g. an order-book
  forecasting competition or a Kaggle contest. → `references/archetype-prediction.md`
- **Agent competition** — an engine/simulator, opponents, a ladder, e.g. a Battlecode-style bot
  competition. → `references/archetype-agent.md`

A research playground with no external deadline (e.g. a personal trading-model project) uses the
prediction archetype with the submission gate removed and a pristine holdout instead.

---

## How to Run This Skill

Templates are in `references/templates.md`; the experiment lifecycle, git policy, statistics,
leakage audit, retraction and postmortem procedures in `references/protocols.md`; migration from a
project-harness repo in `references/upgrade.md`. **Don't write a harness file from memory when a
template exists for it** — read the template section first.

### Step 0: Detect the mode

| Situation | Mode |
|---|---|
| Empty or new repo | **Setup** → Steps 1–8 |
| Repo has a project-harness (`roadmap/`, `architecture_docs/`) | **Migrate** → `references/upgrade.md` |
| Harness exists and the user asks for an operation | **Operate** → Step 9 |
| Task spec, engine version or data changed | **Spec change** → `protocols.md` § Spec change |
| Project ending or deadline passed | **Postmortem** → `protocols.md` § Postmortem |
| Agent archetype, game not revealed yet | **Pre-launch** → `archetype-agent.md` § Pre-launch |

### Step 1: Understand the task

Ask (or read, if the user gives links or files):
- What is predicted / played, and how is it scored? Get the **exact** metric definition.
- Hard constraints: compute, latency, model size, languages, submission format, determinism.
- Data or engine provided, its size, and whether it fits in memory.
- Deadlines: submission freeze, final evaluation, interim tournaments.
- Compute: hardware, provider, budget, and the per-run approval threshold.
- Team: who else contributes and how (e.g. a strategist who writes hypotheses but not code).

If official documentation exists online, fetch it (an `llms.txt` index if offered). The task anchor
is built from sources, never from memory.

### Step 2: Build the task anchor

Create `docs/task-spec.md` from the template: flat tables only, stamped
`Verified against <source/version> on <date>`. Every claim about data, metric or constraints
anywhere in the repo must be checkable there. Unknowns are written `UNKNOWN — check <where>`, never
guessed. Derived budgets (e.g. per-step inference time, memory footprint) are computed here with
the arithmetic shown.

### Step 3: Settle the evaluation protocol with the human

A human decision under the role contract. Propose, with reasons, from the archetype reference:
- the validation scheme and a **holdout never used for tuning or selection**
- seeds per comparison (default 3 independent runs — not an in-run ensemble)
- the promotion criterion (effect vs. seed spread, never a single-run delta)
- a leakage checklist specific to this task
- for agents: opponent pool, map split, time-budget margin

Write `docs/eval-protocol.md` and the locked metric implementation (`eval/`) only after the human
confirms. From then on both are frozen (rule `eval-freeze.md`).

### Step 4: Capture the first architecture spec

Ask the human for the baseline and their first idea. Transcribe into `docs/arch-{component}.md`
from the template — inputs, blocks, shapes, loss, output, parameter/latency budget, and **what is
deliberately left to Claude**. Read the spec back and get approval before implementing.

### Step 5: Set up the experiment system

Create `experiments/LOG.md`, `INBOX.md`, `REJECTED.md`, `FINDINGS.md`, `COMPUTE.md` from templates.
`EXP-000` is always **reproduce the provided baseline** (or the starter bot). It validates the data
pipeline, the metric implementation and the submission path before any idea is tested.

### Step 6: Set up version control

Follow `protocols.md` § Git: `.gitignore` for data, checkpoints and raw results; the directory
layout from the archetype; first commit `[harness] initial setup`.

### Step 7: CLAUDE.md and rules

Create `CLAUDE.md` (≤120 lines) from the template: one-line task, role contract, routing table,
status block, stop points, delegation. Generate the standard rule set from `templates.md` § Rules
plus the archetype's extra rules. Adjust every glob to the real tree — a glob that matches nothing
is a rule that silently never fires.

### Step 8: Present and confirm

Before creating files, show: archetype, directory tree, eval protocol summary, the baseline spec,
rule files with their globs, and deferred components with their triggers. Wait for explicit
confirmation, then create everything and commit.

### Step 9: Operate

Each operation is specified in `protocols.md`. Summary:

| User says | Claude does |
|---|---|
| "process inbox" | Turn each INBOX item into a falsifiable hypothesis + EXP draft; grep `REJECTED.md` for near-duplicates and say so; ask which to run |
| "new experiment: …" | Draft `EXP-NNN.md` (hypothesis, single variable, eval config, pre-registered criterion, cost estimate); branch; wait if above threshold |
| "run EXP-N" | Clean-tree check → run all seeds → write run manifests → update `COMPUTE.md` |
| "log results" | Fill Result, propose Verdict, update `LOG.md`; REJECT also → `REJECTED.md`; suspicious gain → leakage audit first |
| "status report" | Current baseline, last 5 verdicts, open experiments, budget left, days to deadline |
| "retract …" | Retraction procedure: mark, never delete, propagate to FINDINGS / LOG / CLAUDE.md |
| "freeze" / "submit" | The archetype's submission gate |
| "postmortem" | `experiments/POSTMORTEM.md` |

---

## Important Principles

**The human thinks; Claude executes and keeps the books.** The harness exists so the human spends
attention on ideas and architecture while every experiment is run the same way, recorded the same
way, and reproducible from its commit.

**Pre-register, then look.** The success criterion is written in the EXP file before the run.
Changing it after seeing results is recorded as a deviation, never applied silently.

**Too good is a bug until proven otherwise.** A result well above the running best triggers the
leakage audit before it may be logged as ADOPT. Most "breakthroughs" in financial ML are leaks.

**One variable per experiment.** If two things change, the result can't be attributed. A sweep is
one experiment varying one parameter.

**Seed variance is the noise floor.** An effect smaller than the spread across independent seeds
is not an effect. Single-seed results are labelled `PRELIMINARY` and never promoted.

**Headlines are measured the way they'll be realized.** A number that will be quoted (CAGR, score,
win rate) comes from the realistic path — official metric code, real engine, fees, full holdout
window — not from a convenience analyzer.

**The graveyard is worth more than the trophy case.** `REJECTED.md` stops a short project from
re-testing dead ideas. `FINDINGS.md` keeps retractions visible with ⚠ markers — never delete a
wrong claim; strike it and say what replaced it.

**Always-loaded is the scarce resource.** CLAUDE.md stays a snapshot. History lives in LOG,
FINDINGS and git.

**Speed over ceremony.** Strictly maintain CLAUDE.md status, `LOG.md`, `REJECTED.md`,
`FINDINGS.md` and `task-spec.md`. Other docs change only on structural decisions. If bookkeeping
takes longer than the experiment, the harness is too heavy — simplify it.
