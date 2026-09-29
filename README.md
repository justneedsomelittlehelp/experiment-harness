# Experiment Harness

A Claude Code skill for projects whose work is **testing hypotheses**, not building features —
ML/prediction competitions, bot/agent competitions, and research playgrounds.

Descends from [project-harness](https://github.com/justneedsomelittlehelp/project-harness): keeps its
context-engineering core (budgets, routing table, path-scoped rules, one home per fact, an
anti-hallucination anchor) and replaces the phase roadmap with an experiment system.

## The division of labor

| Human | Claude |
|---|---|
| Hypotheses, research direction | Falsifiable experiment specs, prior-art checks |
| Model / agent architecture | Exact implementation, flags ambiguities |
| Eval protocol, promotion criteria | Runs seeds and sweeps, leakage audits |
| Promote / abandon decisions | Proposes verdicts with evidence |
| Spend above threshold | Compute ledger, stops idle machines |
| — | Experiment log, findings, retractions, git |

## What gets created

```
CLAUDE.md                  # role contract, routing table, status snapshot, stop points
docs/
  task-spec.md             # anchor: data, metric, constraints, deadlines (stamped with source + date)
  eval-protocol.md         # splits, holdout, seeds, promotion rule, leakage checklist — frozen
  arch-<component>.md      # human-authored specs
  arch-harness.md
experiments/
  LOG.md  EXP-NNN.md  INBOX.md  REJECTED.md  FINDINGS.md  COMPUTE.md
eval/                      # locked metric implementation
.claude/rules/             # eval-freeze, experiments, src-model, task-spec-sync, compute, harness (+ archetype)
```

## Archetypes

- **Prediction** — fixed data + metric + submission, runtime-constrained inference, large-data handling.
- **Agent** — engine + opponent pool + ladder, pre-launch / launch-day / engine-patch modes, strategist inbox.

## Installation

```
/plugin marketplace add justneedsomelittlehelp/experiment-harness
/plugin install experiment-harness@experiment-harness
```

Or copy `plugins/experiment-harness/skills/experiment-harness/` into `~/.claude/skills/`.

## Usage

- "Set up an experiment harness for this competition" (with links or files)
- "Migrate this project from project-harness"
- Ongoing: "process inbox", "new experiment: …", "run EXP-7", "log results", "status report",
  "retract F3", "freeze", "submit", "engine updated", "postmortem"

## License

MIT
