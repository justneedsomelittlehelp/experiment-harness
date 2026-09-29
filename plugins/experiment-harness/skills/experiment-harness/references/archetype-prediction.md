# Archetype: Prediction Task

A fixed dataset, a metric, and usually a submission with runtime limits. Examples: an order-book
sequence-forecasting competition scored by weighted Pearson under a 1-vCPU inference limit; a
Kaggle contest; a personal trading-model research project (no submission, pristine holdout).

---

## Directory Layout

```
CLAUDE.md
docs/
  task-spec.md
  eval-protocol.md
  arch-<component>.md          # one per model family the human specifies
  arch-data-pipeline.md        # only if preprocessing is non-trivial (sharding, caching, features)
  arch-harness.md
experiments/                   # LOG, EXP-NNN, INBOX, REJECTED, FINDINGS, COMPUTE
eval/
  metric.py                    # exact reimplementation of the official metric, unit-tested
  validate.py                  # scores a predictions file on val/holdout
  splits.json                  # frozen split definitions + data hash
src/
  data/                        # loaders, sharding, causal feature code
  models/
  train.py                     # one entry point, all knobs via config
configs/                       # one YAML per EXP arm
submission/                    # solution entry point + exporter (ONNX etc.)
scripts/
  launch.sh                    # cost estimate → run → stop instance
  submit_gate.sh
results/                       # gitignored; manifests + metrics per EXP/seed
data/                          # gitignored
.claude/rules/
```

---

## Eval Protocol Defaults to Propose

- **Split:** hold out a slice of the *training* set as internal validation; treat the official
  validation set as holdout if the leaderboard/test is hidden. Split by independent unit (sequence,
  day, instrument), never by row.
- **Official metric reimplemented and unit-tested** against any reference implementation or the
  provided baseline score. EXP-000 must reproduce the baseline within noise.
- **Seeds:** 3 independent per arm. Promotion: mean Δ > larger range AND holdout not worse.
- **Loss ↔ metric alignment** is an architecture decision (human), but Claude flags mismatch, e.g.
  training MSE when the metric is amplitude-weighted correlation.
- **Leaderboard discipline:** the public leaderboard is a noisy holdout with a daily budget. Submit
  only promoted models; log each submission score in LOG.md against local holdout. Divergence is
  information, not a target to optimize.

## Leakage Checklist Seeds (add task-specific items)

- [ ] Normalization / scaling statistics are causal (online or from train only) — never computed
      over a whole sequence at inference-time-inaccessible steps.
- [ ] Features at step t use only rows ≤ t. Test: shuffle future rows, predictions at t unchanged.
- [ ] Label horizon vs. split boundary: embargo ≥ horizon when units are time-contiguous.
- [ ] Thresholds / filters / ensembles weights chosen on internal val, not on holdout.
- [ ] Headline numbers from the realistic path (official metric code; for trading, real engine with
      fees and the full holdout window — not active-days annualization).

---

## Runtime-Constrained Inference

When the task limits inference (CPU cores, total time, model size):

1. In `task-spec.md`, derive the per-step budget with the arithmetic shown; if the test size is
   unknown, assume the validation size as the worst case and mark it UNKNOWN.
2. EXP-000 also measures the provided baseline's per-step latency under the real constraint
   (e.g. `taskset -c 0`, 1 intra-op thread, same runtime such as ONNX Runtime). Record it.
3. Every EXP that changes the model reports: params, export size, p50/p99 per-step latency, and
   projected total time as % of the limit. Over the margin in `eval-protocol.md` → REJECT regardless
   of score.
4. Determinism check (two runs, identical outputs) before any submission.
5. Ensembles/multi-seed averaging multiply latency — treat as an architecture decision; propose
   distillation when it would exceed budget.

## Large Data

- Compute in-memory size in `task-spec.md` (rows × features × bytes). If it exceeds RAM, the data
  pipeline is sharded by independent unit into memory-mappable files; this is recorded in
  `arch-data-pipeline.md`.
- Preprocess once, store on persistent storage (e.g. a network volume), record a data hash in
  `eval/splits.json`; every manifest carries that hash.
- Long sequences: truncated BPTT with carried state; chunk length is a config knob.

---

## Submission Gate (`scripts/submit_gate.sh` + rule)

1. Candidate is a promoted baseline tag (`base-NNN`), tree clean.
2. Export (e.g. ONNX) and verify exported outputs match the training framework within tolerance.
3. Package check: required entry file at root, size limit, only allowed libraries, no network use.
4. Local full-run under the real constraint: total projected time within margin; deterministic.
5. Score the packaged artifact through `eval/validate.py` on holdout — must match the EXP result.
6. Log `submitted base-NNN at <time>` in LOG.md; later record the leaderboard score beside it.
7. Respect the daily submission cap; track remaining submissions in CLAUDE.md Status near deadline.

Extra rule file:

```markdown
---
paths:
  - "submission/**"
  - "scripts/submit_gate.sh"
---
# Submission
- Only promoted tags are submitted. Run submit_gate.sh; never upload a package it didn't produce.
- Keep the inference path free of training-only dependencies, network calls and nondeterminism.
- Latency/size limits → `docs/task-spec.md`.
```

## Prize / Verification Readiness

Competitions often require reproducible training code with fixed seeds, a technical report and a
code-review call. The harness already produces most of this: keep `train.py` + configs runnable from
a tag, and draft the report from FINDINGS + POSTMORTEM.

---

## Research Playground Variant (no external deadline)

- Replace the submission gate with a **pristine holdout** declared at setup and touched only for
  promotion decisions; record every touch in LOG.md.
- Replace "deadline" in Status with the current research question.
- Stop points unchanged. Freeze mode unused.
