# Evaluator Guide — Ball Tactics

This document is for the **More Than A Hack 2027** evaluators. It explains what
this repository contains, what it does not contain, how to review it quickly, and
which claims can be checked without access to the private ground truth.

---

## 1. What this repository is

**Team:** Ball Tactics
**Challenge:** More Than A Hack 2027, Phase 1 — ball possession
**Organisers:** Metrica Sports, FC Barcelona, Mobile World Capital Barcelona
**Deadline:** 17 November 2026

This is the team's **pre-development documentation**: challenge analysis,
technical concepts, data/tooling notes and the work plan. It is not the code
submission. The scored artefacts — the 199,319-row CSV, the public code
repository and the technical document — are submitted through the official form.

| In this repository | Not in this repository |
|---|---|
| Challenge breakdown and scoring analysis | Match data (not redistributable) |
| Metric and failure-mode documentation | Trained models or hyperparameters |
| Data schema, frame alignment and known traps | The submission CSV |
| Team roles, data contract and calendar | The final 3–4 page technical report |

---

## 2. The task, in short

Given one match (FC Barcelona vs Athletic Club, LaLiga 2024-25) with video,
automatic tracking of players and ball, tactical tagging and homography, produce
a CSV with one row per video frame:

```
frame,label
11185,A
11186,A
11187,A
...
18240,D
...
18684,H
```

| Label | Meaning |
|---|---|
| `H` | Home team (Barcelona) has controlled possession, ball in play |
| `A` | Away team (Athletic) has controlled possession, ball in play |
| `D` | Ball is **not in play** (dead ball) |
| `X` | Abstain |

"Having the ball" means **controlled possession**, not last touch: a deflection
or a loose ball does not switch possession, and `D` means *play is stopped*, not
*the ball is not visible*.

---

## 3. How submissions are scored

The ranking is decided by **`spellF1`**, which matches whole possessions by
temporal overlap at five thresholds (30 %, 40 %, 50 %, 60 %, 70 %) and averages.
Frame-level `macroF1` is reported but does not decide the ranking.

Consequences the team designed around:

- **Extra `D` labels are the most expensive error**: they split one possession
  into two (`H D H` counts as two possessions).
- **Short invented possessions are punished**: the raw baseline reports 131
  possessions on the public window where there are 17.
- **`D` is not "ball lost"**: an offside is `D` with the ball clearly visible; a
  2-second occlusion during play is not `D`.
- **No safe minimum possession length**: real possessions can last 0.8 s.
- **Noise band:** 0.04 in `spellF1` (0.01 in `macroF1`); smaller differences are
  not meaningful.
- **Published floor to beat:** `spellF1` 0.2433 (`baseline_smooth50`).

The reference implementation is `evaluation/possession_metrics.py` in the
[official challenge repository](https://github.com/metrica-sports/more-than-a-hack-2027);
where documentation and code disagree, the code wins.

---

## 4. Review path (about 10 minutes)

1. **`docs/01-que-hay-que-hacer.md`** — the competition from zero, including the
   per-frame vs. per-possession distinction and the worked example of how
   `macroF1` can look acceptable while `spellF1` collapses.
2. **`docs/02-conceptos.md`** — the technical vocabulary and the figures behind
   each claim (199,319 frames; 158,857 scored; 41 % ball absence; 17 possessions
   in the public 5 minutes; 0.8 s to 42.4 s possession spread).
3. **`docs/03-datos-y-herramientas.md`** — data layout, frame alignment
   (`frame_id - 1`), units (pitch 105.3 × 68.0 m), existing baseline, official
   scorer and validator, and the submission format's fatal errors.
4. **`docs/04-plan-equipo.md`** — five roles mapped to pipeline stages, the frozen
   data contract, the rule that only one role emits the label array, and the
   calendar with gates G1–G3.

A machine-readable summary of the analysis also lives in `AGENTS.md`.

---

## 5. What is verifiable from this repository alone

- **Task framing and metric reading** are consistent across all four documents
  and with the official challenge specification.
- **Submission format requirements** (header `frame,label`, exactly 199,319 data
  rows, frames 0–199318 in ascending order, labels in `{H, A, D, X}`, no empty
  cells, UTF-8) match the official validator.
- **Known traps are documented, not hidden**: the 164,488-frame broadcast file,
  the ATD `-1` offset, case-sensitive Smart Tagging columns, 0–1 normalised
  coordinates, and the scorer reading every CSV left in `evaluation/results/`.
- **Rules are respected on paper**: no manual annotation, no private ground truth,
  no training or hyperparameter tuning on the public 5 minutes, and no match data
  committed to the repository (enforced by `.gitignore`).

What cannot be verified here is the **score itself**: the submission CSV, the
generating code and the technical report are delivered separately, and the
private ground truth is unavailable to the team by design.

---

## 6. Questions an evaluator may want to ask

- Does the submitted CSV pass `validate_submission.py` before any scoring?
- Is the pipeline deterministic (same input → byte-identical CSV)?
- Are dead-ball intervals produced from evidence (tactical tags, restarts,
  possession state) rather than from ball-detection gaps?
- Is the number of possessions on the public window in the right order of
  magnitude (tens, not hundreds)?
- Were hyperparameters chosen without looking at the public-window score?

---

## 7. Status at the time of writing

| Item | Status |
|---|---|
| Challenge analysis and concepts | Complete (this repository) |
| Data contract and role split | Complete (this repository) |
| Reproducible pipeline | Tracked in the team's code repository |
| Submission CSV | Generated from the pipeline |
| Technical report | Submitted with the CSV |
| Repository contains match data | **No, by design** |

---

## 8. Contact

Team **Ball Tactics** — repository:
<https://github.com/Ball-Tactics/Global-Talent-Pre-State>
