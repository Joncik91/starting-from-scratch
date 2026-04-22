# ADR-0003 — Numeric Flag IDs for Choice C Gating

**Status:** Accepted
**Date:** 2026-04-22

## Context

Some scenes have a conditional third choice (C) that only appears when a specific flag is set. The scene table needs a way to say "scene N's choice C is gated by flag X."

Possible encodings for the `scene_flag_required` column:

1. **Flag name as string** (`"food_carried"`, `"scout_trail_known"`, `"has_orders"`, `""`).
2. **Numeric flag id** (`0` = none, `1` = food_carried, `2` = scout_trail_known, `3` = has_orders).

## Decision

Numeric flag ids.

## Rationale

- **Cost of dispatch.** Resolving a flag name in Scratch requires string comparison (`if flag_name = "food_carried" then ...`). Numeric ids use faster integer equality and produce a cleaner if-ladder.
- **Locality of change.** Adding a flag means adding one branch to `is_flag_set`. With flag-name strings, every string-compare dispatch would need the new name added. Numeric ids centralize the mapping.
- **Authoring load.** Writing `2` is less error-prone than typing `"scout_trail_known"` 14 times without a typo.

## Alternatives rejected

- **String names.** Rejected for the reasons above. The lost readability in the data table is regained in `is_flag_set`, which is the one place the id-to-variable mapping lives.
- **One gating column per flag** (boolean columns: `requires_food`, `requires_scout`, `requires_orders`). Rejected — cost scales with n_flags × n_scenes, and most scenes gate on zero flags. This is exactly the kind of table widening that parallel-list designs discourage.

## Consequences

- The `is_flag_set` custom block is the single translation point between the numeric id in the scene table and the named boolean variables. Anywhere the engine needs to know "is flag X set?", it goes through `is_flag_set`.
- Reviewers of the scene table need the legend: `0 = none, 1 = food_carried, 2 = scout_trail_known, 3 = has_orders`. This is documented in the spec (§3) and at the `scene_flag_required` list's comment inside Scratch.
- Single-flag-per-scene is a constraint that shaped the scene-9 split. See `decisions/0004-scene-9-split.md`.
