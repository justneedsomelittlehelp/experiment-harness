---
name: experiment-harness
version: 0.1.3
description: >
  Set up and run a research harness for experiment-driven projects — ML/prediction competitions,
  bot/agent competitions, and research playgrounds — where the human owns ideas and architecture
  and Claude owns implementation, experiment execution, logging and version control. Use this skill
  when the user says "set up an experiment harness", "research harness", "set up this competition",
  "start a new experiment project", or hands over a task plus data/engine and wants Claude to run
  the experiment loop. Also use it for the harness's recurring operations: "new experiment",
  "process inbox", "run EXP-N", "log results", "retract", "status report", "freeze", "submit",
  "task/engine updated", "postmortem", "migrate from project-harness", and "upgrade harness".
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
| Task anchor | `docs/task-spec.md` | Anchor for the outside world: data schema, metric, constraints, submission, deadlines |
| Code anchor | `docs/code-map.md` | Anchor for the repo: entry points, configs, eval scripts, results layout, key functions |
| Foundations | `docs/arch-foundations.md` | Stack and dependency choices with reasons (runtime, export format, versions) |
| Eval protocol | `docs/eval-protocol.md` | Splits, holdout, seeds, promotion criterion, leakage checklist — **frozen** |
| Architecture specs | `docs/arch-{component}.md` | Human-authored model/agent specs; the implementation must match |
| Experiment system | `experiments/` | `LOG.md`, `EXP-NNN.md`, `INBOX.md`, `REJECTED.md`, `FINDINGS.md`, `COMPUTE.md` |
| Harness procedure | `docs/arch-harness.md` | Harness version, maintenance events, audits, retraction pointer |
| Rules | `.claude/rules/*.md` | Invariants that auto-load on matching file reads |

**Two anchors, one precedence rule.** `task-spec.md` is the authority for facts about the task;
`code-map.md` for facts about the repo (paths, entry points, function names). Every claim elsewhere
must be checkable against one of them in a single read. When two docs disagree, correct the anchor
first (against the source or the code), then fix the doc that drifted.

**Two retrieval paths, both needed.** The CLAUDE.md routing table is advisory and fires on
*intent* ("I'm about to change the loss"). Path-scoped rules are mechanical and fire on *file
reads*. Rules cover implementation but not planning; the routing table covers planning but can be
skipped. Generate both, for every doc that has invariants.

**The first 30 lines of every doc stand alone.** Each doc opens with a header block — *Read this
when* / *Does NOT cover* / *Related docs* / *Invariants* — enough to tell whether it's the right doc
and to act safely without reading further. *Does NOT cover* prevents confident answers about things
the doc never addressed; *Related docs* lets a reader move sideways without going back to CLAUDE.md.

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

### Context placement and auto memory

Two surfaces load in full every session: `CLAUDE.md` and Claude Code's **auto memory** index
(`~/.claude/projects/<project>/memory/`). The harness keeps both thin by giving every fact exactly
one home in the repo — and auto memory is allowed to hold **pointers only**.

- **The harness creates no memory file.** Auto memory is Claude's own, machine-local channel; don't
  structure it. Don't create a root `MEMORY.md` — Claude Code doesn't load it.
- **Pointers, not facts — written as a mini routing row.** Every memory entry uses one format:
  `When <situation> → read <path> (<what's there, one line>)`. Example:
  `When a result beats the baseline by a wide margin → read protocols.md § Leakage Audit (the 5 checks to run before logging ADOPT)`.
  The "what's there" line describes the doc's content, never its current values.
- **Only for non-obvious context.** Both CLAUDE.md and the memory index load every session, so a
  memory pointer must not repeat a CLAUDE.md routing row. Use memory for triggers the routing table
  doesn't cover: easy-to-miss situations, lessons tied to a trap, cross-doc links.
- **Never in auto memory:** experiment status, scores, seeds, verdicts, budget figures, deadlines,
  or architecture details. They go stale within days, and a retracted number surviving in memory
  gets quoted again. Their homes are CLAUDE.md Status, `LOG.md`, `FINDINGS.md`, `COMPUTE.md`,
  `task-spec.md`.
- **Allowed:** machine- or user-specific context with no repo home (e.g. "RunPod volume mounted at
  `/workspace`", "user prefers answers in Korean"), and pointers.
- Put these rules in CLAUDE.md (template § Memory) so they're read before anything is saved, and
  prune memory at every audit and after every retraction.
- **`@import` is not a context shortcut.** A file pulled into CLAUDE.md with `@path` loads in full
  at launch like the rest of CLAUDE.md. Use routing rows for long docs, never imports.
- **A rule file without `paths` loads unconditionally** — same cost as CLAUDE.md. Every rule in this
  harness has `paths`.

| Fact | Home |
|---|---|
| Current snapshot (baseline, active EXPs, budget, deadline) | CLAUDE.md Status |
| Data, metric, constraints | `docs/task-spec.md` |
| Where code lives, entry points, function names | `docs/code-map.md` |
| Why a library, runtime, version or export format was chosen | `docs/arch-foundations.md` |
| How we evaluate | `docs/eval-protocol.md` |
| What the model is | `docs/arch-<component>.md` |
| What was tried / believed / rejected | `LOG.md` / `FINDINGS.md` / `REJECTED.md` |
| Money | `COMPUTE.md` |
| Personal notes not to commit | `CLAUDE.local.md` (gitignored) |
| Machine-specific paths, user preferences, pointers | auto memory |

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
project-harness repo and upgrades between experiment-harness versions in `references/upgrade.md`. **Don't write a harness file from memory when a
template exists for it** — read the template section first.

### Step 0: Detect the mode

| Situation | Mode |
|---|---|
| Empty or new repo (an existing non-harness CLAUDE.md is merged, not replaced) | **Setup** → Steps 1–8 |
| Repo has a project-harness (`roadmap/`, `architecture_docs/`) | **Migrate** → `upgrade.md` § Migrate |
| Repo has an experiment-harness older than this skill's version | **Upgrade** → `upgrade.md` § Self-upgrade |
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

**If a `CLAUDE.md` already exists** (e.g. shipped with starter code), read it fully first. Preserve
every existing rule, convention and env note; merge the harness sections into it; don't duplicate
what's already there. The routing table goes near the top.

### Step 2: Build the task anchor

Create `docs/task-spec.md` from the template: flat tables only, stamped
`Verified against <source/version> on <date>`. Every claim about data, metric or constraints
anywhere in the repo must be checkable there. Unknowns are written `UNKNOWN — check <where>`, never
guessed. Derived budgets (e.g. per-step inference time, memory footprint) are computed here with
the arithmetic shown.

Then create `docs/arch-foundations.md`: language, frameworks, training vs. inference runtime,
export format, pinned versions, and why — especially anything forced by the scoring environment
(e.g. "Python 3.10 + ONNX Runtime, 1 thread"). Create `docs/code-map.md` once the directory layout
exists (Step 6): plain lookup tables of paths, entry points and key functions, no prose. For an
empty repo, code-map starts with the layout only and grows as code lands — never list files that
don't exist yet.

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
status block, stop points, memory rules, delegation. Create `docs/arch-harness.md` and stamp
`Harness version: experiment-harness v<this skill's version>` in its header — upgrades detect from it. Tell the user once that auto memory is
machine-local and pointer-only in this harness. Generate the standard rule set from `templates.md` § Rules
plus the archetype's extra rules. Adjust every glob to the real tree — a glob that matches nothing
is a rule that silently never fires. A rule that guards a doc mirrors that doc's Invariants block
exactly — the one duplication the harness accepts, because the two serve different retrieval paths;
changing one means changing both in the same edit.

Deferred components (e.g. a game spec before launch day) get an entry in `arch-harness.md` §
Deferred Components with their trigger, and **no routing row until the file exists** — a row pointing
at a missing file is worse than no row.

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
| "audit harness" | The audit in `arch-harness.md`, including the doc-rot check |
| "upgrade harness" | `upgrade.md` § Self-upgrade |

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

**Always-loaded is the scarce resource.** CLAUDE.md stays a snapshot and auto memory holds
pointers only. History lives in LOG, FINDINGS and git — a number that lives anywhere else will
eventually be quoted after it was retracted.

**A stale doc is worse than no doc.** A doc naming a deleted function or a moved config gets
believed. The audit greps every path and symbol the docs mention and fixes or removes what no
longer exists.

**Speed over ceremony.** Strictly maintain CLAUDE.md status, `LOG.md`, `REJECTED.md`,
`FINDINGS.md` and `task-spec.md`. Other docs change only on structural decisions. If bookkeeping
takes longer than the experiment, the harness is too heavy — simplify it.
