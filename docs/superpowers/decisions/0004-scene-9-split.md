# ADR-0004 — Splitting Scene 9 to Keep Single-Flag Gating

**Status:** Accepted
**Date:** 2026-04-22

## Context

The return trip needs to produce four distinct endings based on the combination of three flags (`food_carried`, `scout_trail_known`, `has_orders`). Two separate concerns:

1. **Choice gating:** The `scene_flag_required` slot (ADR-0003) can gate exactly one of the three choices per row. The return-trip decision wants gating to drive one player-visible fork.
2. **Ending routing:** `scene_next_a/b/c` values are fixed integers written once at init. They can't dynamically pick "scene 10 if food=1, scene 11 if food=0."

## Decision

Split the return trip into three scenes:

- **Scene 9 — the one player-agency point.** Three choices: A "Sneak back quietly" (always), B "Take the main tunnel" (always), C "Report to the queen first" (gated on `has_orders` via `scene_flag_required = 3`). Only choice C is gated. A/B both route to scene 9c; C routes to 9b.
- **Scene 9b — transition (reporting path).** No player input. On entry, `run_scene_side_effects` calls `resolve_ending`, which reads flags and routes to the correct ending.
- **Scene 9c — transition (sneaking path).** Same mechanism as 9b; flag combination chooses ending 10/11/12.

Player agency lives at scene 9. The flag-driven routing at 9b/9c is not agency removal — it's computing the narrative consequence of choices made earlier (at scenes 2, 6, 7, 8, where the flags were actually set).

An earlier version of this decision claimed both 9b and 9c had their own choice C gated on `scout_trail_known`. That was wrong on two counts: (a) `scene_next_*` can't express flag-dependent routing, and (b) at 9b/9c there's no meaningful new choice left to make — the journey is over, the outcome is a function of what the player already accomplished. The `resolve_ending` helper (CB5) is the correct shape.

## Rationale

- **Cheap rows, clean schema.** Two extra rows in the scene table; no second `scene_flag_required_2` column sitting at 0 in most rows.
- **Each scene does one thing.** Scene 9 is a decision point. 9b/9c are transitions. 10-13 are endings. Roles don't overlap.
- **`resolve_ending` is a single, auditable combiner.** The one place where multiple flags combine into a routing decision. Adding/changing an ending means editing one custom block, not a scattered set of `scene_next_*` values.

## Alternatives rejected

- **Widen every row with a second flag-gate slot.** Every scene pays a column-width cost for a constraint only one scene had. Not DRY.
- **Store the gate as a mini-expression** (e.g., `"2&3"` = both flags required). String parsing at render time — an interpreter inside the interpreter. Rejected on complexity grounds.
- **Make 9b/9c player-driven scenes with their own gated choice C.** Considered first; fails because `scene_next_*` is a fixed integer and can't express "scene 10 if food=1, scene 11 if food=0." The food state is already set by the time the player arrives at 9b/9c; the player has no new information to act on. A "continue" button would be agency theater.

## Consequences

- Scene table grows from the original 12 rows to 14. Still within "a few dozen puzzle pieces" scope.
- Transcripts must document the 9 → 9b/9c flow explicitly so a reviewer can trace it.
- `resolve_ending` (CB5) is the sole combiner of multi-flag routing. It also sets `ending_code` directly to avoid needing a second `run_scene_side_effects` pass on the ending scene.
- Each of the four endings (10/11/12/13) is reachable via exactly one flag combination; the test matrix in §6 of the spec lists the canonical path to each.
