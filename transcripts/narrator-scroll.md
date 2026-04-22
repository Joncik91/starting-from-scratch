# Narrator Scroll — View

Role: the View in the MVC split. Reads `current_scene` and the scene-table lists on the Stage and renders the current scene's text + choice labels. Also routes clicks on the scroll into `choose_a/b/c` broadcasts (shared downstream with the Worker Ant's keyboard input).

Costume: `project/assets/narrator-scroll.svg`.

---

## Scripts

| Section | Name | Status |
|---|---|---|
| V1 | `when I receive render_text` — paint scene | Implemented |
| V2 | `when this sprite clicked` — click routing | **Task 8** |

---

## V1. `when I receive render_text` — Paint scene

**Purpose:** The View's only paint operation. Reads `current_scene` and the scene-table lists on the Stage and renders (a) the scene's prose, then (b) the choice labels A and B. The choice-C gate is added in Task 6.

**Contract:**
- Runs on every `render_scene` broadcast (which is every scene transition and the initial paint).
- Does not modify any story state — pure read + paint.
- Skips A/B rendering when the label string is empty (rows 2, 5, 6, 7, 8, 10-15 have empty B; rows 10-15 have empty A too).

**Blocks (Task 6 — complete version):**

```
when I receive [render_text v]
  go to x: (0) y: (60)                         // re-center in case of drift
  show
  say (item (current_scene) of [scene_text v]) for (3) seconds    // main prose

  // choice A — skip empty labels (transition and ending scenes)
  if <not <(item (current_scene) of [scene_choice_a v]) = []>> then
    say (join [1) ] (item (current_scene) of [scene_choice_a v])) for (2) seconds
  end

  // choice B — skip empty labels
  if <not <(item (current_scene) of [scene_choice_b v]) = []>> then
    say (join [2) ] (item (current_scene) of [scene_choice_b v])) for (2) seconds
  end

  // choice C — gated by scene_flag_required + is_flag_set. Two-stage
  // check because Scratch 3.0 lacks custom boolean reporters: first
  // test if flag_required is nonzero, then call is_flag_set (which
  // sets flag_check_result), then check the result and label.
  // See CB3 and ADR-0003.
  if <not <(item (current_scene) of [scene_flag_required v]) = (0)>> then
    is_flag_set (item (current_scene) of [scene_flag_required v])
    if <<(flag_check_result) = (1)> and <not <(item (current_scene) of [scene_choice_c v]) = []>>> then
      say (join [3) ] (item (current_scene) of [scene_choice_c v])) for (2) seconds
    end
  end
```

**Verification (Task 6):**
- Scene 9 with `has_orders=1` paints labels 1, 2, 3. ✓
- Scene 9 with `has_orders=0` paints labels 1, 2 only. ✓
- Scenes with `scene_flag_required=0` never paint label 3. ✓
