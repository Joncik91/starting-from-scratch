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
| S1 | `when green flag clicked` — Initialize | Implemented |
| S2 | `when I receive choose_a` | Implemented |
| S2 | `when I receive choose_b` | Implemented |
| S2 | `when I receive choose_c` | Implemented |
| S3 | `when I receive render_scene` — Render dispatcher | Implemented |
| CB1 | `add_scene (text)(a)(b)(na)(nb)(fid)(c)(nc)` | Implemented |
| CB2 | `apply_choice (letter)` | Implemented |
| CB3 | `is_flag_set (flag_id)` | **Task 6** |
| CB4 | `run_scene_side_effects (scene_id)` | Implemented |
| CB5 | `resolve_ending` | **Task 7** |

Each section will be filled in by its owning task with: Purpose paragraph, Contract (where applicable), and a pseudocode block listing matching the Scratch blocks exactly.

---

## S1. `when green flag clicked` — Initialize

**Purpose:** Reset all runtime state and populate the scene table. This runs exactly once per play (per green-flag click). It is the only place `current_scene` is set to its start value (1), the only place the scene table is built, and the only place the `ready` flag is raised from 0 → 1.

**Contract:**
- Begins by setting `ready=0` so in-flight input from the previous play is ignored during reset.
- Clears all 8 scene lists before calling CB1 — the "clear + rebuild" pattern is cheaper than diffing and keeps the parallel-list invariant bulletproof.
- Raises `ready=1` only AFTER the first `render_scene` completes. This relies on `broadcast render_scene and wait` to block until the render finishes (not just until it starts).

**Blocks:**

```
when green flag clicked
  set [ready v] to (0)                        // gate input during reset

  // reset runtime state to start values
  set [current_scene v] to (1)
  set [food_carried v] to (0)
  set [scout_trail_known v] to (0)
  set [has_orders v] to (0)
  set [ending_code v] to (0)

  // clear the scene table before rebuilding (keeps parallel-list
  // invariant trivially safe — ADR-0001)
  delete all of [scene_text v]
  delete all of [scene_choice_a v]
  delete all of [scene_choice_b v]
  delete all of [scene_next_a v]
  delete all of [scene_next_b v]
  delete all of [scene_flag_required v]
  delete all of [scene_choice_c v]
  delete all of [scene_next_c v]

  // populate the scene table (15 rows in scene-id order)
  add_scene [You wake in the nest. The queen's antennae tap urgently — an important briefing.] [Listen to the queen] [Slip out to the tunnel quietly] (2) (5) (0) [] (0)
  add_scene [The queen's antennae brush yours. "The picnic humans left a sugar cube. Storm is coming. Bring it back — and HURRY."] [Onward to the tunnel exit] [] (3) (0) (0) [] (0)
  add_scene [You reach the tunnel exit. Sunlight and grass. Two paths branch before you.] [Short route — risky ground with a beetle] [Long route — quieter, may find a scout] (4) (6) (0) [] (0)
  add_scene [A massive beetle blocks the path, mandibles clicking.] [Dash past for the sugar] [Retreat — too dangerous] (7) (8) (0) [] (0)
  add_scene [You sneak out before anyone notices. The tunnel exit lies ahead.] [Continue to the tunnel exit] [] (3) (0) (0) [] (0)
  add_scene [A fellow scout greets you and shares a shortcut only the patrols know.] [Take the scout's path to the sugar] [] (7) (0) (0) [] (0)
  add_scene [The sugar cube glitters in the grass. You hoist it — heavier than you are, but ants don't quit.] [Home — time is running out] [] (9) (0) (0) [] (0)
  add_scene [Bruised and limping, you turn back empty-handed.] [Home without the sugar] [] (9) (0) (0) [] (0)
  add_scene [The nest is in sight. One last choice: sneak or report?] [Sneak back quietly] [Take the main tunnel] (11) (11) (3) [Report to the queen first] (10)
  add_scene [...] [] [] (0) (0) (0) [] (0)                  // row 10: reporting transition
  add_scene [...] [] [] (0) (0) (0) [] (0)                  // row 11: sneaking transition
  add_scene [Triumph! You haul the sugar to the larder. The colony feasts tonight.] [] [] (0) (0) (0) [] (0)         // row 12: ending 1 (triumph)
  add_scene [You return empty-handed. The colony will weather the storm on stored grain — this time.] [] [] (0) (0) (0) [] (0)   // row 13: ending 2 (empty-handed)
  add_scene [The scout's shortcut shaved hours off your trip. You arrive with sugar before the storm breaks. Your name is whispered in the tunnels.] [] [] (0) (0) (0) [] (0)      // row 14: ending 3 (hero shortcut)
  add_scene [You reported to the queen, delivered the sugar, and the colony is evacuated ahead of the storm. The queen taps her antennae in approval — full glory.] [] [] (0) (0) (0) [] (0) // row 15: ending 4 (full glory)

  set [current_scene v] to (1)                // redundant but explicit — row 1 is the entry point
  broadcast [render_scene v] and wait         // paint scene 1; wait ensures first frame is up before input
  set [ready v] to (1)                        // input is now accepted
```

**Verification (Task 3):**
- All 8 lists have length 15 after green flag. ✓
- `scene_text[1]` = "You wake in the nest..."
- `scene_text[9]` = "The nest is in sight..."
- `scene_next_c[9]` = 10
- `scene_flag_required[9]` = 3
- `scene_text[15]` = "You reported to the queen..."

---

## CB1. `add_scene (text)(a)(b)(na)(nb)(fid)(c)(nc)` — 8 inputs

**Purpose:** The only writer to the scene table. Appends one row across all 8 parallel lists in one call, keeping the parallel-list invariant (equal lengths) trivially safe. Called 15 times from S1; never called anywhere else.

**Contract:**
- All 8 inputs are always provided (positional). Empty string / 0 are valid "no value" sentinels depending on column.
- Runs synchronously; no broadcasts, no screen refresh suppression needed.

**Blocks:**

```
define add_scene (text) (a) (b) (na) (nb) (fid) (c) (nc)
  add (text) to [scene_text v]
  add (a) to [scene_choice_a v]
  add (b) to [scene_choice_b v]
  add (na) to [scene_next_a v]
  add (nb) to [scene_next_b v]
  add (fid) to [scene_flag_required v]
  add (c) to [scene_choice_c v]
  add (nc) to [scene_next_c v]
```

No inner comments — the body is one append per input in input order, self-evident given the purpose comment on the hat.

---

## S3. `when I receive render_scene` — Render dispatcher

**Purpose:** Fan out a single `render_scene` signal into the two render phases: paint first (Narrator via `render_text`), then reactors (Beetle via `update_reactors`). `broadcast render_text and wait` is critical — without it, the Beetle's animation (Task 9) can start mid-paint on slower devices, producing visible jitter. This was caught in the Sonnet peer review (spec §10).

**Contract:**
- Called by CB2 after a successful `apply_choice` step.
- Also called by S1 after the scene table is built, to paint the initial scene.

**Blocks:**

```
when I receive [render_scene v]
  broadcast [render_text v] and wait          // V1 paints; block until done
  broadcast [update_reactors v]               // R1 reacts (show/hide)
```

---

## CB2. `apply_choice (letter)` — 1 input

**Purpose:** The heart of the interpreter — advance the story state machine by exactly one step. Reads the appropriate `scene_next_*` cell, applies the E2/E3 guards, updates `current_scene`, fires scene-entry side effects, broadcasts the render. "Run without screen refresh" is checked so current_scene and ending_code are both coherent before any render tick.

**Contract (locked — DO NOT reorder):**
1. Capture `next_id` from `scene_next_<letter>` at `current_scene` BEFORE mutating current_scene.
2. Guard E2: if `next_id = 0`, no such choice — no-op.
3. Guard E3: if `ending_code > 0`, story ended — no-op.
4. Set `current_scene = next_id`.
5. Call `run_scene_side_effects(current_scene)` — operates on the scene JUST ENTERED.
6. Broadcast `render_scene`.

**Local:** Scratch custom blocks have no true locals; we use a Stage variable `next_id` as a scratchpad (watcher hidden). It is not a story-state variable.

**Blocks:**

```
define apply_choice (letter)                      // "Run without screen refresh" = TRUE
  set [next_id v] to (0)
  if <(letter) = [a]> then
    set [next_id v] to (item (current_scene) of [scene_next_a v])
  end
  if <(letter) = [b]> then
    set [next_id v] to (item (current_scene) of [scene_next_b v])
  end
  if <(letter) = [c]> then
    set [next_id v] to (item (current_scene) of [scene_next_c v])
  end
  if <(next_id) = (0)> then
    stop [this script v]                          // E2: no such choice here
  end
  if <(ending_code) > (0)> then
    stop [this script v]                          // E3: story already ended
  end
  set [current_scene v] to (next_id)
  run_scene_side_effects (current_scene)          // entry side effects on scene JUST ENTERED
  broadcast [render_scene v]
```

---

## CB4. `run_scene_side_effects (scene_id)` — 1 input

**Purpose:** Apply side effects on entry to a scene. Called by CB2 after `current_scene` is updated. The parameter is the scene being entered, not the one being left — that's what lets scene 2's entry set `has_orders`, scene 6's set `scout_trail_known`, etc.

**Contract:**
- Called exactly once per CB2 invocation.
- Does not broadcast; does not re-enter itself for the new scene. Only CB2 drives scene transitions.

**Blocks (Task 5 version — CB5 calls at rows 10/11 are stubbed as comments; Task 7 wires them in):**

```
define run_scene_side_effects (scene_id)
  // flag sets on scene entry
  if <(scene_id) = (2)> then
    set [has_orders v] to (1)                     // queen's briefing
  end
  if <(scene_id) = (6)> then
    set [scout_trail_known v] to (1)              // met the scout
  end
  if <(scene_id) = (7)> then
    set [food_carried v] to (1)                   // got the sugar
  end

  // transition-scene hooks (Task 7):
  // if <(scene_id) = (10)> then
  //   resolve_ending
  // end
  // if <(scene_id) = (11)> then
  //   resolve_ending
  // end

  // defensive ending_code sets — dead code on the normal path
  // (resolve_ending sets ending_code at the transitions), retained
  // so a future direct route into 12-15 still flags the story ended
  if <(scene_id) = (12)> then set [ending_code v] to (1) end
  if <(scene_id) = (13)> then set [ending_code v] to (2) end
  if <(scene_id) = (14)> then set [ending_code v] to (3) end
  if <(scene_id) = (15)> then set [ending_code v] to (4) end
```

---

## S2. Input handlers — `choose_a`, `choose_b`, `choose_c`

**Purpose:** Thin entry points from the Controller (Worker Ant's keyboard hats, Narrator Scroll's clicks) into the interpreter. Each hat is counted as its own script by Scratch.

**Blocks:**

```
when I receive [choose_a v]
  apply_choice [a]

when I receive [choose_b v]
  apply_choice [b]

when I receive [choose_c v]
  apply_choice [c]
```
