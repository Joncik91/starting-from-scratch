# Ant Adventure — Design Spec

**Project:** A Worker Ant's Urgent Day — a data-driven interactive story in Scratch
**Author:** Joncik (self-taught, learning CS fundamentals)
**Date:** 2026-04-22
**Assignment:** CS50 "Starting from Scratch" — implement any project of your choice in Scratch (scratch.mit.edu) subject to the sprite/script/conditional/loop/variable/custom-block requirements.
**Review status:** Design reviewed twice by an independent model (Sonnet); logic holes, flag-gating ambiguity, and broadcast race identified and corrected before implementation. See commit history for the review trail.

---

## 1. Goal and Learning Frame

Build an interactive branching story in Scratch about a worker ant sent on an urgent mission. The story is **data-driven**: scenes are rows in parallel lists on the Stage; one interpreter loop reads the scene table and renders the current scene. Player choices advance `current_scene` and can set flags that gate later choices and endings.

The project is chosen to teach CS fundamentals the author is deliberately trying to fill in:

- **State machines** — the game is literally one, with `current_scene` as the program counter.
- **Separation of data from logic** — scenes as data, interpreter as code. Adding a scene requires only data.
- **MVC-style responsibility split** — enforced by sprite roles.
- **Flags as first-class state** — distinct from program position.

The project sits above Scratch lecture demos in complexity and below full games like Oscartime / Ivy's Hardest Game, as the assignment requires.

---

## 2. Architecture Overview

### MVC split (enforced by sprite role)

| Role | Owner | Responsibility | Forbidden |
|---|---|---|---|
| **Model** | Stage (scripts + global vars/lists) | Owns all story state and interpreter logic. Every write to story state goes through here. | — |
| **View** | Narrator Scroll sprite | Renders the current scene's text and choices. Reads Model state only. | Writing any story state. |
| **Controller** | Worker Ant sprite + Narrator click handlers | Translates keyboard/mouse input into `choose_a/b/c` broadcasts. | Applying the choice itself. |
| **Reactor** | Beetle sprite | Reads `current_scene` and flags to decide whether to appear/animate. | Writing any story state. |

### The central loop

```
[player input]
  → broadcast choose_a | choose_b | choose_c
    → Stage: apply_choice(letter)
        - looks up next scene id
        - guards against invalid/post-ending input
        - updates current_scene
        - runs scene-entry side effects (may set flags or ending_code)
        - broadcasts render_scene
          → Stage: broadcast render_text (and wait)
              → Narrator repaints text + choice labels
          → Stage: broadcast update_reactors
              → Beetle shows or hides based on current_scene
  → wait for next input
```

`broadcast render_text and wait` precedes `broadcast update_reactors` to eliminate a visual race where the Beetle's animation could start mid-paint on slower devices.

### Key design property

Adding a new scene requires adding a row to the 8 parallel lists. No script changes. This is the payoff of the data-driven choice and the fundamental CS lesson of the project.

---

## 3. Data Model

Scratch offers only variables (scalars) and lists (1-indexed arrays). There are no structs, so a "scene" is represented as a row spanning 8 parallel lists. This is exactly how relational database rows are stored at the physical level.

### Variables (Stage globals)

| Name | Type | Purpose |
|---|---|---|
| `current_scene` | number (1..14) | The program counter. Points into the scene table. |
| `food_carried` | boolean (0/1) | Flag: did the ant retrieve the sugar? |
| `scout_trail_known` | boolean (0/1) | Flag: did the ant meet the scout on the long path? |
| `has_orders` | boolean (0/1) | Flag: did the ant hear the queen's briefing? |
| `ending_code` | number (0..4) | 0 during play; 1/2/3/4 at an ending. |
| `ready` | boolean (0/1) | Init guard: 0 during initialize, 1 once first render is dispatched. V2 input ignores clicks while `ready = 0`. |

### Lists (Stage globals — the scene table)

All lists are the **same length**. Row N across every list describes scene N. This invariant is mechanically protected by the `add_scene` custom block (the only writer).

| List | Holds | Example (scene 1) |
|---|---|---|
| `scene_text` | prose shown to the player | `"You wake in the nest. The queen's antennae tap urgently."` |
| `scene_choice_a` | label for choice A | `"Listen to the queen"` |
| `scene_choice_b` | label for choice B | `"Slip out to the tunnel"` |
| `scene_next_a` | scene id if A picked | `2` |
| `scene_next_b` | scene id if B picked | `5` |
| `scene_flag_required` | **numeric** flag id to gate choice C (0=none, 1=food_carried, 2=scout_trail_known, 3=has_orders) | `0` |
| `scene_choice_c` | label for conditional choice C, `""` if unused | `""` |
| `scene_next_c` | scene id if C picked, `0` if unused | `0` |

**Why numeric flag ids rather than flag names:** storing names would force a string-compare dispatch ladder per render. Numeric ids resolve via `is_flag_set` in one small if-ladder (O(n_flags)), not per scene. See `decisions/0003-numeric-flag-ids.md`.

### The scene table (14 scenes)

```
 1  Nest (start)                → 2 (listen) or 5 (sneak)
 2  Queen's briefing            → 3              side-effect: has_orders=1
 3  Tunnel exit                 → 4 (short/risky) or 6 (long/scout)
 4  Beetle encounter            → 7 (fight past) or 8 (retreat)
 5  Sneak out early             → 3              (no has_orders)
 6  Scout meets you             → 7              side-effect: scout_trail_known=1 (bypasses beetle)
 7  Sugar cube found!           → 9              side-effect: food_carried=1
 8  Injured, retreat            → 9
 9  Return trip                 → 9b (report, if orders) or 9c (sneak)
9b  Reporting in (transition)   → resolves to ending 13 via flag-driven routing
9c  Sneaking back (transition)  → resolves to ending 10/11/12 via flag-driven routing
10  Ending: triumph             ending_code=1  (food_carried=1, scout=0, orders=0)
11  Ending: empty-handed        ending_code=2  (food_carried=0)
12  Ending: hero shortcut       ending_code=3  (food_carried=1, scout=1, orders=0)
13  Ending: full glory          ending_code=4  (orders=1 — reported in; food/scout flavor the text)
```

**Scene 9 layout.** Scene 9 is the single player-agency point on the return trip. It offers:
- **Choice A (default):** "Sneak back quietly" → scene 9c. Always visible.
- **Choice B (default):** "Take the main tunnel" → scene 9c. Always visible. (Narrative flavor — different path, same destination.)
- **Choice C (gated by `scene_flag_required = 3`, has_orders):** "Report to the queen first" → scene 9b. Only rendered if `has_orders = 1`.

Only choice C is flag-gated. The gate machinery (`scene_flag_required` + `is_flag_set`) is used exactly as designed: it controls whether C appears.

**Scenes 9b and 9c are transition scenes, not decision scenes.** Each has no player choices. On entry, `run_scene_side_effects` calls the ending resolver (see CB5 below) which reads the flags and sets `current_scene` directly to the correct ending (10/11/12/13). A brief "..." transition is rendered by the Narrator for narrative feel, then the ending scene is entered on the next tick.

Player agency is at scene 9 (sneak vs report); the flag-driven routing at 9b/9c computes the narrative consequence of flags collected earlier in the story. This is not agency removal — the agency was spent when the flags were set (scenes 2, 6, 7, 8). See `decisions/0004-scene-9-split.md`.

### Parallel-list invariant

At all times:
`length(scene_text) == length(scene_choice_a) == ... == length(scene_next_c)`

Maintained by:
- Initialize script clears all 8 lists before populating.
- **Only** `add_scene` writes to the lists, and it appends one row across all 8 in one call.

A future reader who tries to `add X to [some_scene_list]` outside `add_scene` breaks this invariant. The design doc and the `add_scene` block's comment both call this out.

---

## 4. Scripts and Custom Blocks

### Scripts (9 hat-block scripts total)

Scratch counts each hat block (`when I receive ...`, `when key pressed`, etc.) as its own script.

#### Stage (Model)

**S1. `when green flag clicked` — Initialize**
- Set `ready = 0`.
- Reset all variables: `current_scene=1`, all flags=0, `ending_code=0`.
- Delete all items from each of the 8 scene lists.
- Call `add_scene(...)` 14 times to populate the scene table.
- Broadcast `render_scene` (and wait).
- Set `ready = 1`.

**S2. `when I receive choose_a` / `choose_b` / `choose_c` — Input handlers (3 hats)**
- Each one calls `apply_choice("a")` / `("b")` / `("c")`. The three hats collapse to one logical script but Scratch counts them as three. This is intentional — each input kind gets its own entry point.

**S3. `when I receive render_scene` — Render dispatcher**
- `broadcast render_text and wait` (Narrator paints)
- `broadcast update_reactors` (Beetle decides appearance)

#### Narrator Scroll (View)

**V1. `when I receive render_text` — Paint scene**
- Look up `scene_text[current_scene]`, `scene_choice_a[current_scene]`, `scene_choice_b[current_scene]`.
- Look up `scene_flag_required[current_scene]`. If it is nonzero AND `is_flag_set(flag_id)` is true AND `scene_choice_c[current_scene]` is non-empty, also render choice C.
- Use `say` for main text; costume-swap or stamped text for choice labels.

**V2. `when this sprite clicked` — Click routing**
- Guard: if `ready = 0`, ignore.
- Determine which choice zone the click landed in (simple coordinate comparisons).
- Broadcast `choose_a` / `choose_b` / `choose_c` as appropriate.

#### Worker Ant (Controller)

**C1. `when [1/2/3] key pressed` — Keyboard input**
- Maps key 1 → `choose_a`, key 2 → `choose_b`, key 3 → `choose_c`.
- Same `ready` guard applies.

#### Beetle (Reactor)

**R1. `when I receive update_reactors` — Show/hide**
- If `current_scene = 4`: show, play a small menacing animation (costume swap + wiggle).
- Otherwise: hide.
- No writes to any variable. Pure function of `current_scene`.

### Custom blocks (5 — four with inputs, one no-input helper)

The assignment requires ≥1 custom block with ≥1 input. Four of the five qualify; CB5 is a parameter-less helper that earns its place by isolating flag-to-ending mapping from `run_scene_side_effects`.

**CB1. `add_scene (text) (a_label) (b_label) (next_a) (next_b) (flag_id) (c_label) (next_c)` — 8 inputs**
Appends one row atomically across all 8 scene lists. The only writer to the scene table. Without it, initialize would be 112 near-identical `add X to [list]` blocks; with it, initialize is 14 readable declarative lines.

**CB2. `apply_choice (letter)` — 1 input**
The heart of the interpreter. Locked contract (do not reorder):

```
1. Set local [next_id] = item(current_scene) of scene_next_{letter}   [capture before mutating]
2. If next_id = 0: stop this script              [E2 guard — no such choice]
3. If ending_code > 0: stop this script          [E3 guard — story over]
4. Set current_scene = next_id
5. Call run_scene_side_effects(current_scene)    [operates on the scene JUST ENTERED]
6. Broadcast render_scene
```

The capture-before-mutate pattern ensures scene-entry side effects observe the scene being entered, not the one being left.

**CB3. `is_flag_set (flag_id) :: reports boolean` — 1 input**
```
if flag_id = 1: report (food_carried = 1)
if flag_id = 2: report (scout_trail_known = 1)
if flag_id = 3: report (has_orders = 1)
otherwise:       report (false)
```
Used by V1 to decide whether choice C should render.

**CB4. `run_scene_side_effects (scene_id)` — 1 input**
```
if scene_id = 2:  set has_orders = 1
if scene_id = 6:  set scout_trail_known = 1
if scene_id = 7:  set food_carried = 1
if scene_id = 9b: call resolve_ending          [transition scene — pick ending from flags]
if scene_id = 9c: call resolve_ending          [transition scene — pick ending from flags]
if scene_id = 10: set ending_code = 1
if scene_id = 11: set ending_code = 2
if scene_id = 12: set ending_code = 3
if scene_id = 13: set ending_code = 4
```
Isolated so `apply_choice` stays pure flow control. (Note: "9b" and "9c" are scene ids in the implementation — they will be stored as integers 9 and 10 in practice, with scenes 10-13 shifted to 11-14. Final numeric assignment happens during implementation; the spec uses labels 9/9b/9c/10-13 for readability.)

**CB5. `resolve_ending` — no input**
Computes the correct ending scene id from the current flag state and sets `current_scene` to it. Called by `run_scene_side_effects` on entry to 9b or 9c.

```
if has_orders = 1:
  set current_scene = 13                [full glory — reported in]
else if scout_trail_known = 1 and food_carried = 1:
  set current_scene = 12                [hero shortcut]
else if food_carried = 1:
  set current_scene = 10                [triumph — food, no scout/orders]
else:
  set current_scene = 11                [empty-handed — no food]
```

This is the only place in the engine where multiple flags are combined into a single routing decision. Keeping it in its own block keeps `run_scene_side_effects` a flat ladder and keeps the combination logic auditable in one spot.

**Execution flow on entry to 9b or 9c (this is the subtle part).**

1. `apply_choice("c")` runs with `current_scene = 9`. Per contract: captures `next_id = 9b`, updates `current_scene = 9b`, calls `run_scene_side_effects(9b)`.
2. `run_scene_side_effects(9b)` calls `resolve_ending`.
3. `resolve_ending` reads the flags, sets `current_scene` to the chosen ending id (e.g. 13), and sets `ending_code` to the matching code.
4. `resolve_ending` returns. `run_scene_side_effects` returns. `apply_choice` broadcasts `render_scene` once.
5. The Narrator renders the ending scene's text. `ending_code > 0`, so future input is dropped by the E3 guard in `apply_choice`.

**Why `resolve_ending` sets `ending_code` itself:** `run_scene_side_effects` is only called once per `apply_choice` invocation — it does not re-enter for the new `current_scene`. If `resolve_ending` only set `current_scene`, `ending_code` would remain 0 and the E3 guard wouldn't fire, leaving the game accepting input after reaching an ending. Setting both in the same block closes the hole.

The ladder entries in `run_scene_side_effects` for scenes 10-13 are therefore dead code on the normal path. They're retained defensively — if a future change ever routes directly to 10-13 (e.g., a debug shortcut), the ending_code still gets set. Value: ~5 lines of defensive code in exchange for one fewer landmine.

Final `resolve_ending`:

```
if has_orders = 1:
  set current_scene = 13; set ending_code = 4
else if scout_trail_known = 1 and food_carried = 1:
  set current_scene = 12; set ending_code = 3
else if food_carried = 1:
  set current_scene = 10; set ending_code = 1
else:
  set current_scene = 11; set ending_code = 2
```


---

## 5. Error Handling and Edge Cases

Realistic failures only. Over-engineering error handling is a larger learning anti-pattern than missing a few rare cases.

### Handled

**E1. Player interacts before initialize completes.**
`ready` variable starts at 0. Input handlers (V2, C1) check `ready` and ignore input until 1.

**E2. Player picks a choice the current scene doesn't offer (e.g., key 3 when no choice C).**
`apply_choice` reads `scene_next_<letter>` → returns 0 → guard stops the script. The input is a no-op.

**E3. Player keeps pressing keys after an ending.**
`apply_choice`'s `ending_code > 0` guard drops further inputs. Replay is `green flag`.

### Not handled (deliberate)

- **Scene-table corruption during runtime.** `add_scene` is the only writer; runtime cannot corrupt the table.
- **Index out of range in `next_*`.** Authoring error, not runtime error. Surfaces immediately on a playthrough.
- **Save/load, accessibility, i18n.** Out of scope for this assignment.

### Invariant summary

- `length(scene_*) is equal across all 8 lists` — protected by `add_scene` being the sole writer.
- `current_scene ∈ [1, 14]` while `ending_code = 0`.
- `ending_code ∈ {0,1,2,3,4}`.
- Flags are always 0 or 1.

---

## 6. Testing and Verification

Scratch has no unit tests; verification is a structured manual playthrough matrix.

### Path coverage

One playthrough per ending. Document the exact key sequence for each.

| Ending | Sequence (player choices) | Flags at end | Notes |
|---|---|---|---|
| 10 (triumph) | 1→5→3→4→7→9(sneak)→9c→10 | food=1 | Orders=0, scout=0 — `resolve_ending` picks 10 |
| 11 (empty-handed) | 1→5→3→4→8→9(sneak)→9c→11 | none | Food=0 — `resolve_ending` picks 11 |
| 12 (hero shortcut) | 1→5→3→6→7→9(sneak)→9c→12 | food=1, scout=1 | Orders=0, both scout and food — `resolve_ending` picks 12 |
| 13 (full glory) | 1→2→3→6→7→9(report C)→9b→13 | food=1, scout=1, orders=1 | Orders=1 — `resolve_ending` picks 13 |

Note: at scene 9 the player picks choice A, B, or C. The transition through 9b/9c is automatic (one "..." display frame, no input), and the ending scene is entered via `resolve_ending`.

### Flag gating

| Check | Expected |
|---|---|
| At scene 9, choice C ("Report to the queen first") appears | Only if `has_orders = 1` |
| At scene 9 with `has_orders = 0` | Only choices A and B visible |
| At scene 9 with `has_orders = 1` | All three choices visible |

### Input robustness

| Probe | Expected |
|---|---|
| Hammer keys during init | No advance; `ready` gate holds |
| Press `3` when no choice C | No advance; E2 guard |
| Press keys after reaching ending 10/11/12/13 | No advance; E3 guard |
| Click outside choice zones | No broadcast fired |

### Visual

| Check | Expected |
|---|---|
| Narrator repaints cleanly each scene | No overlap or ghosting |
| Beetle appears only at scene 4 | Hidden everywhere else |
| No flicker from broadcast ordering | `render_text and wait` holds the order |

---

## 7. Repository Layout and Deliverables

```
starting-from-scratch/
├── README.md                                      # project overview + play link
├── .gitignore
│
├── docs/superpowers/
│   ├── specs/2026-04-22-ant-adventure-design.md   # this file
│   ├── plans/2026-04-22-ant-adventure-plan.md     # implementation plan (next)
│   └── decisions/
│       ├── 0001-data-driven-engine.md
│       ├── 0002-stage-as-model.md
│       ├── 0003-numeric-flag-ids.md
│       ├── 0004-scene-9-split.md
│       └── 0005-sb3-as-artifact.md
│
├── project/
│   ├── ant-adventure.sb3                          # the Scratch project (binary)
│   └── assets/                                    # source art (SVG/PNG)
│
└── transcripts/
    ├── README.md                                  # how to read a transcript
    ├── stage.md                                   # Model scripts + vars/lists
    ├── narrator-scroll.md                         # View scripts
    ├── worker-ant.md                              # Controller scripts
    └── beetle.md                                  # Reactor scripts
```

### The `.sb3` problem

`.sb3` is a binary zip. GitHub cannot diff or review it. The **transcripts** are the source of truth for review — every script appears there with a purpose paragraph, contract, and commented block listing. The `.sb3` is committed for reproducibility only. See `decisions/0005-sb3-as-artifact.md`.

### Transcript format

Every script in every transcript file follows this shape:

```
Script / Custom Block: <name>
Purpose: <one paragraph on what this does and why it exists>
Contract (if applicable): <ordering, pre/postconditions>

Blocks:
  <pseudocode rendering of the actual Scratch blocks>
  // comments match the comments attached inside Scratch itself
```

### Commit discipline — "what / why / when / where"

Git provides *when* (timestamp) and *where* (diff) mechanically. Commit messages carry:
- **What:** subject line, imperative, ≤72 chars.
- **Why:** body — motivation, lesson captured, link to decisions/ entries.
- **Where:** footer — files touched with one-line summary, for quick scanning.
- **When:** explicit date line + section/sprint tag.

Template:

```
<type>: <what changed — imperative, ≤72 chars>

Why:
  - <motivation or problem solved>
  - <link to decisions/000X-*.md if relevant>

Where:
  - <path/to/file>: <one-line summary>
  - <path/to/file>: <one-line summary>

When: <YYYY-MM-DD> (Section: <e.g. "Spec authoring", "Engine wiring">)

(Optional) Verification:
  - <what was run/checked>
```

`<type>` is one of: `spec`, `docs`, `engine`, `content`, `assets`, `transcript`, `chore`.

### Comment policy (inside Scratch + mirrored in transcripts)

- Every custom block: block-comment on `define` hat with purpose + contract.
- Every broadcast site: comment naming the listener(s) and why.
- Every non-obvious conditional: comment explaining *why*, not *what*.
- Every variable and list: comment at initialize explaining purpose + valid values.
- **No comments that describe what the blocks plainly show.** Comments earn their place by explaining intent, invariants, or non-obvious ordering.

---

## 8. Requirements Cross-Check

### CS50 assignment requirements

| Requirement | Met by |
|---|---|
| ≥2 sprites | 3 sprites: Worker Ant, Narrator Scroll, Beetle |
| ≥1 non-cat sprite | All three |
| ≥3 scripts | 9 hat-block scripts (S1, S2×3, S3, V1, V2, C1, R1) |
| ≥1 conditional | `is_flag_set`, scene-C gate in V1, `apply_choice` guards, `run_scene_side_effects` ladder, Beetle show/hide |
| ≥1 loop | Init populates 14 scenes; click-zone routing uses repeat-until |
| ≥1 variable | 6: `current_scene`, `food_carried`, `scout_trail_known`, `has_orders`, `ending_code`, `ready` |
| ≥1 custom block with ≥1 input | 4 qualifying: `add_scene` (8 inputs), `apply_choice` (1), `is_flag_set` (1), `run_scene_side_effects` (1). A 5th helper (`resolve_ending`, no input) isolates flag-to-ending mapping. |
| More complex than lecture demos, less than Oscartime | Data-driven interpreter, 14 scenes, 3 meaningful flags, 4 distinct endings. Confirmed twice by independent review. |
| "A few dozen puzzle pieces" | ~130-160 blocks total |
| Well-designed, leverage abstraction | 4 custom blocks each named after a meaningful operation |
| No overly long scripts | Longest script is initialize (14 × `add_scene` + resets) |

### Author-added requirements

| Requirement | Met by |
|---|---|
| Spec, plans, decisions uploaded to GitHub with code | `docs/superpowers/{specs,plans,decisions}/` + `transcripts/` + `project/*.sb3` |
| Commits state what, why, when, where | Template in section 7 |
| Solid, DRY code | `add_scene` (DRY init), single mutation point (`apply_choice`), one flag resolver, one side-effect ladder |
| Comments are very important | Policy in section 7, mirrored between Scratch and transcripts |

---

## 9. Out of Scope

Explicitly **not** part of this project:

- Save/load persistence.
- Accessibility features beyond the default Scratch runtime.
- Internationalization / translations.
- Timer-based endings (`time_remaining` was considered and cut — decorative variable with no real gate).
- Runtime validation of the scene table (the author writes it; errors surface on first playthrough).

---

## 10. Review Trail

- **2026-04-22, brainstorm section 2 (data model):** Sonnet peer review caught scene-3 fork logic bug (long-safe route still hit the beetle), cosmetic queen-briefing scene (no downstream effect), and flag-name-as-string encoding issue. Fixed: scene 6 now bypasses beetle, `has_orders` flag added as third flag with real effect, `scene_flag_required` uses numeric ids.
- **2026-04-22, brainstorm section 3 (scripts):** Sonnet peer review caught the scene-9 single-flag-slot limitation (could not gate both `scout_trail_known` and `has_orders` on one row), the `pick_ending_by_flags` redundancy with player choices, and the broadcast race between `render_text` and `update_reactors`. Fixed: scene 9 split into 9/9b/9c; `broadcast render_text and wait` used in S3.
- **2026-04-22, spec self-review:** Author caught that (a) the `scene_flag_required` gate only controls choice C, not choices A/B as the earlier draft implied; (b) the `next_*` fields are fixed integers and can't dynamically route based on a flag, so a flag-driven ending resolver is required at 9b/9c; (c) `pick_ending_by_flags` (renamed `resolve_ending`) was wrongly deleted — it's the right shape for "compute the narrative consequence of flags collected earlier." Fixed: scene 9 restructured so only choice C is gated (by `has_orders`); scenes 9b/9c are transition scenes with no player choices; CB5 (`resolve_ending`) reinstated with `ending_code` set in the same block to avoid a second `run_scene_side_effects` pass.
