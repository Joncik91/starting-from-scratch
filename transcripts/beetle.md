# Beetle — Reactor

Role: a reactor in the MVC split. Reads `current_scene` and shows/hides itself. Never writes to story state.

Costume: `project/assets/beetle.svg` (costume: beetle-idle, plus beetle-menace for animation — Task 9).

---

## Scripts

| Section | Name | Status |
|---|---|---|
| R1 | `when I receive update_reactors` — show/hide | Implemented |

---

## R1. `when I receive update_reactors` — Show/hide + wiggle

**Purpose:** The Beetle's entire contribution to the project. Appears at scene 4 (the beetle encounter) with a brief mandible-raising wiggle; hidden everywhere else. Pure function of `current_scene` — Beetle never writes story state.

**Costumes:**
- `beetle-idle` — base pose
- `beetle-menace` — mandibles slightly raised, pronotum enlarged

**Timing note:** The wiggle is two iterations of (menace → wait → idle → wait), ~0.4 seconds total. S3 (the render dispatcher on Stage) broadcasts `update_reactors` AFTER `broadcast render_text and wait`, so the Narrator's paint completes before the Beetle's animation starts — no visual jitter.

**Loop rationale:** The wiggle body is a `repeat (2)` block rather than four unrolled costume swaps. This is DRY (one swap pair expressed once, not twice), and it is the project's **single loop block** — satisfying the CS50 assignment requirement for "≥1 loop." Changing the wiggle from 2 cycles to N cycles is now a one-number edit, not a copy-paste. See the post-implementation review note in `transcripts/stage.md` §"Test Matrix Results" for the why.

**Blocks:**

```
when I receive [update_reactors v]
  if <(current_scene) = (4)> then
    show
    repeat (2)                            // the project's loop block
      switch costume to [beetle-menace v]
      wait (0.1) seconds
      switch costume to [beetle-idle v]
      wait (0.1) seconds
    end
  else
    hide
  end
```

**Verification (post-implementation review):**
- Scene 4: Beetle visible with wiggle (2 iterations). ✓
- All other scenes: Beetle hidden. ✓
- Animation does not overlap the Narrator's paint (S3 ordering holds).
- `.sb3` inspection confirms exactly one `control_repeat` block in the project, in this script. ✓
