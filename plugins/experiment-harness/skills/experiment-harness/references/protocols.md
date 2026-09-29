# Protocols

Procedures for every recurring operation. SKILL.md Step 9 routes here.

---

## Experiment Lifecycle

### 1. Draft (from "new experiment" or "process inbox")
1. **Prior-art check.** Grep `REJECTED.md`, `FINDINGS.md` and `LOG.md` with 2–4 keyword variants of
   the idea. If anything overlaps, quote it to the human *before* drafting and ask whether the new
   version differs in a way that matters.
2. Write the hypothesis as one falsifiable sentence. If the idea can't be falsified with the current
   eval protocol, say so and propose what would make it testable.
3. Identify the **single variable**. If the idea needs two changes, propose two EXPs (or a factorial
   design with the human's OK).
4. Pre-register the criterion using the promotion rule in `eval-protocol.md`.
5. Estimate cost (seeds × run time × rate). Above threshold → stop and ask.
6. If the change touches architecture → stop point; get the human's approval of a spec version bump
   first.
7. Create `experiments/EXP-NNN.md` (Status: DRAFT), add the LOG row with verdict `—`.

### 2. Run
1. `git status` must be clean. Create branch `exp/NNN-<slug>` from the current baseline tag.
2. Commit the change: `[EXP-NNN] <change>`.
3. Launch all seeds with independent base seeds (not one run with an internal ensemble).
4. Each run writes `results/EXP-NNN/<seed>/manifest.json` (see eval protocol) and checkpoints.
5. Update `COMPUTE.md` with actual hours when runs finish. Stop the machine.
6. Set EXP Status: RUNNING → DONE.

### 3. Log
1. Fill the Result table from manifests/result files only — never from memory or console scrollback.
2. Compare against the pre-registered criterion. Report mean, min, max across seeds for both arms.
3. **Suspicion check:** if the gain exceeds the largest gain in `LOG.md` so far by a wide margin, or
   holdout beats validation, or a previously noisy metric becomes suddenly clean → run the Leakage
   Audit before proposing a verdict.
4. Propose a verdict with a one-line reason. REJECT → append to `REJECTED.md` in the same edit.
5. Write the Lesson line.
6. If the verdict changes a belief → add/update a `FINDINGS.md` entry.
7. ADOPT → stop point: ask the human to confirm promotion. On confirmation: merge to `main`, tag
   `base-NNN`, update CLAUDE.md Status.
8. Commit `[EXP-NNN] log: <verdict>`.

### 4. Process inbox
For each open INBOX row: prior-art check → hypothesis → EXP draft (not created yet) → present a
table of drafts with estimated cost and overlaps → human picks → create the chosen EXP files and
mark INBOX rows `→ EXP-NNN` or `declined (<reason>)`.

---

## Git Policy

- `main` holds only adopted states. Each adopted state is tagged `base-NNN`.
- One branch per experiment: `exp/NNN-<slug>`. Rejected branches are kept (not deleted) so the exact
  code behind any logged number can be checked out.
- Commit prefixes: `[harness]`, `[eval]`, `[EXP-NNN]`, `[spec]`, `[infra]`, `[submit]`.
- Eval changes are never mixed into an experiment commit.
- Never commit data, checkpoints or raw results; commit their hashes and paths in manifests.
- A run on a dirty tree is invalid for logging; rerun or record the diff in Deviations.
- Before any destructive git operation (force-push, branch delete, history rewrite) → ask.

---

## Statistics

**Independent seeds.** Replication means separate runs with different base seeds and different data
order. An ensemble of seeds inside one run is a *model*, not a replication.

**Default comparison (3 seeds per arm).** Report per-seed values, mean and range. Promote only if
the mean difference exceeds the larger of the two arms' ranges *and* the criterion in
`eval-protocol.md` holds. With 3 seeds, formal tests are weak; the range rule is deliberately
conservative.

**More seeds when:** the effect is within 1.5× the range, or the decision is a baseline promotion
before a deadline. Go to 5.

**Proportions (win rate, hit rate).** Use the Wilson 95% interval. Minimum sample for any claim: the
interval half-width must be smaller than the effect being claimed.

**Sequential tests (agents).** For long match runs, SPRT with H0: p=0.50, H1: p=0.55, α=β=0.05 stops
early when clear. Log the stopping rule in the EXP.

**Threshold-dependent metrics.** A metric that depends on a confidence threshold is only comparable
across seeds/models if the threshold is set by a rule (fixed quantile of predictions, or chosen on
validation) — not by a fixed number, because calibration shifts between seeds.

**Multiple comparisons.** A sweep over k values will produce a best value by chance. Confirm the
winner with a fresh 3-seed run before logging it as the result; report the sweep as exploration.

---

## Leakage Audit

Run before logging any suspicious result. Prefer delegating to a subagent that has not seen the
result, giving it the eval protocol, the EXP file and the diff.

1. **Temporal/label leakage:** does any label, normalization statistic, rolling feature or lookback
   window reach past a split boundary? Is there an embargo equal to the label horizon?
2. **Selection leakage:** were thresholds, filters, early-stopping points or hyperparameters chosen
   on the data the headline is reported on?
3. **Feature–label coupling:** can the label be computed (even partially) from the model's inputs?
   Test: a trivial rule on inputs alone — how close does it get?
4. **Implementation:** does the scorer match `eval/` exactly? Were rows dropped differently between
   arms? Is the realized-path metric (engine, fees, full window) used for the headline?
5. **Replication:** does the effect hold across 3 independent seeds?
6. Record the audit outcome in the EXP Deviations section. If a leak is found → fix, rerun, and if
   any prior claim depended on it → Retraction.

---

## Retraction

Retraction is a stop point for headline claims (anything in CLAUDE.md Status or FINDINGS marked
high confidence). The human confirms; Claude executes.

1. In `FINDINGS.md`: strike the claim, add `⚠ RETRACTED <date>`, the reason, and what replaces it.
   Never delete.
2. In `LOG.md`: mark affected rows `⚠ RETRACTED (see F<n>)`.
3. In the EXP files: set Status RETRACTED and add a dated note under Deviations.
4. In CLAUDE.md Status: replace the number with the corrected one.
5. Show the retracted and corrected numbers side by side, and decompose the gap (e.g. annualization
   window, fees, seed variance, leak) so the lesson is concrete.
6. Add the mechanism to `eval-protocol.md` Leakage Checklist if it's new (eval change → approval).
7. Commit `[harness] retract F<n>: <reason>`.

---

## Status Report

Output (chat, ≤15 lines): current baseline with score ± spread; best pending candidate; last 5
verdicts; experiments running; budget used/left and burn rate; days to deadline and freeze date;
open INBOX count; any `STALE?` rows. Then overwrite CLAUDE.md Status to match.

---

## Spec Change

Triggered by a new engine/data version, a rules update, or a corrected metric definition.

1. Fetch the sources again; rebuild the changed sections of `task-spec.md`; update the stamp.
2. Diff old vs. new spec. List affected docs, code paths and EXPs.
3. Mark EXPs whose conclusions may depend on the change `STALE?` in LOG.md (don't delete).
4. Re-run EXP-000 (baseline reproduction) on the new version before anything else.
5. Present the impact list to the human and propose which STALE? experiments to re-run.

---

## Freeze

From the freeze date in CLAUDE.md Status: Mode → FREEZE. Claude declines new hypotheses and
architecture changes by default (the human can override explicitly). Allowed: bug fixes,
robustness/timeout fixes, determinism checks, submission-gate runs, and confirming the chosen
model on holdout with extra seeds.

---

## Postmortem

At the end: fill `POSTMORTEM.md` from LOG, FINDINGS, REJECTED and COMPUTE only — every number cites
its EXP. Keep it factual: hypotheses, evidence, rejection criteria, retractions and how they were
caught. Then propose harness improvements for next time.
