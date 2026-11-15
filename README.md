# Global Talent Pre-State

Working documentation for team **Ball Tactics** for Phase 1 of
[More Than A Hack 2027](https://github.com/metrica-sports/more-than-a-hack-2027),
the ball-possession challenge organised by Metrica Sports, FC Barcelona and
Mobile World Capital Barcelona.

> **Goal:** state, for every frame of a match, which team has the ball
> (`H` / `A`), whether the ball is dead (`D`), or abstain (`X`).
>
> **Metric that decides the ranking:** `spellF1` (complete possessions recovered).
> **Floor to beat:** `spellF1` 0.2433.

This repository is the documentation **before development**: context, concepts
and work split. It contains no match code or data.

---

## Documents

| Document | Content | Format |
|---|---|---|
| [01 · What has to be done](docs/01-que-hay-que-hacer.md) | The competition explained from scratch: what you get, what you submit, how you are scored | `.md` · `.docx` |
| [02 · Concepts](docs/02-conceptos.md) | The technical terms, with the figures that make them verifiable | `.md` · `.docx` |
| [03 · Data and tools](docs/03-datos-y-herramientas.md) | What is in the data folder, which programs already exist, submission format | `.md` · `.docx` |
| [04 · Team plan](docs/04-plan-equipo.md) | Roles, data contract, calendar with gates, risks | `.md` · `.docx` |

The `.docx` files live in `docs/` next to the `.md` files, with the same name.

**If you are only going to read one thing:** start with 01.

---

## The five figures you must know

| Figure | What it is |
|---|---|
| **199,319** | Rows in the submission file (one per frame) |
| **158,857** | Frames that are actually scored |
| **41 %** | Scored frames in which the **ball is missing** |
| **17** | Real possessions in the public 5 minutes |
| **0.2433** | `spellF1` of the published floor. It has to be beaten |

---

## The idea in one sentence

Frames are not what is scored: **complete possessions** are scored. The file has
199,319 lines, but the grade depends on a few dozen stretches.

---

## Dates

| Milestone | Date |
|---|---|
| Start | 25 October 2026 |
| Gate G1 (baseline reproduced) | 1 November |
| Gate G2 (first version that beats the floor) | 8 November |
| Gate G3 (reproducibility and documentation) | 15 November |
| **Submission** | **17 November 2026** |

---

## Rules that cannot be broken

1. Labels are produced by **an algorithm**. No manual annotation.
2. **You may not use** the private ground truth or human data about this match.
3. The public 5 minutes are there to **check** the pipeline, **not** to train on
   or to pick hyperparameters.
4. **Match data is not uploaded to this repository**: it is not redistributable.
