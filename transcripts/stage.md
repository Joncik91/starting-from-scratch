# Stage — Model, Interpreter, Scene Table

The Stage owns all story state (variables + lists) and all interpreter logic (S1-S3 scripts + CB1-CB5 custom blocks). Sprites never write to Stage state; they either read it (View, Reactor) or drive it through broadcasts (Controller).

See `docs/superpowers/decisions/0002-stage-as-model.md` for rationale.

---

## Variables (all "For all sprites")

| Name | Initial | Purpose |
|---|---|---|
| `current_scene` | 0 (1 after init) | Program counter into the scene table. Range [1, 15] while `ending_code=0`. Written only by CB2 `apply_choice` and CB5 `resolve_ending`. |
| `food_carried` | 0 | Flag: did the ant retrieve the sugar? Set at scene 7 by CB4. Read by CB3 (id=1) and CB5. |
| `scout_trail_known` | 0 | Flag: did the ant meet the scout? Set at scene 6 by CB4. Read by CB3 (id=2) and CB5. |
| `has_orders` | 0 | Flag: did the ant hear the queen's briefing? Set at scene 2 by CB4. Read by CB3 (id=3) and CB5. Gates choice C at scene 9. |
| `ending_code` | 0 | 0=playing, 1/2/3/4 at endings 12/13/14/15. Set by CB5 (primary) and defensively by CB4. Read by CB2 as the E3 guard. |
| `ready` | 0 | Init guard. 0 during S1, 1 after first render dispatched. V2 and C1 ignore input while `ready=0`. |

Watchers visible during development: `current_scene`, `food_carried`, `scout_trail_known`, `has_orders`, `ending_code`. Hidden: `ready` and all lists. This is a debug convenience, not a runtime requirement.

---

## Lists (all "For all sprites", initially empty)

All eight lists are parallel: row N across every list describes scene N. Lengths are always equal. The `add_scene` custom block (CB1) is the ONLY writer.

| List | Purpose |
|---|---|
| `scene_text` | Prose shown to the player. |
| `scene_choice_a` | Label for choice A. |
| `scene_choice_b` | Label for choice B. |
| `scene_next_a` | Scene id to jump to if A picked. 0 = "no such choice" (E2 guard in CB2). |
| `scene_next_b` | Scene id to jump to if B picked. 0 = "no such choice". |
| `scene_flag_required` | Numeric flag id (0=none, 1=food_carried, 2=scout_trail_known, 3=has_orders) that gates choice C. See ADR-0003. |
| `scene_choice_c` | Label for conditional choice C. Empty string when no C. |
| `scene_next_c` | Scene id if C picked. 0 = "no C on this scene". |

**Parallel-list invariant** (enforced by CB1 being the only writer):
`length(scene_text) == length(scene_choice_a) == ... == length(scene_next_c)`

---

## Scripts and Custom Blocks

| Section | Name | Status |
|---|---|---|
| S1 | `when green flag clicked` — Initialize | **Task 3** |
| S2 | `when I receive choose_a` | **Task 5** |
| S2 | `when I receive choose_b` | **Task 5** |
| S2 | `when I receive choose_c` | **Task 5** |
| S3 | `when I receive render_scene` — Render dispatcher | **Task 4** |
| CB1 | `add_scene (text)(a)(b)(na)(nb)(fid)(c)(nc)` | **Task 3** |
| CB2 | `apply_choice (letter)` | **Task 5** |
| CB3 | `is_flag_set (flag_id)` | **Task 6** |
| CB4 | `run_scene_side_effects (scene_id)` | **Task 5** |
| CB5 | `resolve_ending` | **Task 7** |

Each section will be filled in by its owning task with: Purpose paragraph, Contract (where applicable), and a pseudocode block listing matching the Scratch blocks exactly.
