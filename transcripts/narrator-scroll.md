# Narrator Scroll — View

Role: the View in the MVC split. Reads `current_scene` and the scene-table lists on the Stage and renders the current scene's text + choice labels. Also routes clicks on the scroll into `choose_a/b/c` broadcasts (shared downstream with the Worker Ant's keyboard input).

Costume: `project/assets/narrator-scroll.svg`.

---

## Scripts

| Section | Name | Status |
|---|---|---|
| V1 | `when I receive render_text` — paint scene | partial (Task 4; C gate Task 6) |
| V2 | `when this sprite clicked` — click routing | **Task 8** |

---

## V1. `when I receive render_text` — Paint scene

**Purpose:** The View's only paint operation. Reads `current_scene` and the scene-table lists on the Stage and renders (a) the scene's prose, then (b) the choice labels A and B. The choice-C gate is added in Task 6.

**Contract:**
- Runs on every `render_scene` broadcast (which is every scene transition and the initial paint).
- Does not modify any story state — pure read + paint.
- Skips A/B rendering when the label string is empty (rows 2, 5, 6, 7, 8, 10-15 have empty B; rows 10-15 have empty A too).

**Blocks (Task 4 version — no choice C yet):**

```
when I receive [render_text v]
  go to x: (0) y: (60)                         // re-center in case of drift
  show
  say (item (current_scene) of [scene_text v]) for (3) seconds    // main prose

  // choice A — skip if label is empty (end/transition scenes)
  if <not <(item (current_scene) of [scene_choice_a v]) = []>> then
    say (join [1) ] (item (current_scene) of [scene_choice_a v])) for (2) seconds
  end

  // choice B — skip if label is empty
  if <not <(item (current_scene) of [scene_choice_b v]) = []>> then
    say (join [2) ] (item (current_scene) of [scene_choice_b v])) for (2) seconds
  end

  // choice C rendering added in Task 6 (with is_flag_set gate)
```

**Verification (Task 4):**
- Green flag paints scene 1 prose, then label "1) Listen to the queen", then "2) Slip out to the tunnel quietly". ✓
- Nothing advances without input (correct — input is Task 5).
