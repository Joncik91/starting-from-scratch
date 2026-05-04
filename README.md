# Starting from Scratch — *A Worker Ant's Urgent Day*

[![CS50: Starting from Scratch](https://img.shields.io/badge/CS50-Starting%20from%20Scratch-A6192E)](https://cs50.harvard.edu/x/)
[![Built in Scratch 3](https://img.shields.io/badge/Scratch-3.0-FFAB19?logo=scratch&logoColor=white)](https://scratch.mit.edu/)
[![Endings: 4](https://img.shields.io/badge/endings-4-E8954A)]()
[![Scenes: 14](https://img.shields.io/badge/scenes-14-E8954A)]()
[![License: MIT](https://img.shields.io/badge/license-MIT-E8954A.svg)](LICENSE)

An interactive branching story in Scratch, built as a data-driven interpreter to practice CS fundamentals (state machines, separation of data from logic, MVC-style responsibility splits).

**Play:** Load `project/ant-adventure.sb3` in Scratch (https://scratch.mit.edu → File → Load from your computer). Then click the green flag.

## What this project is

A worker ant is sent on an urgent mission. The player makes choices that set flags; flags gate later choices and determine which of four endings you reach.

Under the hood:

- **14 scenes** stored as rows in **8 parallel lists** on the Stage.
- **One interpreter loop** reads the current scene, renders it, waits for input, applies the choice, repeats.
- **3 flags** (`food_carried`, `scout_trail_known`, `has_orders`) — each gates a real choice or ending.
- **4 distinct endings.**
- **3 sprites** with separated roles: Worker Ant (input), Narrator Scroll (view), Beetle (reactor).
- **5 custom blocks** (`add_scene`, `apply_choice`, `is_flag_set`, `run_scene_side_effects`, `resolve_ending`) that keep the engine DRY.

## How to read this repo

- **`docs/superpowers/specs/`** — the design spec. Start here.
- **`docs/superpowers/decisions/`** — ADRs explaining *why* the design is shaped the way it is.
- **`docs/superpowers/plans/`** — the implementation plan (written after the spec, before building).
- **`transcripts/`** — human-readable documentation of every sprite's scripts, block by block, with comments that mirror the comments inside Scratch itself. **This is the source of truth for code review.**
- **`project/ant-adventure.sb3`** — the Scratch project file (binary). Download it, open in Scratch, run it. Not reviewable on GitHub directly.
- **`project/assets/`** — source art for the sprites and backdrop.

## How to play

1. Open the `.sb3` in Scratch (scratch.mit.edu → File → Load from your computer) or visit the published link above.
2. Click the green flag.
3. Use mouse clicks on choice labels, or press keys `1`/`2`/`3` to pick choice A/B/C.
4. Reach one of the four endings. Click the green flag to replay.

## Assignment requirements (CS50 "Starting from Scratch")

All assignment requirements are met. See `docs/superpowers/specs/2026-04-22-ant-adventure-design.md` §8 for the full cross-check.

## Commit conventions

Commits carry **what** in the subject line, **why** and **where** in the body, plus an explicit **when** / section tag. Template in the spec (§7).

---

## License

MIT — see [LICENSE](LICENSE). Covers the Scratch project, the
transcripts, and the docs in this repo. The CS50 course materials
themselves are subject to the [course's own terms](https://cs50.harvard.edu/x/).

## Status

- [x] Spec approved (2026-04-22)
- [x] Plan approved (2026-04-22)
- [x] Assets created (Task 1)
- [x] Scratch project scaffolded (Task 2)
- [x] Scene table + initialize (Task 3)
- [x] Render path (Task 4)
- [x] Interpreter + keyboard input (Task 5)
- [x] Choice-C flag gate (Task 6)
- [x] Ending resolver — all four paths pass (Task 7)
- [x] Click input (Task 8)
- [x] Beetle reactor (Task 9)
- [x] Comment sweep + test matrix (Task 10)
- [x] Published / README updated (Task 11)

All spec §8 requirements met. See `transcripts/stage.md` §"Test Matrix Results" for the final verification log.
