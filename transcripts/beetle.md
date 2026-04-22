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

**Timing note:** The wiggle is four costume swaps over ~0.4 seconds. S3 (the render dispatcher on Stage) broadcasts `update_reactors` AFTER `broadcast render_text and wait`, so the Narrator's paint completes before the Beetle's animation starts — no visual jitter.

**Blocks:**

```
when I receive [update_reactors v]
  if <(current_scene) = (4)> then
    show
    switch costume to [beetle-menace v]
    wait (0.1) seconds
    switch costume to [beetle-idle v]
    wait (0.1) seconds
    switch costume to [beetle-menace v]
    wait (0.1) seconds
    switch costume to [beetle-idle v]
  else
    hide
  end
```

**Verification (Task 9):**
- Scene 4: Beetle visible with wiggle. ✓
- All other scenes: Beetle hidden. ✓
- Animation does not overlap the Narrator's paint (S3 ordering holds).
