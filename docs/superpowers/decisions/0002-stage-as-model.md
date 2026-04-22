# ADR-0002 — Stage as Model

**Status:** Accepted
**Date:** 2026-04-22

## Context

In a data-driven Scratch project, story state (variables, lists, interpreter logic) has to live somewhere. Candidates:

1. The Stage itself.
2. A dedicated hidden "engine" sprite.
3. State distributed across the existing sprites (Ant, Narrator, Beetle).

## Decision

All story state and interpreter logic live on the Stage. Sprites are pure in their role (Ant = input, Narrator = view, Beetle = reactor).

## Rationale

- **Global reachability.** Stage variables and lists are readable by every sprite without plumbing. This is exactly what a Model layer needs.
- **No distraction.** The Stage has no costume/motion behavior to muddle with. It's the idiomatic Scratch home for "pure logic + data."
- **No sprite count padding.** A hidden engine sprite inflates the sprite count with no user-visible value; the Stage gives us the same capability for free.

## Alternatives rejected

- **Dedicated hidden engine sprite.** Functionally equivalent to Stage-as-Model, minus the padding cost. The marginal "encapsulation" benefit does not exist in practice — Scratch's "for this sprite only" variables aren't readable from other sprites anyway, so any cross-sprite use requires global vars regardless.
- **Distributed state.** The Scratch anti-pattern. Leads to "which sprite owns this flag?" confusion, duplicated logic, and is impossible to keep DRY. Explicitly avoided.

## Consequences

- Every sprite's scripts read Stage globals freely but never write to them except through a broadcast that triggers a Stage-side handler. The one exception is deliberate and documented: `ending_code` and flags are written only by `run_scene_side_effects` on the Stage.
- Reviewers can see the entire story state on the Stage with its variable watchers during debugging.
- The sprite-role boundaries are enforced by convention + code review, not by Scratch's type system (it has none).
