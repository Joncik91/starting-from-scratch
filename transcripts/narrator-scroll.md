# Narrator Scroll — View

Role: the View in the MVC split. Reads `current_scene` and the scene-table lists on the Stage and renders the current scene's text + choice labels. Also routes clicks on the scroll into `choose_a/b/c` broadcasts (shared downstream with the Worker Ant's keyboard input).

Costume: `project/assets/narrator-scroll.svg`.

---

## Scripts

| Section | Name | Status |
|---|---|---|
| V1 | `when I receive render_text` — paint scene | Implemented |
| V2 | `when this sprite clicked` — click routing | Implemented |

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

---

## V2. `when this sprite clicked` — Click routing

**Purpose:** Translate clicks on the Narrator Scroll into `choose_a/b/c` broadcasts. This duplicates C1's downstream path — both input methods funnel into the same three broadcasts; the interpreter doesn't care which input method was used.

**Click zones:** The scroll is 260 px wide, centered at x=0. The three zones are thirds by mouse-x:
- `mouse_x < -43` → choice A
- `-43 ≤ mouse_x ≤ 43` → choice B
- `mouse_x > 43` → choice C

Scratch's `say` bubbles are not clickable regions, so we use the scroll body as the input surface. Documented here so reviewers know this is intentional and not a UX regression.

**Contract:**
- Same E1 guard as C1 (check `ready`).
- Invalid choices (e.g. clicking "C" at a scene with no choice C) are caught by CB2's E2 guard — V2 does not pre-check.

**Blocks:**

```
when this sprite clicked
  if <not <(ready) = (1)>> then
    stop [this script v]                       // E1: ignore clicks during init
  end
  if <(mouse x) < (-43)> then
    broadcast [choose_a v]
  else
    if <(mouse x) > (43)> then
      broadcast [choose_c v]
    else
      broadcast [choose_b v]
    end
  end
```

**Verification (Task 8):**
- Click left third at scene 1 → advances to scene 2. ✓
- Click right third at scene 9 with has_orders=1 → choice C route (scene 10, ending 15). ✓
- Click right third at scene 2 (no C) → no-op (CB2 E2 guard). ✓
- Full path walked by clicking only: ending reached matches keyboard path. ✓
