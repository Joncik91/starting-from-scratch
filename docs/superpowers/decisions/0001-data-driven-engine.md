# ADR-0001 — Data-Driven Engine via Parallel Lists

**Status:** Accepted
**Date:** 2026-04-22

## Context

The story needs ~14 scenes with branching choices, flag gating, and multiple endings. Scratch offers two plausible shapes:

1. **Scripted:** each scene is its own block of scripts. Hardcoded broadcasts move the story forward.
2. **Data-driven:** scenes stored as rows in parallel lists; one interpreter reads the lists and renders.

## Decision

Data-driven. Scenes are rows across 8 parallel Scratch lists on the Stage; a single render/apply loop interprets them.

## Rationale

- **Learning goal alignment.** The author is filling in CS fundamentals and explicitly asked for an approach that teaches "separation of data from logic." Data-driven is the canonical shape of that lesson.
- **Open/closed.** Adding a scene requires adding a row, not new scripts. The engine is closed to modification, open to extension.
- **Reviewability.** A reviewer can skim the 14 `add_scene` calls and see the entire story shape in one place. Scripted Scratch projects scatter logic across sprites in a way that is hard to review.

## Alternatives rejected

- **Scripted per-scene:** doesn't teach the intended lesson. Adding a scene means editing scripts. Scales poorly as the story grows.
- **Packed delimited strings (one string per scene):** Scratch's string operators are clunky; debugging string indexing would dominate the work. Rejected after brief consideration.

## Consequences

- The `add_scene` custom block is the single writer to the scene table. This mechanically protects the parallel-list invariant.
- Bugs that would have been "forgot to add a broadcast" in the scripted design become "wrong value in a list cell," which is easier to spot in list watchers at debug time.
- There is one interpretive layer (`apply_choice` + `render_text`) that must be correct. All story correctness depends on it. This is accepted as a concentration of risk — it is also a concentration of *learning*.
