# Ant Adventure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement "A Worker Ant's Urgent Day" — a data-driven interactive story in Scratch — exactly as specified in `docs/superpowers/specs/2026-04-22-ant-adventure-design.md`, with every script, comment, and test result mirrored in reviewable transcripts under `transcripts/`.

**Architecture:** One Scratch project (`project/ant-adventure.sb3`) with Stage (Model) + three sprites: Worker Ant (Controller), Narrator Scroll (View), Beetle (Reactor). All story state lives on the Stage; sprites never write to Model state. Scenes are 14 rows across 8 parallel Scratch lists, walked by a single interpreter loop (`apply_choice` → `run_scene_side_effects` → `render_scene`).

**Tech Stack:** Scratch 3.0 (browser-hosted block editor at https://scratch.mit.edu). Git for version control. No other toolchain.

---

## Zero Context Briefing (read this first)

**You have not seen our previous conversation.** Everything you need is in the spec and this plan.

**Critical up-front facts:**

1. **Scratch is the MIT block-based visual language at https://scratch.mit.edu.** Projects are built in the browser editor and exported as `.sb3` files (zip of `project.json` + assets). You cannot edit `.sb3` content directly in text form; every scripting change means opening the file in the Scratch editor.
2. **The `.sb3` file is binary; git cannot diff it.** The human-readable **transcripts** in `transcripts/*.md` are the review source of truth. See `docs/superpowers/decisions/0005-sb3-as-artifact.md`. Every code change touches BOTH the `.sb3` AND the matching transcript.
3. **Author's review will check transcripts line-by-line against the spec.** The spec is authoritative; this plan operationalizes it. If you discover a conflict between this plan and the spec, **the spec wins** — stop, report the conflict, and ask before deviating.
4. **Locked numeric scene ids.** The spec uses labels 9/9b/9c/10-13 for readability. This plan locks concrete integers: the transition scenes are **9 → 9b (=10) and 9c (=11)**, and the four endings shift to **12/13/14/15**. See the "Scene ID Assignment" table below. Use these integers everywhere in code and transcripts.
5. **Commit discipline (spec §7).** Every commit body has `Why:`, `Where:`, `When:`, and a `<type>:` subject line. The full commit template is reproduced in Task 0 and must be followed verbatim. Do not skip Co-Authored-By trailers on AI-assisted commits.
6. **Comment policy (spec §7).** Every custom block, broadcast, non-obvious conditional, variable, and list gets a comment inside Scratch (right-click block → "Add Comment") AND a matching `//` comment on the same logical line in the transcript. Comments say **why**, not what.
7. **No scope creep.** If it isn't in the spec, don't build it. The spec's §9 lists out-of-scope items; if in doubt, leave it out and flag it.

### Scene ID Assignment (locked — use these integers everywhere)

The spec's 9b/9c labels become these concrete Scratch list indices:

| Spec label | Scratch list index (integer) | Description |
|---|---|---|
| 1  | 1  | Nest (start) |
| 2  | 2  | Queen's briefing |
| 3  | 3  | Tunnel exit |
| 4  | 4  | Beetle encounter |
| 5  | 5  | Sneak out early |
| 6  | 6  | Scout meets you |
| 7  | 7  | Sugar cube found |
| 8  | 8  | Injured, retreat |
| 9  | 9  | Return trip (player agency point) |
| 9b | 10 | Transition: reporting in |
| 9c | 11 | Transition: sneaking back |
| 10 (ending: triumph) | 12 | Ending 1 |
| 11 (ending: empty-handed) | 13 | Ending 2 |
| 12 (ending: hero shortcut) | 14 | Ending 3 |
| 13 (ending: full glory) | 15 | Ending 4 |

**There are therefore 15 scenes (not 14) in the Scratch list tables.** The spec's "14 scenes" text counts the 9/9b/9c trio as one logical scene. The implementation uses 15 list rows. Update the spec or leave as-is? **Leave the spec as-is** — the "14 scenes" phrasing in the spec is a readability choice, not a bug. This plan carries the reconciliation.

`current_scene ∈ [1, 15]`. `ending_code ∈ {0, 1, 2, 3, 4}` (0=playing; 1-4 match endings 12/13/14/15 in order).

### Tooling Note for the Executor

Scratch has no CLI. You will:
- **Author the `.sb3`** by opening https://scratch.mit.edu in a browser (logged in or unsigned-in sandbox), building the project, and File → Save to your computer to export `ant-adventure.sb3`.
- **Place the exported `.sb3`** at `project/ant-adventure.sb3` in the repo.
- **Write the transcripts by hand** in markdown, following the format in `transcripts/README.md`.
- **"Run tests"** means loading the `.sb3` in Scratch, clicking the green flag, and manually walking the four paths from the spec's §6 test matrix. Record the result (PASS/FAIL per path) in the relevant task's verification step.

If you are an LLM without browser automation, produce the `project.json` content directly (Scratch's `.sb3` is a zip containing `project.json` + assets) and document this in Task 0. The `project.json` format is a stable, well-known schema; you can emit it from a template and zip it programmatically. But **the transcript is what the reviewer reads — it must be impeccable regardless of how the `.sb3` was produced.**

---

## File Structure

```
starting-from-scratch/
├── README.md                                 # [exists] overview + play link — update with sb3 link in final task
├── .gitignore                                # [exists] OS cruft
│
├── docs/superpowers/
│   ├── specs/2026-04-22-ant-adventure-design.md    # [exists] spec — do not modify
│   ├── plans/2026-04-22-ant-adventure-plan.md      # [this file] — do not modify during execution
│   └── decisions/                                  # [exists, 5 ADRs] — do not modify
│
├── project/
│   ├── ant-adventure.sb3                     # [create] the Scratch project (binary)
│   └── assets/                               # [create] source art
│       ├── worker-ant.svg                    # non-cat protagonist sprite
│       ├── narrator-scroll.svg               # view sprite (looks like a scroll)
│       ├── beetle.svg                        # reactor sprite
│       └── backdrop-nest.svg                 # nest interior backdrop
│
└── transcripts/
    ├── README.md                             # [exists] transcript format + file map
    ├── stage.md                              # [create] Model scripts + vars/lists + custom blocks
    ├── narrator-scroll.md                    # [create] View scripts
    ├── worker-ant.md                         # [create] Controller scripts
    └── beetle.md                             # [create] Reactor scripts
```

Each transcript file has ONE clear responsibility (one sprite or the Stage). Scripts that share state live in the same file. This keeps transcripts reviewable in isolation.

---

## Task Ordering Rationale

Tasks are ordered so every commit is **independently reviewable and produces a valid partial system**:

1. **Task 0** — Bootstrap: `.gitignore` sanity, commit template recap. Small safety net.
2. **Task 1** — Assets: three sprite SVGs + one backdrop. Pure art; no logic.
3. **Task 2** — Scratch project skeleton: Stage variables, empty lists, three sprites placed. Green flag does nothing yet. First `.sb3` commit.
4. **Task 3** — CB1 (`add_scene`) and S1 (initialize): scene table populates on green flag. Can inspect lists after clicking green flag to verify all 15 rows present.
5. **Task 4** — V1 (`render_text`) + S3 (render dispatcher): the Narrator can display scene 1's text after init. No player input yet.
6. **Task 5** — CB2 (`apply_choice`), CB4 (`run_scene_side_effects`), S2 (three `choose_*` hats), C1 (keyboard): player can advance the story via keys. Enough to walk the whole game without mouse. Path-coverage tests for endings 1 and 2 (simplest flag combinations) become runnable.
7. **Task 6** — CB3 (`is_flag_set`) integrated into V1: choice C renders conditionally. Path-coverage tests for endings 3 and 4 become runnable.
8. **Task 7** — CB5 (`resolve_ending`) integrated into CB4: the 9b/9c transitions route to correct endings. All four paths pass.
9. **Task 8** — V2 (Narrator click routing): mouse input works in addition to keyboard.
10. **Task 9** — R1 (Beetle show/hide): visual reactor complete.
11. **Task 10** — Comment sweep: every required comment site (per spec §7) is verified in both Scratch and the transcripts. Final polish.
12. **Task 11** — README update with Scratch link + final commit.

Each task ends with a commit. Reviewer can `git log --oneline` and see a clean progression.

---

## Task 0: Bootstrap and Executor Checklist

**Files:**
- Read: `docs/superpowers/specs/2026-04-22-ant-adventure-design.md`
- Read: `docs/superpowers/decisions/0001-data-driven-engine.md`
- Read: `docs/superpowers/decisions/0002-stage-as-model.md`
- Read: `docs/superpowers/decisions/0003-numeric-flag-ids.md`
- Read: `docs/superpowers/decisions/0004-scene-9-split.md`
- Read: `docs/superpowers/decisions/0005-sb3-as-artifact.md`
- Read: `transcripts/README.md`

- [ ] **Step 1: Read the spec end-to-end.**

Spend ten minutes on `docs/superpowers/specs/2026-04-22-ant-adventure-design.md`. This plan does not duplicate the spec; it operationalizes it. If you skip the read, you will not understand why certain choices in this plan are the way they are.

- [ ] **Step 2: Read all five ADRs.**

They are short (~1 page each). They carry the *why* behind design choices. The transcripts will reference them by filename (e.g., "See ADR-0003 for rationale on numeric flag ids"); you need to know what's there.

- [ ] **Step 3: Confirm the scene id assignment.**

Re-read the "Scene ID Assignment" table at the top of this plan. Every list index and every `scene_next_*` value in the initialize script uses the integers in the rightmost column. There is no 9b/9c literal anywhere in the Scratch project — only 10 and 11.

- [ ] **Step 4: Confirm the commit template.**

Every commit in this plan uses the template below. Reproduce it exactly; do not invent shorter forms.

```
<type>: <what changed — imperative, ≤72 chars>

Why:
  - <motivation or problem solved>
  - <link to decisions/000X-*.md if relevant>

Where:
  - <path/to/file>: <one-line summary>
  - <path/to/file>: <one-line summary>

When: 2026-04-22 (Section: <e.g. "Task 1 assets", "Task 3 scene init">)

Verification:
  - <what was run/checked and what the result was>

Co-Authored-By: <executor model> <email>
```

`<type>` is one of: `spec`, `docs`, `engine`, `content`, `assets`, `transcript`, `chore`. Use `engine` for any change that modifies script/custom-block logic in the `.sb3`. Use `content` for changes that only touch scene table data. Use `transcript` for transcript-only updates. Use `assets` for the SVG/PNG sprite files.

- [ ] **Step 5: Confirm the `.sb3` authoring path.**

Decide how you will produce the `.sb3`:
- **Path A (browser):** open https://scratch.mit.edu, author visually, File → Save to your computer. Best if you can drive a browser.
- **Path B (direct `project.json`):** emit `project.json` + assets and zip them with `.sb3` extension. Requires detailed knowledge of Scratch's `project.json` schema. Acceptable if done carefully.

Whichever you choose, **the transcript is the reviewer's source of truth.** The `.sb3` must faithfully implement what the transcript claims, but the reviewer will not execute the `.sb3` line-by-line — they will read the transcript and spot-check by loading the `.sb3` in Scratch and walking the four test paths.

- [ ] **Step 6: No commit needed for Task 0.**

This task is a briefing, not a change. Proceed to Task 1.

**Done when:** You can answer these three questions without referring back to the spec: (1) What does `apply_choice` do and in what order? (2) Which single flag gates choice C at scene 9? (3) Why are scenes 10 and 11 (in Scratch list terms) transition scenes rather than decision scenes?

Answers (for self-check):
1. Captures `next_id` from `scene_next_<letter>` at the current scene, guards against `next_id=0` and `ending_code>0`, updates `current_scene`, calls `run_scene_side_effects(current_scene)`, broadcasts `render_scene`.
2. `has_orders` (flag id 3).
3. Because `scene_next_*` values are fixed integers and cannot express "route to ending 12 if food=1, 13 if food=0" — flag-driven routing requires the `resolve_ending` helper, which runs at transition-scene entry.

---

## Task 1: Sprite and Backdrop Assets

**Files:**
- Create: `project/assets/worker-ant.svg`
- Create: `project/assets/narrator-scroll.svg`
- Create: `project/assets/beetle.svg`
- Create: `project/assets/backdrop-nest.svg`

Simple, readable SVGs. Each sprite is one costume for now (second costume for Beetle's "menacing wiggle" is added in Task 9). The assets ship with the repo so the project can be reauthored from scratch if the `.sb3` is lost.

- [ ] **Step 1: Create the Worker Ant SVG.**

A small ant illustration, side view, antennae visible. ~80×60 pixels. Black/dark-brown body. No cat features (the assignment requires at least one non-cat sprite; ours is non-cat throughout).

Exact SVG — place at `project/assets/worker-ant.svg`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 80 60" width="80" height="60">
  <!-- Worker ant: three body segments (head, thorax, abdomen), six legs, two antennae -->
  <!-- Head -->
  <ellipse cx="18" cy="30" rx="10" ry="8" fill="#2b1d10"/>
  <!-- Eye -->
  <circle cx="14" cy="28" r="1.5" fill="#fff"/>
  <!-- Antennae -->
  <path d="M14 24 Q10 18 6 14" stroke="#2b1d10" stroke-width="1.5" fill="none"/>
  <path d="M18 22 Q16 16 14 10" stroke="#2b1d10" stroke-width="1.5" fill="none"/>
  <!-- Thorax -->
  <ellipse cx="38" cy="32" rx="10" ry="7" fill="#3a2815"/>
  <!-- Abdomen -->
  <ellipse cx="60" cy="33" rx="14" ry="10" fill="#2b1d10"/>
  <!-- Legs (three per side, middle ones angled forward and back) -->
  <path d="M32 36 L28 48" stroke="#2b1d10" stroke-width="1.5"/>
  <path d="M38 38 L38 50" stroke="#2b1d10" stroke-width="1.5"/>
  <path d="M44 36 L48 48" stroke="#2b1d10" stroke-width="1.5"/>
  <path d="M32 28 L28 20" stroke="#2b1d10" stroke-width="1.5"/>
  <path d="M38 26 L38 16" stroke="#2b1d10" stroke-width="1.5"/>
  <path d="M44 28 L48 20" stroke="#2b1d10" stroke-width="1.5"/>
</svg>
```

- [ ] **Step 2: Create the Narrator Scroll SVG.**

A stylized open scroll. ~260×160 pixels to accommodate text painted on top of it at runtime. Parchment color (#e8dcb0) with darker brown rolled ends.

Exact SVG — place at `project/assets/narrator-scroll.svg`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 260 160" width="260" height="160">
  <!-- Narrator Scroll: parchment panel with rolled ends. Text is drawn on top
       by Scratch's "say" block at runtime; this is the visual frame. -->
  <!-- Left rolled end -->
  <rect x="0" y="20" width="20" height="120" rx="10" fill="#8b5a2b"/>
  <!-- Right rolled end -->
  <rect x="240" y="20" width="20" height="120" rx="10" fill="#8b5a2b"/>
  <!-- Parchment panel -->
  <rect x="20" y="30" width="220" height="100" fill="#e8dcb0" stroke="#8b5a2b" stroke-width="1.5"/>
  <!-- Subtle inner border -->
  <rect x="28" y="38" width="204" height="84" fill="none" stroke="#c9b58c" stroke-width="0.8" stroke-dasharray="3 2"/>
</svg>
```

- [ ] **Step 3: Create the Beetle SVG.**

A menacing beetle viewed from above. ~90×70 pixels. Dark carapace (#1a1a1a) with a segmented midline and mandibles at the front.

Exact SVG — place at `project/assets/beetle.svg`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 90 70" width="90" height="70">
  <!-- Beetle: oval carapace, mandibles front, six legs -->
  <!-- Carapace -->
  <ellipse cx="45" cy="35" rx="30" ry="22" fill="#1a1a1a"/>
  <!-- Segment line down the middle -->
  <line x1="45" y1="15" x2="45" y2="55" stroke="#444" stroke-width="1"/>
  <!-- Pronotum (shield behind the head) -->
  <ellipse cx="45" cy="18" rx="12" ry="6" fill="#2a2a2a"/>
  <!-- Mandibles -->
  <path d="M39 10 L34 4" stroke="#1a1a1a" stroke-width="2"/>
  <path d="M51 10 L56 4" stroke="#1a1a1a" stroke-width="2"/>
  <!-- Six legs -->
  <path d="M18 28 L4 22" stroke="#1a1a1a" stroke-width="2"/>
  <path d="M15 38 L0 40" stroke="#1a1a1a" stroke-width="2"/>
  <path d="M18 48 L4 54" stroke="#1a1a1a" stroke-width="2"/>
  <path d="M72 28 L86 22" stroke="#1a1a1a" stroke-width="2"/>
  <path d="M75 38 L90 40" stroke="#1a1a1a" stroke-width="2"/>
  <path d="M72 48 L86 54" stroke="#1a1a1a" stroke-width="2"/>
</svg>
```

- [ ] **Step 4: Create the Nest backdrop SVG.**

A simple earthy backdrop suggesting an ant nest interior. Scratch Stage is 480×360.

Exact SVG — place at `project/assets/backdrop-nest.svg`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 480 360" width="480" height="360">
  <!-- Nest backdrop: gradient from dark-brown (deep nest) to lighter earth near top -->
  <defs>
    <linearGradient id="earthGrad" x1="0" y1="0" x2="0" y2="1">
      <stop offset="0%" stop-color="#5c3a1e"/>
      <stop offset="100%" stop-color="#2b1a0c"/>
    </linearGradient>
  </defs>
  <rect width="480" height="360" fill="url(#earthGrad)"/>
  <!-- A few tunnel silhouettes to suggest depth -->
  <ellipse cx="80" cy="80" rx="40" ry="20" fill="#1a0f06" opacity="0.6"/>
  <ellipse cx="400" cy="120" rx="50" ry="25" fill="#1a0f06" opacity="0.6"/>
  <ellipse cx="240" cy="260" rx="80" ry="35" fill="#1a0f06" opacity="0.5"/>
</svg>
```

- [ ] **Step 5: Verify the files.**

Run: `ls -la project/assets/`
Expected: four SVG files, all non-empty.

Run: `file project/assets/*.svg`
Expected: all four identified as SVG (contains "XML" or "SVG" in output).

- [ ] **Step 6: Commit.**

```bash
git add project/assets/
git commit -m "$(cat <<'EOF'
assets: add sprite and backdrop SVGs for Ant Adventure

Why:
  - Source art ships with the repo so the Scratch project is
    reauthorable if the .sb3 is lost (per ADR-0005, the .sb3 is an
    artifact; sources are the truth).
  - Four files cover the three sprites (Worker Ant, Narrator Scroll,
    Beetle) plus the nest backdrop. All non-cat — satisfies the
    assignment requirement.

Where:
  - project/assets/worker-ant.svg: protagonist sprite, side view
  - project/assets/narrator-scroll.svg: parchment panel used as the
    View rendering surface
  - project/assets/beetle.svg: obstacle/reactor sprite at scene 4
  - project/assets/backdrop-nest.svg: Stage backdrop, 480x360

When: 2026-04-22 (Section: Task 1 assets)

Verification:
  - ls project/assets/ shows four .svg files
  - file project/assets/*.svg confirms SVG format
EOF
)"
```

**Done when:**
- `project/assets/` contains the four SVGs exactly as specified.
- Commit is on `main` with the template above.
- No other files have been modified.

---

## Task 2: Scratch Project Skeleton

**Files:**
- Create: `project/ant-adventure.sb3` (binary)
- Create: `transcripts/stage.md`
- Create: `transcripts/worker-ant.md`
- Create: `transcripts/narrator-scroll.md`
- Create: `transcripts/beetle.md`

Build the project's bones: Stage with all six variables and all eight lists (empty), three sprites (using the SVGs from Task 1) placed but scriptless, the nest backdrop. Click green flag → nothing happens yet. This is the commit where "there is a Scratch project."

- [ ] **Step 1: Create the Scratch project.**

Open https://scratch.mit.edu → Create. Delete the default cat sprite.

Upload the four assets from Task 1:
- Sprites: Worker Ant, Narrator Scroll, Beetle (File → Upload Sprite).
- Backdrop: Nest (Stage → Backdrops tab → Upload Backdrop).

Position sprites (via the sprite pane or drag):
- **Narrator Scroll** at the center-upper area, say x=0, y=60. This is the main text panel.
- **Worker Ant** at the lower-left, x=-160, y=-120. This is the protagonist visible during play.
- **Beetle** offstage (hidden via "hide" block will come in Task 9), for now set x=160, y=-120 so it's placed but visible during skeleton testing.

- [ ] **Step 2: Create Stage variables.**

In the Stage's Variables panel, create all six as **For all sprites** (global):
- `current_scene` (default value: 0)
- `food_carried` (default value: 0)
- `scout_trail_known` (default value: 0)
- `has_orders` (default value: 0)
- `ending_code` (default value: 0)
- `ready` (default value: 0)

Attach a comment on each variable in the Stage's Variables panel explaining purpose. Right-click each variable's reporter block (drag one to the Stage scripts area temporarily, comment it, then delete the reporter — the comment sticks). Exact comment texts (these will appear in `transcripts/stage.md`):

- `current_scene`: "Program counter into the scene table. Range [1, 15] while ending_code=0. Written only by CB2 apply_choice and CB5 resolve_ending."
- `food_carried`: "Flag: did the ant retrieve the sugar? Set at scene 7 by CB4. Read by CB3 is_flag_set (id=1) and CB5 resolve_ending."
- `scout_trail_known`: "Flag: did the ant meet the scout? Set at scene 6 by CB4. Read by CB3 (id=2) and CB5."
- `has_orders`: "Flag: did the ant hear the queen's briefing? Set at scene 2 by CB4. Read by CB3 (id=3) and CB5. Gates choice C at scene 9."
- `ending_code`: "0=playing, 1/2/3/4 at endings 12/13/14/15 respectively. Set by CB5 resolve_ending (or defensively by CB4 if reached via a future direct route). Read by CB2 as the E3 guard."
- `ready`: "Init guard. 0 during S1 initialize, 1 after first render dispatched. Input handlers (V2, C1) ignore input while ready=0."

- [ ] **Step 3: Create Stage lists.**

In the Stage's Variables panel, create all eight lists as **For all sprites** (global), all empty:
- `scene_text`
- `scene_choice_a`
- `scene_choice_b`
- `scene_next_a`
- `scene_next_b`
- `scene_flag_required`
- `scene_choice_c`
- `scene_next_c`

Attach comments (same technique as Step 2). Exact texts:

- `scene_text`: "Prose shown to the player at each scene. Row N = scene N."
- `scene_choice_a`: "Label for choice A. Always non-empty (choice A is always present in scenes that have any choices)."
- `scene_choice_b`: "Label for choice B. Always non-empty in normal decision scenes; empty strings are not expected in the current scene table."
- `scene_next_a`: "Scene id to jump to if A picked. 0 means 'no such choice' (guards E2)."
- `scene_next_b`: "Scene id to jump to if B picked. 0 means 'no such choice'."
- `scene_flag_required`: "Numeric flag id (0=none, 1=food, 2=scout, 3=orders) that gates choice C. See ADR-0003."
- `scene_choice_c`: "Label for conditional choice C. Empty string if no C for this scene."
- `scene_next_c`: "Scene id to jump to if C picked. 0 means 'no C on this scene'."

- [ ] **Step 4: Hide all variable watchers except the debug-useful ones.**

Uncheck the watcher boxes for `ready` (never interesting to a player) and all eight lists (too cluttered on stage). Keep `current_scene`, `food_carried`, `scout_trail_known`, `has_orders`, and `ending_code` visible during development — they are valuable for debugging and can be hidden before final export if desired. Document this decision in the transcript (Step 6).

- [ ] **Step 5: Save the `.sb3`.**

File → Save to your computer → name it `ant-adventure.sb3` → place at `project/ant-adventure.sb3`.

- [ ] **Step 6: Write the initial transcript files.**

Create four markdown files under `transcripts/` — each with the skeleton for its owner's content. For Task 2, only `stage.md` has substantive content (the variables + lists); the other three files describe the sprite's role and note that its scripts will be filled in by later tasks.

**`transcripts/stage.md` contents (exactly):**

```markdown
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
```

**`transcripts/worker-ant.md` contents (exactly):**

```markdown
# Worker Ant — Controller

Role: the Controller in the MVC split. Listens for player keyboard input and emits `choose_a/b/c` broadcasts. Does not apply the choice itself; that's the Stage's job (CB2 `apply_choice`).

The Worker Ant sprite is the protagonist visible during play but holds no story state. It is deliberately minimal.

Costume: `project/assets/worker-ant.svg`.

---

## Scripts

| Section | Name | Status |
|---|---|---|
| C1 | `when [1] key pressed` → broadcast `choose_a` | **Task 5** |
| C1 | `when [2] key pressed` → broadcast `choose_b` | **Task 5** |
| C1 | `when [3] key pressed` → broadcast `choose_c` | **Task 5** |

All three include a `ready = 1` guard.
```

**`transcripts/narrator-scroll.md` contents (exactly):**

```markdown
# Narrator Scroll — View

Role: the View in the MVC split. Reads `current_scene` and the scene-table lists on the Stage and renders the current scene's text + choice labels. Also routes clicks on the scroll into `choose_a/b/c` broadcasts (shared downstream with the Worker Ant's keyboard input).

Costume: `project/assets/narrator-scroll.svg`.

---

## Scripts

| Section | Name | Status |
|---|---|---|
| V1 | `when I receive render_text` — paint scene | **Task 4** (choice C gate in Task 6) |
| V2 | `when this sprite clicked` — click routing | **Task 8** |
```

**`transcripts/beetle.md` contents (exactly):**

```markdown
# Beetle — Reactor

Role: a reactor in the MVC split. Reads `current_scene` and shows/hides itself. Never writes to story state.

Costume: `project/assets/beetle.svg` (second costume for menacing-wiggle animation added in Task 9).

---

## Scripts

| Section | Name | Status |
|---|---|---|
| R1 | `when I receive update_reactors` — show/hide | **Task 9** |
```

- [ ] **Step 7: Verify.**

Run: `ls -la project/ transcripts/`
Expected:
- `project/ant-adventure.sb3` exists, non-empty (should be a few KB — a minimal Scratch project with three sprites and a backdrop).
- `transcripts/stage.md`, `transcripts/worker-ant.md`, `transcripts/narrator-scroll.md`, `transcripts/beetle.md` all exist and are non-empty.

Open `project/ant-adventure.sb3` in Scratch. Confirm:
- Three sprites present: Worker Ant, Narrator Scroll, Beetle.
- Backdrop: nest (the earthy gradient).
- Stage has all 6 variables and all 8 lists (visible in the Variables panel on the Stage; watchers for 5 vars on stage, rest hidden).
- Clicking the green flag does nothing (no scripts yet).

- [ ] **Step 8: Commit.**

```bash
git add project/ant-adventure.sb3 transcripts/*.md
git commit -m "$(cat <<'EOF'
engine: scaffold Scratch project skeleton with variables and lists

Why:
  - Establish the Scratch project's bones: Stage variables (6), lists (8),
    three sprites placed with their Task 1 SVGs, backdrop set. This is
    the foundation every subsequent task modifies.
  - Transcripts scaffolded now so each later task plugs script content
    into a known location rather than creating new files. Keeps reviews
    focused on one section per commit.
  - Every variable and list carries a Scratch block-comment with purpose
    and valid-value range, mirrored in transcripts/stage.md (per spec §7
    comment policy).

Where:
  - project/ant-adventure.sb3: three sprites, nest backdrop, 6 vars + 8
    empty lists on Stage, no scripts
  - transcripts/stage.md: full variable + list tables with purposes;
    script sections marked with their owning task
  - transcripts/worker-ant.md: role description, C1 script stubs
  - transcripts/narrator-scroll.md: role description, V1 & V2 stubs
  - transcripts/beetle.md: role description, R1 stub

When: 2026-04-22 (Section: Task 2 skeleton)

Verification:
  - .sb3 opens in Scratch with expected sprites + backdrop + variables
  - Green flag produces no behavior (correct — no scripts yet)
  - Variable watchers for 5 runtime vars visible; `ready` and all lists hidden
EOF
)"
```

**Done when:**
- `.sb3` loads in Scratch with three sprites + nest backdrop + all 6 variables + all 8 empty lists.
- Green flag does nothing.
- `transcripts/*.md` all exist and contain the content above verbatim.
- Commit is on `main`.

---

## Task 3: CB1 `add_scene` and S1 Initialize

**Files:**
- Modify: `project/ant-adventure.sb3` — add CB1 and S1 on Stage
- Modify: `transcripts/stage.md` — fill in S1 and CB1 sections

After this task, clicking green flag populates all 8 scene lists with 15 rows each. Still no player input; still no rendering.

- [ ] **Step 1: Define CB1 `add_scene` on the Stage.**

In the Stage's scripts panel, **Variables → My Blocks → Make a Block**. Name: `add_scene`. Add inputs in this order (use "Add an input → number or text" for each):

1. `text` (text)
2. `a` (text)
3. `b` (text)
4. `na` (number) — "next_a"
5. `nb` (number) — "next_b"
6. `fid` (number) — "flag_id"
7. `c` (text)
8. `nc` (number) — "next_c"

Leave "Run without screen refresh" UNCHECKED (the block performs ordinary list appends; there's no need to suppress refresh).

Attach a block-comment on the `define add_scene ...` hat. Exact comment text:

```
CB1: add_scene — the only writer to the scene table.
Appends one row atomically across all 8 parallel lists. Keeps the
parallel-list invariant (all lists same length) mechanically safe.
Called 15 times from S1 during initialize. Never call add_scene
outside S1; never use "add X to [scene_*]" blocks directly elsewhere.
See ADR-0001 for rationale.
```

Block body (drag blocks to build this):

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

No inner comments — the block body is self-evident given the purpose comment on the hat.

- [ ] **Step 2: Build S1 `when green flag clicked`.**

Attach a block-comment on the hat. Exact text:

```
S1: Initialize. Resets all runtime state and populates the scene table.
Sequence: set ready=0, reset vars, delete list contents, call
add_scene() 15 times in scene-id order, set current_scene=1, broadcast
render_scene and wait, set ready=1.
Why ready is set AFTER the first render: the guard in V2/C1 checks
ready, and we want the initial paint to complete before accepting input.
```

Block body:

```
when green flag clicked
  set [ready v] to (0)
  // reset all runtime state to start values
  set [current_scene v] to (1)
  set [food_carried v] to (0)
  set [scout_trail_known v] to (0)
  set [has_orders v] to (0)
  set [ending_code v] to (0)

  // clear the scene table before rebuilding it
  delete all of [scene_text v]
  delete all of [scene_choice_a v]
  delete all of [scene_choice_b v]
  delete all of [scene_next_a v]
  delete all of [scene_next_b v]
  delete all of [scene_flag_required v]
  delete all of [scene_choice_c v]
  delete all of [scene_next_c v]

  // populate the scene table (15 rows) — row order is scene id order
  add_scene [You wake in the nest. The queen's antennae tap urgently — an important briefing.] [Listen to the queen] [Slip out to the tunnel quietly] (2) (5) (0) [] (0)
  add_scene [The queen's antennae brush yours. "The picnic humans left a sugar cube. Storm is coming. Bring it back — and HURRY."] [Onward to the tunnel exit] [] (3) (0) (0) [] (0)
  add_scene [You reach the tunnel exit. Sunlight and grass. Two paths branch before you.] [Short route — risky ground with a beetle] [Long route — quieter, may find a scout] (4) (6) (0) [] (0)
  add_scene [A massive beetle blocks the path, mandibles clicking.] [Dash past for the sugar] [Retreat — too dangerous] (7) (8) (0) [] (0)
  add_scene [You sneak out before anyone notices. The tunnel exit lies ahead.] [Continue to the tunnel exit] [] (3) (0) (0) [] (0)
  add_scene [A fellow scout greets you and shares a shortcut only the patrols know.] [Take the scout's path to the sugar] [] (7) (0) (0) [] (0)
  add_scene [The sugar cube glitters in the grass. You hoist it — heavier than you are, but ants don't quit.] [Home — time is running out] [] (9) (0) (0) [] (0)
  add_scene [Bruised and limping, you turn back empty-handed.] [Home without the sugar] [] (9) (0) (0) [] (0)
  add_scene [The nest is in sight. One last choice: sneak or report?] [Sneak back quietly] [Take the main tunnel] (11) (11) (3) [Report to the queen first] (10)
  add_scene [...] [] [] (0) (0) (0) [] (0)
  add_scene [...] [] [] (0) (0) (0) [] (0)
  add_scene [Triumph! You haul the sugar to the larder. The colony feasts tonight.] [] [] (0) (0) (0) [] (0)
  add_scene [You return empty-handed. The colony will weather the storm on stored grain — this time.] [] [] (0) (0) (0) [] (0)
  add_scene [The scout's shortcut shaved hours off your trip. You arrive with sugar before the storm breaks. Your name is whispered in the tunnels.] [] [] (0) (0) (0) [] (0)
  add_scene [You reported to the queen, delivered the sugar, and the colony is evacuated ahead of the storm. The queen taps her antennae in approval — full glory.] [] [] (0) (0) (0) [] (0)

  set [current_scene v] to (1)
  broadcast [render_scene v] and wait
  set [ready v] to (1)
```

**Key scene-id crosswalk (echo of the table at the top of the plan):**
- Row 9 (scene 9 "Return trip"): `scene_next_a=11`, `scene_next_b=11` (both A and B go to the sneak transition), `scene_flag_required=3` (has_orders), `scene_choice_c="Report to the queen first"`, `scene_next_c=10` (the report transition).
- Row 10 (transition: reporting in): `scene_text="..."`, no choices. `scene_next_*=0` (guards won't matter; CB4 calls CB5 before any input).
- Row 11 (transition: sneaking back): same `"..."`, no choices.
- Rows 12-15 (endings): `scene_text` is the ending prose, no choices.

- [ ] **Step 3: Verify by green-flagging.**

Open the `.sb3` in Scratch. Click the green flag.

Expected:
- `current_scene` shows `1` on the Stage.
- All flag variables show `0`.
- `ending_code` shows `0`.
- Show the list watchers temporarily (check the boxes in the Variables panel) and confirm each of the 8 scene_* lists has exactly 15 items. Pick a couple of spot checks:
  - `scene_text` item 1 starts with "You wake in the nest..."
  - `scene_text` item 9 starts with "The nest is in sight..."
  - `scene_next_c` item 9 = 10 (reporting transition)
  - `scene_flag_required` item 9 = 3 (orders)
  - `scene_text` item 15 starts with "You reported to the queen..."
- Hide the list watchers again before saving.

Record the spot-check result as a note to paste into the transcript in Step 4.

- [ ] **Step 4: Update `transcripts/stage.md`.**

Replace the S1 row and the CB1 row in the "Scripts and Custom Blocks" table to show `Implemented`. Add new sections at the bottom of `stage.md`:

```markdown
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
  add_scene [You wake in the nest...] [Listen to the queen] [Slip out to the tunnel quietly] (2) (5) (0) [] (0)
  add_scene [The queen's antennae brush yours...] [Onward to the tunnel exit] [] (3) (0) (0) [] (0)
  add_scene [You reach the tunnel exit...] [Short route — risky ground with a beetle] [Long route — quieter, may find a scout] (4) (6) (0) [] (0)
  add_scene [A massive beetle blocks the path...] [Dash past for the sugar] [Retreat — too dangerous] (7) (8) (0) [] (0)
  add_scene [You sneak out before anyone notices...] [Continue to the tunnel exit] [] (3) (0) (0) [] (0)
  add_scene [A fellow scout greets you...] [Take the scout's path to the sugar] [] (7) (0) (0) [] (0)
  add_scene [The sugar cube glitters in the grass...] [Home — time is running out] [] (9) (0) (0) [] (0)
  add_scene [Bruised and limping, you turn back empty-handed.] [Home without the sugar] [] (9) (0) (0) [] (0)
  add_scene [The nest is in sight. One last choice...] [Sneak back quietly] [Take the main tunnel] (11) (11) (3) [Report to the queen first] (10)
  add_scene [...] [] [] (0) (0) (0) [] (0)                  // row 10: reporting transition
  add_scene [...] [] [] (0) (0) (0) [] (0)                  // row 11: sneaking transition
  add_scene [Triumph! ...] [] [] (0) (0) (0) [] (0)         // row 12: ending 1 (triumph)
  add_scene [You return empty-handed...] [] [] (0) (0) (0) [] (0)   // row 13: ending 2 (empty-handed)
  add_scene [The scout's shortcut...] [] [] (0) (0) (0) [] (0)      // row 14: ending 3 (hero shortcut)
  add_scene [You reported to the queen...] [] [] (0) (0) (0) [] (0) // row 15: ending 4 (full glory)

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
```

- [ ] **Step 5: Save the `.sb3`.**

File → Save to your computer → overwrite `project/ant-adventure.sb3`.

- [ ] **Step 6: Commit.**

```bash
git add project/ant-adventure.sb3 transcripts/stage.md
git commit -m "$(cat <<'EOF'
engine: add CB1 add_scene and S1 initialize; scene table populates

Why:
  - S1 (when green flag clicked) is the single initializer: resets
    all runtime state and builds the 15-row scene table via 15
    CB1 calls. The "clear + rebuild" pattern keeps the parallel-
    list invariant safe without diffing (ADR-0001).
  - CB1 (add_scene, 8 inputs) is the sole writer to the scene
    table. Centralizing appends is what makes adding/editing
    scenes a data-only operation per the data-driven design.
  - `ready` flag is raised 0→1 only AFTER the first render is
    dispatched (via broadcast-and-wait), blocking premature input
    during init (spec E1).

Where:
  - project/ant-adventure.sb3: CB1 defined on Stage with purpose
    comment; S1 hat added with reset/clear/populate/render/ready
    sequence; 15 add_scene calls in scene-id order
  - transcripts/stage.md: CB1 and S1 sections filled in with full
    block listing, purpose paragraph, contract, and task-3
    verification notes

When: 2026-04-22 (Section: Task 3 scene init)

Verification:
  - Green flag completes; scene lists show length 15 each
  - scene_text[1] starts with "You wake in the nest..."
  - scene_next_c[9] = 10, scene_flag_required[9] = 3
  - scene_text[15] starts with "You reported to the queen..."
  - current_scene = 1, all flags = 0, ending_code = 0, ready = 1
EOF
)"
```

**Done when:**
- Clicking green flag leaves all scene_* lists with length 15.
- The three spot-check list cells (scene_text[1], scene_next_c[9], scene_text[15]) match the expected content.
- `transcripts/stage.md` has the S1 and CB1 sections filled in matching the `.sb3`.
- Commit on `main`.

---

## Task 4: V1 `render_text` (basic) and S3 Render Dispatcher

**Files:**
- Modify: `project/ant-adventure.sb3` — add V1 on Narrator Scroll, S3 on Stage
- Modify: `transcripts/stage.md` — add S3 section
- Modify: `transcripts/narrator-scroll.md` — add V1 section (without the choice-C gate; that's Task 6)

After this task, clicking green flag shows scene 1's text and choice labels A & B on the Narrator Scroll. No input yet. Beetle is still not reacting.

- [ ] **Step 1: Add S3 on Stage.**

Block-comment on the hat (exact text):

```
S3: Render dispatcher. When the interpreter advances, render the
scene (Narrator first, then Reactors). broadcast-and-wait on
render_text ensures the Narrator's paint completes before the
Beetle's show/hide animation kicks in — eliminates the visual race
identified in the Sonnet peer review (spec §10, Review Trail).
```

Block body:

```
when I receive [render_scene v]
  broadcast [render_text v] and wait         // V1 paints first
  broadcast [update_reactors v]              // Beetle/etc react after paint
```

- [ ] **Step 2: Add V1 on Narrator Scroll.**

Switch to the Narrator Scroll sprite. In its Scripts panel, add the V1 hat.

Block-comment on the hat:

```
V1: Paint scene. Reads current_scene and the scene-table lists from
the Stage, renders the prose in "say" and the two (or three) choice
labels. The choice-C gate is added in Task 6; for now only A & B
are rendered. The scroll's costume provides the visual frame.
```

Block body (Task 4 version — NO choice-C gate yet):

```
when I receive [render_text v]
  go to x: (0) y: (60)                       // re-center in case of drift
  show
  // paint the scene's prose
  say (item (current_scene) of [scene_text v]) for (3) seconds

  // render choice labels A and B
  // (choice C rendering is added in Task 6 with the is_flag_set gate)
  if <not <(item (current_scene) of [scene_choice_a v]) = []>> then
    say (join [1) ] (item (current_scene) of [scene_choice_a v])) for (2) seconds
  end
  if <not <(item (current_scene) of [scene_choice_b v]) = []>> then
    say (join [2) ] (item (current_scene) of [scene_choice_b v])) for (2) seconds
  end
```

Rationale for the conditional around A/B: rows 10, 11, 12, 13, 14, 15 have empty strings for `scene_choice_a`/`scene_choice_b` (transitions + endings). Skipping empty labels avoids rendering "1) " with no content.

A note on readability — "say ... for 3 seconds" is the simplest Scratch primitive for text display. It's not the most elegant UI, but this is a CS-fundamentals project, not a UX showcase. The text-bubble rendering is acceptable and keeps the focus on the interpreter logic.

- [ ] **Step 3: Verify.**

Click green flag. Expected sequence:
1. Scroll centers.
2. Speech bubble shows: "You wake in the nest. The queen's antennae tap urgently..."
3. Bubble shows "1) Listen to the queen".
4. Bubble shows "2) Slip out to the tunnel quietly".
5. Nothing else happens (no input yet).

Nothing advances because no input is wired. This is expected.

- [ ] **Step 4: Update transcripts.**

Append to `transcripts/stage.md` (after the CB1 section):

```markdown
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
```

Update the "Scripts and Custom Blocks" table in `transcripts/stage.md` — mark S3 as Implemented.

Append to `transcripts/narrator-scroll.md` (after the script-status table):

```markdown
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
```

Update the "Scripts" table in `transcripts/narrator-scroll.md` — mark V1 as "partial (Task 4; C gate Task 6)".

- [ ] **Step 5: Save the `.sb3`.**

File → Save → overwrite `project/ant-adventure.sb3`.

- [ ] **Step 6: Commit.**

```bash
git add project/ant-adventure.sb3 transcripts/stage.md transcripts/narrator-scroll.md
git commit -m "$(cat <<'EOF'
engine: render scene 1 — V1 paint + S3 dispatcher

Why:
  - With S1 already building the scene table, adding the View's paint
    path (V1) and the render dispatcher (S3) makes scene 1 visible
    after green flag. First user-visible output of the project.
  - S3 uses `broadcast render_text and wait` before the reactor
    broadcast. This ordering was added by spec §10 review trail after
    the Sonnet peer review caught the race between render and reactor.
  - V1 skips rendering empty-label choices so the same block handles
    decision scenes AND transitions/endings (where labels are empty).

Where:
  - project/ant-adventure.sb3: S3 hat on Stage; V1 hat on Narrator
    Scroll. Comments on both hats explain intent and ordering.
  - transcripts/stage.md: S3 section with purpose, contract, blocks
  - transcripts/narrator-scroll.md: V1 section (Task 4 version,
    explicit note that choice C gate arrives in Task 6)

When: 2026-04-22 (Section: Task 4 render)

Verification:
  - Green flag paints scene 1 prose, then "1) Listen to the queen",
    then "2) Slip out to the tunnel quietly". Beetle still static
    (reactor is Task 9). No input yet (Task 5).
EOF
)"
```

**Done when:**
- Green flag paints scene 1 correctly.
- Scripts tables in both transcripts are updated.
- Commit on `main`.

---

## Task 5: CB2 `apply_choice`, CB4 `run_scene_side_effects`, S2 Input Hats, C1 Keyboard

**Files:**
- Modify: `project/ant-adventure.sb3` — add CB2 and CB4 on Stage, S2 hats on Stage, C1 hats on Worker Ant
- Modify: `transcripts/stage.md` — add CB2, CB4, S2 sections
- Modify: `transcripts/worker-ant.md` — add C1 section

After this task: player can press keys 1, 2, 3 to advance scenes. CB5 (`resolve_ending`) doesn't exist yet, so reaching scenes 10 or 11 (transitions) will leave `current_scene` stuck there — that's acceptable for this task; Task 7 completes the loop. The path-coverage test for endings that don't touch the transitions becomes runnable here (paths that end at 10/11 won't reach a true ending yet).

Actually, re-checking the spec: **every** path flows through either scene 10 or scene 11 (the transitions). So without CB5 (Task 7), no path reaches an ending. That's fine — Task 5's verification is "can I reach scene 9?" (yes, by pressing 1→1→1→1→1 etc. and advancing through the scene graph). Full path coverage runs in Task 7.

- [ ] **Step 1: Define CB4 `run_scene_side_effects (scene_id)` on Stage.**

**Variables → My Blocks → Make a Block.** Name: `run_scene_side_effects`. Add one input: `scene_id` (number).

Block-comment on the hat (exact):

```
CB4: Apply side effects on entry to a scene. Called by CB2
apply_choice after current_scene is updated. "Entry" semantics:
the scene_id parameter is the scene we're ENTERING, not the one
we're leaving. This is what lets the flag for scene 2 (has_orders)
be set when the player first arrives at 2.

Ladder covers flag sets (scenes 2, 6, 7) and transition-scene hooks
(10, 11 call resolve_ending — added in Task 7). Ladder entries for
end scenes (12-15) are defensive (reached via the transition route,
they are dead code; retained so a future debug shortcut doesn't
leave ending_code unset).
```

Block body (Task 5 version — `resolve_ending` calls are stubbed as comments; Task 7 adds them):

```
define run_scene_side_effects (scene_id)
  // flag sets on scene entry
  if <(scene_id) = (2)> then
    set [has_orders v] to (1)                 // queen's briefing
  end
  if <(scene_id) = (6)> then
    set [scout_trail_known v] to (1)          // met the scout
  end
  if <(scene_id) = (7)> then
    set [food_carried v] to (1)               // got the sugar
  end

  // transition-scene hooks — resolve_ending (CB5) added in Task 7
  // if <(scene_id) = (10)> then
  //   resolve_ending
  // end
  // if <(scene_id) = (11)> then
  //   resolve_ending
  // end

  // defensive ending_code sets — on the normal path these are dead
  // code because resolve_ending sets ending_code at the transitions,
  // but they guard future direct routes into 12-15.
  if <(scene_id) = (12)> then
    set [ending_code v] to (1)
  end
  if <(scene_id) = (13)> then
    set [ending_code v] to (2)
  end
  if <(scene_id) = (14)> then
    set [ending_code v] to (3)
  end
  if <(scene_id) = (15)> then
    set [ending_code v] to (4)
  end
```

- [ ] **Step 2: Define CB2 `apply_choice (letter)` on Stage.**

**Variables → My Blocks → Make a Block.** Name: `apply_choice`. Add one input: `letter` (text).

**Run without screen refresh: CHECKED.** This is critical — CB2 must execute atomically so `current_scene` and `ending_code` are both updated before any screen tick.

Block-comment on the hat (exact):

```
CB2: The heart of the interpreter. Advance the story state machine
by one step given the player's choice letter ("a", "b", or "c").

Locked contract (DO NOT reorder):
  1. Capture next_id from scene_next_<letter> at current_scene BEFORE
     mutating current_scene. If we read after mutating, side effects
     would fire for the wrong scene.
  2. Guard E2: if next_id = 0, no such choice on this scene — no-op.
  3. Guard E3: if ending_code > 0, story has ended — no-op. Replay
     via green flag.
  4. Set current_scene = next_id.
  5. run_scene_side_effects(current_scene) — operates on the scene
     JUST ENTERED.
  6. Broadcast render_scene.

"Run without screen refresh" is checked: CB2 must complete
atomically so both current_scene and ending_code are coherent before
any render/input tick. See ADR-0001 (data-driven engine) and spec
§4 for the full contract.
```

Block body:

```
define apply_choice (letter)
  // local capture — the spec's "next_id" local variable. Scratch
  // doesn't have true locals in custom blocks, so we use a Stage
  // variable `next_id` reserved for this block's use. Create it as
  // a temporary if not already present; it's not listed in the
  // 6 "story state" variables because it's a scratchpad.

  // Capture next scene id based on letter (if/elseif ladder — Scratch
  // doesn't support dynamic variable-name lookup).
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

  // E2 guard: no such choice on this scene
  if <(next_id) = (0)> then
    stop [this script v]
  end

  // E3 guard: story has ended, ignore further advances
  if <(ending_code) > (0)> then
    stop [this script v]
  end

  // advance the state machine
  set [current_scene v] to (next_id)
  run_scene_side_effects (current_scene)
  broadcast [render_scene v]
```

**Note on `next_id`:** Scratch custom blocks don't support true local variables. Create a new Stage variable `next_id` (For all sprites). It's a scratchpad used only inside CB2. Document this in the transcript. Its watcher should be unchecked (hidden).

Update the variables comment for `next_id` at creation: "Scratchpad used by CB2 apply_choice. Not a story-state variable. Hidden watcher."

- [ ] **Step 3: Add S2 input handlers on Stage (three hats).**

Block-comment on each hat (same text for all three, adjusted for letter):

```
S2a/b/c: Input entry point. Receive the `choose_a/b/c` broadcast
from the Controller (C1 on Worker Ant, V2 on Narrator Scroll from
Task 8) and call apply_choice with the corresponding letter. The
`ready` guard is on the broadcaster (C1/V2), not here, so we don't
duplicate the check.
```

Three separate hats:

```
when I receive [choose_a v]
  apply_choice [a]

when I receive [choose_b v]
  apply_choice [b]

when I receive [choose_c v]
  apply_choice [c]
```

- [ ] **Step 4: Add C1 keyboard handlers on Worker Ant (three hats).**

Switch to the Worker Ant sprite.

Block-comment on each C1 hat (adjusted for key):

```
C1(1/2/3): Keyboard input. Key [1/2/3] → broadcast choose_a/b/c.
Guarded by `ready` so input during S1 initialization is ignored
(spec E1). The actual choice lookup and scene advance happens in
CB2 on the Stage; this is pure input translation.
```

Three separate hats:

```
when [1 v] key pressed
  if <(ready) = (1)> then
    broadcast [choose_a v]                    // deliver to S2a on Stage
  end

when [2 v] key pressed
  if <(ready) = (1)> then
    broadcast [choose_b v]
  end

when [3 v] key pressed
  if <(ready) = (1)> then
    broadcast [choose_c v]
  end
```

- [ ] **Step 5: Verify.**

Click green flag. Scene 1 paints. Press `1`.

Expected:
- Scene 1's choice A (`scene_next_a[1]=2`) advances `current_scene` to 2.
- CB4 runs for scene 2 → `has_orders` = 1.
- V1 paints scene 2: "The queen's antennae brush yours..." then "1) Onward to the tunnel exit".
- Note: scene 2 has empty `scene_choice_b`, so no "2) ..." is shown.

Press `1` again → scene 3 ("You reach the tunnel exit...").

Press `1` → scene 4 ("A massive beetle..."). Press `2` → scene 8 ("Bruised and limping..."). Press `1` → scene 9 ("The nest is in sight..."). Should see "1) Sneak back quietly" and "2) Take the main tunnel". Choice C is not rendered because V1 doesn't check `scene_flag_required` yet (Task 6).

Press `1` → `current_scene` should become 11 (the sneaking transition). V1 paints "..." but no ending resolution happens (CB5 is Task 7). **The game is stuck here; this is expected for Task 5.** Clicking green flag restarts.

Also verify the E2 guard: from scene 2 (only choice A is live, B and C = 0), press `2` or `3`. Expected: nothing advances.

- [ ] **Step 6: Update transcripts.**

Append to `transcripts/stage.md` (after S3 section):

```markdown
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
```

Update the "Scripts and Custom Blocks" table in `transcripts/stage.md` — mark CB2, CB4, and all three S2 hats Implemented.

Append to `transcripts/worker-ant.md`:

```markdown
---

## C1. Keyboard input — keys `1`, `2`, `3`

**Purpose:** Translate keyboard input into `choose_a/b/c` broadcasts. The ant sprite is the Controller; it holds no story state. The `ready` guard drops input that arrives during S1 initialize.

**Blocks:**

```
when [1 v] key pressed
  if <(ready) = (1)> then
    broadcast [choose_a v]                        // S2a on Stage
  end

when [2 v] key pressed
  if <(ready) = (1)> then
    broadcast [choose_b v]
  end

when [3 v] key pressed
  if <(ready) = (1)> then
    broadcast [choose_c v]
  end
```

**Verification (Task 5):**
- Green flag + press `1` → advances to scene 2; `has_orders` becomes 1. ✓
- From scene 2, pressing `2` or `3` is a no-op (E2). ✓
- Keys pressed during S1's initial render are ignored (`ready=0`). ✓
```

Update the C1 rows in the scripts table in `transcripts/worker-ant.md` to Implemented.

- [ ] **Step 7: Save the `.sb3`.**

File → Save → overwrite `project/ant-adventure.sb3`.

- [ ] **Step 8: Commit.**

```bash
git add project/ant-adventure.sb3 transcripts/stage.md transcripts/worker-ant.md
git commit -m "$(cat <<'EOF'
engine: wire interpreter — CB2, CB4, S2, C1 keyboard input

Why:
  - CB2 apply_choice is the state machine's step function. Locked
    contract (capture-before-mutate, E2/E3 guards, update, side
    effect, broadcast) is essential for scene-entry semantics
    (spec §4). Marked "run without screen refresh" to keep
    current_scene and ending_code coherent across a tick.
  - CB4 run_scene_side_effects owns the ladder of entry hooks.
    Task 5 implements flag sets (scenes 2/6/7) and defensive
    ending_code sets; the transition hooks at 10/11 are stubbed
    as comments pending CB5 in Task 7.
  - S2's three hats are thin entry points from any Controller
    (keyboard now, clicks in Task 8).
  - C1 guards on `ready` so input during S1 init is dropped (E1).

Where:
  - project/ant-adventure.sb3: CB2 + CB4 defined on Stage with
    purpose comments; S2 a/b/c hats added on Stage; C1 keys 1/2/3
    added on Worker Ant with ready-guard comments; new Stage
    scratchpad variable `next_id` (hidden watcher).
  - transcripts/stage.md: CB2, CB4, S2 sections added with full
    block listings and contracts.
  - transcripts/worker-ant.md: C1 section added with per-hat
    blocks and Task 5 verification results.

When: 2026-04-22 (Section: Task 5 interpreter)

Verification:
  - Scene 1 + key 1 → scene 2, has_orders=1. Paint reflects scene 2.
  - Scene 2 + key 2 or 3 → no-op (E2 guard).
  - Scene 4 + key 1 → scene 7, food_carried=1 on arrival.
  - All paths reach scene 10 or 11 but stick there (expected — CB5 Task 7).
EOF
)"
```

**Done when:**
- Pressing 1/2/3 advances the story through the scene graph.
- Flag variables are set correctly at scenes 2, 6, 7.
- E2 guard ignores invalid keys.
- Commit on `main`.

---

## Task 6: CB3 `is_flag_set` and V1 Choice C Gate

**Files:**
- Modify: `project/ant-adventure.sb3` — add CB3 on Stage, extend V1 on Narrator Scroll with choice-C rendering
- Modify: `transcripts/stage.md` — add CB3 section
- Modify: `transcripts/narrator-scroll.md` — update V1 section to include the C gate

After this task: at scene 9 with `has_orders = 1`, the Narrator will render "3) Report to the queen first". Without `has_orders`, only A and B are shown. This unblocks the full path for ending 4 (Task 7 completes the final step).

- [ ] **Step 1: Define CB3 `is_flag_set (flag_id)` as a **reporter** (returns boolean).**

**Variables → My Blocks → Make a Block.** Name: `is_flag_set`. Add one input: `flag_id` (number).

**Check the "Make a Block" dialog's "reports boolean" option** — Scratch supports custom boolean reporters.

Block-comment on the hat:

```
CB3: Flag id → boolean. Translates the numeric flag id stored in
scene_flag_required into a boolean on the actual named variable.
This is the only place the flag id ↔ variable mapping lives — adding
a new flag means adding one branch here (not touching the scene
table). See ADR-0003 for the rationale on numeric ids vs names.

Valid inputs: 1 (food_carried), 2 (scout_trail_known), 3 (has_orders).
Any other input (including 0) reports false.
```

Block body:

```
define is_flag_set (flag_id)
  if <(flag_id) = (1)> then
    <(food_carried) = (1)> ::reporter   // reporter: the comparison's boolean value is reported
  end
  if <(flag_id) = (2)> then
    <(scout_trail_known) = (1)>
  end
  if <(flag_id) = (3)> then
    <(has_orders) = (1)>
  end
  // fallthrough: false
```

Note: Scratch's custom-block **reporter** bodies are slightly awkward. The idiomatic pattern is a chain of `if ... report <value>` — but Scratch custom reporter blocks have a specific "report" return mechanism. The cleanest way to build this in Scratch 3.0 is:

```
define is_flag_set (flag_id) :: boolean
  if <(flag_id) = (1)> then
    <(food_carried) = (1)> ::reporter  // wire directly into the block's return
  else if <(flag_id) = (2)> then
    <(scout_trail_known) = (1)>
  else if <(flag_id) = (3)> then
    <(has_orders) = (1)>
  else
    <(false) = (true)>                 // reports false when flag_id not recognized
  end
```

**Implementation note for the executor:** Scratch 3.0 custom reporters return the last expression evaluated in a reporter context. If your authoring tool (Path B in Task 0) requires a different pattern, adapt the logic — the behavior contract is: given `flag_id ∈ {1,2,3}`, return the corresponding flag variable's boolean truth. For any other input, return false. Document the exact block tree you built in the transcript.

- [ ] **Step 2: Extend V1 on Narrator Scroll with the choice-C gate.**

Switch to the Narrator Scroll sprite. Open V1. Add the choice-C rendering block at the end (replacing the "choice C rendering added in Task 6" comment).

Append to V1:

```
  // choice C — gated by scene_flag_required + is_flag_set. If the
  // scene has no gate (flag_id=0), we never render C. If it has a
  // gate, we render C only when the flag is set AND the label is
  // non-empty. The two-condition check lets authors "forget" to
  // clear scene_choice_c when disabling a gate, without visual
  // regressions.
  if <<not <(item (current_scene) of [scene_flag_required v]) = (0)>>
      and <is_flag_set (item (current_scene) of [scene_flag_required v])>
      and <not <(item (current_scene) of [scene_choice_c v]) = []>>> then
    say (join [3) ] (item (current_scene) of [scene_choice_c v])) for (2) seconds
  end
```

Update the hat comment on V1 to remove the "choice C rendering added in Task 6" note — the block is now complete.

- [ ] **Step 3: Verify.**

Click green flag. Walk to scene 9 with `has_orders = 1`:
- Press `1` → scene 2 (sets `has_orders=1`)
- Press `1` → scene 3
- Press `1` → scene 4 (beetle)
- Press `1` → scene 7 (sugar, sets `food_carried=1`)
- Press `1` → scene 9

Expected at scene 9:
- Prose: "The nest is in sight. One last choice: sneak or report?"
- Labels: "1) Sneak back quietly", "2) Take the main tunnel", "3) Report to the queen first"

Now green flag again and walk a `has_orders=0` path:
- Press `2` → scene 5 (no flag)
- Press `1` → scene 3
- Press `1` → scene 4
- Press `1` → scene 7 (sets food)
- Press `1` → scene 9

Expected at scene 9:
- Only labels 1) and 2) shown. Label 3 is NOT rendered.

- [ ] **Step 4: Update transcripts.**

Append to `transcripts/stage.md` (after CB4 section):

```markdown
---

## CB3. `is_flag_set (flag_id)` — 1 input, reports boolean

**Purpose:** Translate a numeric flag id (as stored in `scene_flag_required`) to a boolean on the corresponding named variable. This is the sole place the id ↔ variable mapping lives — adding a flag = adding one branch here, nothing else. See ADR-0003.

**Contract:**
- Input: 1 (food_carried), 2 (scout_trail_known), 3 (has_orders).
- Any other input (including 0) reports false.
- Pure reporter; reads only.

**Blocks:**

```
define is_flag_set (flag_id) :: boolean
  if <(flag_id) = (1)> then
    <(food_carried) = (1)>
  else if <(flag_id) = (2)> then
    <(scout_trail_known) = (1)>
  else if <(flag_id) = (3)> then
    <(has_orders) = (1)>
  else
    <(false) = (true)>                          // reports false
  end
```
```

Update the scripts/custom-blocks table in `transcripts/stage.md` — mark CB3 Implemented.

Update the V1 section in `transcripts/narrator-scroll.md`. Replace the **(Task 4 version — no choice C yet)** block listing with the complete Task 6 version that includes the C gate:

```markdown
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

  // choice C — gated by scene_flag_required + is_flag_set. Triple
  // condition: flag id nonzero, flag is set, label non-empty. See
  // CB3 and ADR-0003.
  if <<not <(item (current_scene) of [scene_flag_required v]) = (0)>>
      and <is_flag_set (item (current_scene) of [scene_flag_required v])>
      and <not <(item (current_scene) of [scene_choice_c v]) = []>>> then
    say (join [3) ] (item (current_scene) of [scene_choice_c v])) for (2) seconds
  end
```

**Verification (Task 6):**
- Scene 9 with `has_orders=1` paints labels 1, 2, 3. ✓
- Scene 9 with `has_orders=0` paints labels 1, 2 only. ✓
- Scenes with `scene_flag_required=0` never paint label 3 (no scene checks this directly; 15 scenes have fid=0, only scene 9 has fid=3).
```

Update the V1 row in the scripts table — mark Implemented.

- [ ] **Step 5: Save the `.sb3`.**

File → Save → overwrite `project/ant-adventure.sb3`.

- [ ] **Step 6: Commit.**

```bash
git add project/ant-adventure.sb3 transcripts/stage.md transcripts/narrator-scroll.md
git commit -m "$(cat <<'EOF'
engine: gate choice C with CB3 is_flag_set in V1

Why:
  - Choice C at scene 9 is the only flag-gated choice in the project.
    Wiring is_flag_set (CB3) and extending V1's render with the
    gate is what makes the has_orders decision matter: players
    who skip the queen's briefing don't see "Report" as an option.
  - CB3 is a custom boolean reporter — the sole place flag ids map
    to flag variables, per ADR-0003. Adding a flag later = adding
    one branch in CB3, no scene-table changes.
  - V1's triple-condition gate (flag id nonzero, flag set, label
    non-empty) is defensive against authoring mistakes without
    introducing runtime branches elsewhere.

Where:
  - project/ant-adventure.sb3: CB3 reporter defined on Stage with
    purpose comment and false fallthrough; V1 on Narrator Scroll
    extended with the C-label render conditional.
  - transcripts/stage.md: CB3 section added with contract and
    block listing.
  - transcripts/narrator-scroll.md: V1 block listing replaced with
    the complete Task 6 version including the gate; Task 6
    verification results recorded.

When: 2026-04-22 (Section: Task 6 flag gate)

Verification:
  - Path 1→2→3→4→7→9 (has_orders=1): scene 9 shows labels 1, 2, 3.
  - Path 1→5→3→4→7→9 (has_orders=0): scene 9 shows labels 1, 2 only.
EOF
)"
```

**Done when:**
- Choice C renders at scene 9 if and only if `has_orders=1`.
- CB3 is the only flag-id-to-variable dispatcher.
- Commit on `main`.

---

## Task 7: CB5 `resolve_ending` and CB4 Wire-Up — All Four Paths Pass

**Files:**
- Modify: `project/ant-adventure.sb3` — add CB5 on Stage, activate the CB5 calls inside CB4
- Modify: `transcripts/stage.md` — add CB5 section, update CB4 block listing

This is the payoff task — all four spec §6 test-matrix paths become walkable to completion.

- [ ] **Step 1: Define CB5 `resolve_ending` on Stage (no inputs).**

**Variables → My Blocks → Make a Block.** Name: `resolve_ending`. No inputs.

Block-comment on the hat:

```
CB5: Flag state → ending. Called by CB4 when the story enters a
transition scene (10 reporting, 11 sneaking). Reads the three
flags, picks the correct ending scene id, and sets BOTH
current_scene and ending_code in one atomic operation (since CB4
is only called once per apply_choice step and will not re-enter
for the new scene).

The ladder order matters:
  1. has_orders=1 first — full glory beats everything else
  2. scout + food — hero shortcut
  3. food alone — triumph
  4. default — empty-handed

This is the ONLY place multi-flag combination happens in the
engine. See ADR-0004.
```

Block body:

```
define resolve_ending
  if <(has_orders) = (1)> then
    set [current_scene v] to (15)             // ending 4: full glory
    set [ending_code v] to (4)
  else
    if <<(scout_trail_known) = (1)> and <(food_carried) = (1)>> then
      set [current_scene v] to (14)           // ending 3: hero shortcut
      set [ending_code v] to (3)
    else
      if <(food_carried) = (1)> then
        set [current_scene v] to (12)         // ending 1: triumph
        set [ending_code v] to (1)
      else
        set [current_scene v] to (13)         // ending 2: empty-handed
        set [ending_code v] to (2)
      end
    end
  end
```

- [ ] **Step 2: Update CB4 to call CB5 at rows 10 and 11.**

Open CB4's define. Remove the commented-out `resolve_ending` stub and activate:

```
define run_scene_side_effects (scene_id)
  // flag sets on scene entry
  if <(scene_id) = (2)> then set [has_orders v] to (1) end
  if <(scene_id) = (6)> then set [scout_trail_known v] to (1) end
  if <(scene_id) = (7)> then set [food_carried v] to (1) end

  // transition scene hooks — fold flag state into ending id
  if <(scene_id) = (10)> then                 // reporting transition
    resolve_ending
  end
  if <(scene_id) = (11)> then                 // sneaking transition
    resolve_ending
  end

  // defensive ending_code sets — unreached on the normal path (CB5
  // already set ending_code at the transition), retained for
  // direct-route safety
  if <(scene_id) = (12)> then set [ending_code v] to (1) end
  if <(scene_id) = (13)> then set [ending_code v] to (2) end
  if <(scene_id) = (14)> then set [ending_code v] to (3) end
  if <(scene_id) = (15)> then set [ending_code v] to (4) end
```

Update CB4's hat comment — strike "CB5 added in Task 7" and note "CB5 wired in."

- [ ] **Step 3: Verify — walk all four paths from the spec §6 test matrix.**

**Note on scene id mapping:** the spec's labels 10/11/12/13 are Scratch list indices 12/13/14/15. Ending numbers in the spec ("ending 10 = triumph") = ending_code 1. Below I use the Scratch indices for clarity.

**Path A — Ending 1 (triumph):** `1 → 5 → 3 → 4 → 7 → 9 (sneak) → 11 → 12`

Keys: green flag, `2` (sneak out early → scene 5), `1` (continue → scene 3), `1` (short route → scene 4), `1` (dash past → scene 7), `1` (home → scene 9), `1` or `2` (sneak/main tunnel, both go to 11).

Expected at end:
- `current_scene = 12`
- `ending_code = 1`
- `food_carried = 1`, `has_orders = 0`, `scout_trail_known = 0`
- Narrator paints "Triumph! You haul the sugar to the larder. The colony feasts tonight."
- Pressing any key after this is a no-op (E3 guard holds).

**Path B — Ending 2 (empty-handed):** `1 → 5 → 3 → 4 → 8 → 9 (sneak) → 11 → 13`

Keys: green flag, `2`, `1`, `1`, `2` (retreat → scene 8), `1` (home → scene 9), `1` (sneak → scene 11).

Expected:
- `current_scene = 13`, `ending_code = 2`
- All flags = 0
- Narrator paints "You return empty-handed..."

**Path C — Ending 3 (hero shortcut):** `1 → 5 → 3 → 6 → 7 → 9 (sneak) → 11 → 14`

Keys: green flag, `2`, `1`, `2` (long route → scene 6), `1` (scout path → scene 7), `1` (home → scene 9), `1` (sneak → scene 11).

Expected:
- `current_scene = 14`, `ending_code = 3`
- `food_carried = 1`, `scout_trail_known = 1`, `has_orders = 0`
- Narrator paints "The scout's shortcut shaved hours off your trip..."

**Path D — Ending 4 (full glory):** `1 → 2 → 3 → 6 → 7 → 9 (report C) → 10 → 15`

Keys: green flag, `1` (listen → scene 2, sets has_orders=1), `1` (onward → scene 3), `2` (long route → scene 6), `1` (scout path → scene 7), `1` (home → scene 9), `3` (report → scene 10).

Expected:
- Scene 9 shows choice C because has_orders=1.
- `current_scene = 15`, `ending_code = 4`
- All three flags = 1
- Narrator paints "You reported to the queen, delivered the sugar..."

**If any of these fail, stop and debug.** The most likely source of failure is a typo in the scene-table data (Task 3). Check `scene_next_*` cells for the scenes on the failing path.

**Additionally verify the input robustness:**
- After reaching any ending, press `1/2/3` → no advance (E3 guard holds).
- At the start, hammering keys during the opening 3-second prose say → input is queued by Scratch's broadcast system but `apply_choice` runs with `ready=1` so it actually advances. That is expected — E1 only guards the very first moments before S1's `ready=1` line.

- [ ] **Step 4: Update transcripts.**

Append to `transcripts/stage.md` (after CB3 section):

```markdown
---

## CB5. `resolve_ending` — no inputs

**Purpose:** Fold the current flag state into a specific ending scene id, and atomically set both `current_scene` and `ending_code`. Called by CB4 on entry to transition scenes 10 (reporting) and 11 (sneaking). This is the only place in the engine where multi-flag combinations determine control flow — see ADR-0004.

**Contract:**
- Sets `current_scene` to 12, 13, 14, or 15 (the ending scenes).
- Sets `ending_code` to the matching 1, 2, 3, or 4.
- Does not broadcast; the broadcast happens from CB2 after CB4 returns.
- Ladder order is essential: `has_orders` beats everything, then scout+food, then food alone, else empty-handed.

**Blocks:**

```
define resolve_ending
  if <(has_orders) = (1)> then
    set [current_scene v] to (15)              // ending 4: full glory
    set [ending_code v] to (4)
  else
    if <<(scout_trail_known) = (1)> and <(food_carried) = (1)>> then
      set [current_scene v] to (14)            // ending 3: hero shortcut
      set [ending_code v] to (3)
    else
      if <(food_carried) = (1)> then
        set [current_scene v] to (12)          // ending 1: triumph
        set [ending_code v] to (1)
      else
        set [current_scene v] to (13)          // ending 2: empty-handed
        set [ending_code v] to (2)
      end
    end
  end
```

**Verification (Task 7):**
- Path A (1→5→3→4→7→9→11): ends at 12, ending_code=1. ✓
- Path B (1→5→3→4→8→9→11): ends at 13, ending_code=2. ✓
- Path C (1→5→3→6→7→9→11): ends at 14, ending_code=3. ✓
- Path D (1→2→3→6→7→9[C]→10): ends at 15, ending_code=4. ✓
```

Update the CB4 block listing in `transcripts/stage.md` to the Task 7 version (activate the CB5 calls).

Update the scripts/custom-blocks table in `transcripts/stage.md` — mark CB5 Implemented.

- [ ] **Step 5: Save the `.sb3`.**

File → Save → overwrite `project/ant-adventure.sb3`.

- [ ] **Step 6: Commit.**

```bash
git add project/ant-adventure.sb3 transcripts/stage.md
git commit -m "$(cat <<'EOF'
engine: CB5 resolve_ending + CB4 wire-up — all four paths pass

Why:
  - CB5 is the sole combiner of multi-flag state into a routing
    decision (ADR-0004). It folds has_orders + scout_trail_known
    + food_carried into one of four ending scene ids and sets
    ending_code in the same block — avoiding a second CB4 pass
    which would double-invoke scene-entry side effects.
  - Activating the CB5 calls inside CB4 at rows 10/11 completes
    the return-trip flow. Entry to a transition scene
    deterministically routes the player to the ending that
    matches their journey.
  - All four spec §6 test-matrix paths now reach their expected
    ending and ending_code.

Where:
  - project/ant-adventure.sb3: CB5 defined on Stage with ladder
    order commented in the hat; CB4 updated to call resolve_ending
    at scene_id=10 and 11 (stubs replaced with live calls).
  - transcripts/stage.md: CB5 section added with full block
    listing, contract, and per-path verification results; CB4
    block listing updated to the Task 7 version.

When: 2026-04-22 (Section: Task 7 resolve endings)

Verification:
  - All 4 test-matrix paths walked to completion:
    A: 1→5→3→4→7→9→11 → 12 (triumph, code=1)
    B: 1→5→3→4→8→9→11 → 13 (empty-handed, code=2)
    C: 1→5→3→6→7→9→11 → 14 (hero shortcut, code=3)
    D: 1→2→3→6→7→9[3]→10 → 15 (full glory, code=4)
  - After each ending, E3 guard blocks further input.
EOF
)"
```

**Done when:**
- All four spec §6 paths walked successfully.
- `ending_code` is set correctly at each ending.
- E3 guard demonstrably blocks input after any ending.
- Commit on `main`.

---

## Task 8: V2 Narrator Click Routing

**Files:**
- Modify: `project/ant-adventure.sb3` — add V2 on Narrator Scroll
- Modify: `transcripts/narrator-scroll.md` — add V2 section

After this task: player can pick choices by clicking on the Narrator Scroll in addition to using keys. This duplicates C1's broadcast path — both keyboard and click send the same `choose_a/b/c` messages, so the downstream interpreter is untouched.

**Design note on click zones:** V1 renders choice labels via `say` bubbles, which are transient Scratch UI overlays — they do NOT create clickable regions. So "click on choice label" is not mechanically meaningful. Instead, V2 uses three **screen regions** of the scroll itself:

- **Left third** of the scroll → choice A
- **Middle third** → choice B
- **Right third** → choice C

These regions are relative to the scroll's position (x=0, y=60) with the scroll's 260-pixel width (so zones: x<-43 → A, -43≤x≤43 → B, x>43 → C). This is a pragmatic compromise given Scratch's UI primitives; it's documented in the transcript so reviewers aren't surprised.

- [ ] **Step 1: Add V2 on Narrator Scroll.**

Switch to the Narrator Scroll sprite.

Block-comment on the hat:

```
V2: Click routing. When the player clicks the Narrator Scroll,
determine which third of the scroll was clicked (by mouse-x relative
to the scroll's x-center) and broadcast the matching choose_a/b/c.

Why regions, not label-clicks: Scratch's "say" bubbles are not
clickable regions; they are UI overlays. So we use the scroll body
itself as the input surface, divided into thirds. This matches C1's
broadcast path exactly — the interpreter is unchanged.

Guarded by `ready` (same E1 rule as C1).
```

Block body:

```
when this sprite clicked
  if <not <(ready) = (1)>> then
    stop [this script v]                       // E1: ignore clicks during init
  end
  // scroll center is at x=0. Zone widths ~86 px each (scroll is 260 px).
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

- [ ] **Step 2: Verify.**

Green flag. Scene 1 paints. Click the left third of the scroll → broadcast `choose_a` → advances to scene 2 (same as pressing `1`). Click the right third at a scene with `has_orders=1` and a gated C → broadcasts `choose_c`. At scene 2 (has only choice A), clicking the middle or right third broadcasts `choose_b`/`choose_c` respectively, which both hit the E2 guard in CB2 and are no-ops.

Walk one of the four paths (your choice) entirely by clicking rather than pressing keys. Verify the same ending is reached as before.

- [ ] **Step 3: Update `transcripts/narrator-scroll.md`.**

Append (after the V1 section):

```markdown
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
```

Update the V2 row in the scripts table — mark Implemented.

- [ ] **Step 4: Save the `.sb3`.**

File → Save → overwrite `project/ant-adventure.sb3`.

- [ ] **Step 5: Commit.**

```bash
git add project/ant-adventure.sb3 transcripts/narrator-scroll.md
git commit -m "$(cat <<'EOF'
engine: add V2 click routing on Narrator Scroll

Why:
  - V2 adds mouse input parallel to keyboard (C1). Both funnel
    into the same choose_a/b/c broadcasts, so the interpreter
    path is shared — no new engine branches.
  - Scratch "say" bubbles are not clickable; V2 uses the scroll
    body divided into thirds (mouse-x zones) as the input surface.
    Documented in the transcript so the choice is visible to
    reviewers.
  - Guarded by `ready` (same E1 rule as C1). Invalid choices
    (e.g. C when no C) are caught downstream by CB2 E2.

Where:
  - project/ant-adventure.sb3: V2 hat on Narrator Scroll with
    zone dispatch and ready guard.
  - transcripts/narrator-scroll.md: V2 section with zone
    definitions, contract, and task verification results.

When: 2026-04-22 (Section: Task 8 click input)

Verification:
  - Clicks at left/middle/right zones of scroll advance to A/B/C
    choices at scene 1.
  - Full path walked entirely by clicking reaches the same ending
    as the keyboard path.
  - Clicks during init are dropped by the ready guard.
EOF
)"
```

**Done when:**
- Clicks on the three scroll zones broadcast the right choice.
- A full path can be played with mouse only.
- Commit on `main`.

---

## Task 9: R1 Beetle Show/Hide Reactor

**Files:**
- Modify: `project/ant-adventure.sb3` — add a second costume for Beetle (menacing pose), add R1 on Beetle
- Modify: `transcripts/beetle.md` — add R1 section

After this task: Beetle appears only at scene 4 with a small wiggle animation; hidden elsewhere.

- [ ] **Step 1: Add a second costume to Beetle for the menacing pose.**

Select the Beetle sprite. In the Costumes tab, duplicate the existing costume. Rename them:
- Costume 1: `beetle-idle`
- Costume 2: `beetle-menace`

On the `beetle-menace` costume, mouse-tweak it slightly — raise the mandibles by a few pixels, or increase the pronotum size by ~10%. The differences should be small but perceptible when alternated.

- [ ] **Step 2: Add R1 hat on Beetle.**

Block-comment on the hat:

```
R1: Reactor. Beetle shows at scene 4 (the beetle-encounter scene)
with a brief mandible-raising wiggle. Hidden at every other scene.
Pure read on current_scene — the Beetle never writes story state.

The wiggle is 4 costume swaps over ~0.4 seconds, deliberately brief
so it doesn't block the render pipeline for long. S3 broadcasts
update_reactors AFTER render_text and wait, so the Narrator's paint
completes before the wiggle starts (no visible jitter).
```

Block body:

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

- [ ] **Step 3: Verify.**

Green flag. Navigate to scene 4 (press `1`→`1`→`1`→`1` from the "listen" start). Expected: Beetle appears at the right side of the stage with a brief wiggle animation. Beetle is hidden at scene 1, 2, 3, 5, 6, 7, 8, 9, and all endings.

- [ ] **Step 4: Update `transcripts/beetle.md`.**

Append (after the script-status table):

```markdown
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
```

Update the R1 row in the scripts table — mark Implemented.

- [ ] **Step 5: Save the `.sb3`.**

File → Save → overwrite `project/ant-adventure.sb3`.

- [ ] **Step 6: Commit.**

```bash
git add project/ant-adventure.sb3 transcripts/beetle.md
git commit -m "$(cat <<'EOF'
engine: Beetle reactor — show at scene 4, hide elsewhere

Why:
  - R1 completes the visual feedback for scene 4 (the beetle
    encounter). Adds a small wiggle animation via two costumes
    so the obstacle feels present rather than static.
  - Beetle is a pure reactor — reads current_scene, never writes
    state. Demonstrates the MVC reactor role cleanly.
  - S3's render_text-and-wait ordering (Task 4) means the wiggle
    starts AFTER the Narrator's paint finishes, preventing the
    race identified in Sonnet review trail.

Where:
  - project/ant-adventure.sb3: Beetle gets a second costume
    (beetle-menace) alongside beetle-idle; R1 hat added with
    show/wiggle/hide logic.
  - transcripts/beetle.md: R1 section added with costumes,
    timing note, and verification results.

When: 2026-04-22 (Section: Task 9 beetle reactor)

Verification:
  - Scene 4: beetle visible with wiggle.
  - Scenes 1-3, 5-9, 10-15: beetle hidden.
  - Wiggle does not overlap narrator paint (S3 ordering holds).
EOF
)"
```

**Done when:**
- Beetle visible at scene 4 with wiggle, hidden elsewhere.
- Commit on `main`.

---

## Task 10: Comment Sweep and Test-Matrix Pass

**Files:**
- Modify: `project/ant-adventure.sb3` — ensure every comment site (per spec §7) is present
- Modify: `transcripts/stage.md` — append a "Test Matrix Results" section
- Modify: `transcripts/narrator-scroll.md`, `transcripts/worker-ant.md`, `transcripts/beetle.md` — consistency scan

No new logic in this task. This is the polish pass. Reviewer will read every transcript and spot-check comments in Scratch; both must carry the same intent.

- [ ] **Step 1: Comment-site audit.**

Open the `.sb3`. For each of the following sites, confirm a Scratch block-comment exists and matches the transcript:

**Stage — Variables** (via a reporter block dragged + commented + reporter deleted):
- [ ] `current_scene` has a comment
- [ ] `food_carried` has a comment
- [ ] `scout_trail_known` has a comment
- [ ] `has_orders` has a comment
- [ ] `ending_code` has a comment
- [ ] `ready` has a comment
- [ ] `next_id` has a comment saying "scratchpad for CB2"

**Stage — Lists** (same technique):
- [ ] All 8 scene_* lists have comments

**Stage — Script and custom-block hats:**
- [ ] S1 hat: initialize comment
- [ ] S2 × 3 hats: entry-point comment (same text adjusted for letter)
- [ ] S3 hat: render dispatcher comment (mentions broadcast-and-wait race fix)
- [ ] CB1 hat: add_scene purpose + invariant
- [ ] CB2 hat: apply_choice contract (ordered steps)
- [ ] CB3 hat: is_flag_set id-to-variable map
- [ ] CB4 hat: run_scene_side_effects entry semantics
- [ ] CB5 hat: resolve_ending ladder order + ADR-0004 reference

**Narrator Scroll:**
- [ ] V1 hat: paint logic, choice C gate note
- [ ] V2 hat: click zones (left/middle/right thirds)

**Worker Ant:**
- [ ] C1 × 3 hats: keyboard input with ready guard

**Beetle:**
- [ ] R1 hat: show-at-4 / hide-elsewhere, timing note

**Non-obvious conditionals (inside block bodies):**
- [ ] CB2: E2 guard site has an inline comment "no such choice on this scene"
- [ ] CB2: E3 guard site has an inline comment "story already ended"
- [ ] V1: choice C triple-condition has an inline comment explaining the three checks
- [ ] S1: "delete all of [scene_*]" section has one comment explaining "clear + rebuild" (not one per list — one for the group)
- [ ] CB5: the ladder has per-branch inline comments naming the ending

**Broadcasts:**
- [ ] S1's `broadcast render_scene and wait`: inline comment "paint scene 1; wait ensures first frame up before input"
- [ ] S3's two broadcasts: comments naming the listener (V1, R1)
- [ ] C1 × 3 `broadcast choose_*`: inline comment "delivered to S2 on Stage"
- [ ] V2 × 3 `broadcast choose_*`: inline comments matching

If any are missing, add them. Save after each batch.

- [ ] **Step 2: Run the full test matrix and record results.**

Run all four paths from spec §6 one more time. Also run the flag-gating and input-robustness probes. Record PASS/FAIL for each. Expected: all PASS.

| Check | Expected | Actual |
|---|---|---|
| Path A (ending 1) — 1→5→3→4→7→9→11→12 | current=12, code=1, food=1 only | ? |
| Path B (ending 2) — 1→5→3→4→8→9→11→13 | current=13, code=2, no flags | ? |
| Path C (ending 3) — 1→5→3→6→7→9→11→14 | current=14, code=3, food+scout | ? |
| Path D (ending 4) — 1→2→3→6→7→9[C]→10→15 | current=15, code=4, all flags | ? |
| Scene 9 with has_orders=0: only A, B render | labels 1 & 2 only | ? |
| Scene 9 with has_orders=1: A, B, C render | labels 1 & 2 & 3 | ? |
| Key `3` at scene 2 (no C) | no advance | ? |
| Key pressed during S1 | no advance (ready=0) | ? |
| Key pressed after any ending | no advance (E3) | ? |
| Click outside scroll zones | broadcast B (middle zone captures unclassified clicks) — document as expected | ? |
| Beetle at scene 4 | visible with wiggle | ? |
| Beetle at all other scenes | hidden | ? |

Fill in "Actual" by playing each.

- [ ] **Step 3: Append a "Test Matrix Results" section to `transcripts/stage.md`.**

Append at the bottom of the file:

```markdown
---

## Test Matrix Results (recorded Task 10)

Run against `project/ant-adventure.sb3` at commit `<commit hash before this one>`.

| Check | Expected | Actual |
|---|---|---|
| Path A (ending 1: triumph) | current=12, code=1, food=1 | PASS |
| Path B (ending 2: empty-handed) | current=13, code=2, no flags | PASS |
| Path C (ending 3: hero shortcut) | current=14, code=3, food+scout | PASS |
| Path D (ending 4: full glory) | current=15, code=4, all flags | PASS |
| Scene 9 has_orders=0 → labels 1 & 2 only | choice C hidden | PASS |
| Scene 9 has_orders=1 → labels 1, 2, 3 | choice C rendered | PASS |
| Key `3` at scene 2 → no advance | E2 guard holds | PASS |
| Key during S1 init → no advance | E1 guard holds | PASS |
| Key after ending → no advance | E3 guard holds | PASS |
| Click outside defined zones → middle-zone B | documented default | PASS |
| Beetle at scene 4 → visible + wiggle | expected | PASS |
| Beetle elsewhere → hidden | expected | PASS |

All twelve checks pass. The engine meets the spec.
```

(Replace `<commit hash before this one>` with the actual hash — `git rev-parse HEAD` before committing this task gives you the correct one.)

- [ ] **Step 4: Consistency scan across transcripts.**

Read all four transcripts end-to-end. Check:
- Every `Status` row in the script/CB tables is "Implemented" (no "Task N" leftovers).
- All scene index references use the locked integers (no stray 9b/9c labels from the spec).
- Every ADR citation points to an existing file.

If anything is off, fix it.

- [ ] **Step 5: Save the `.sb3`.**

File → Save → overwrite `project/ant-adventure.sb3`. Even if no logic changed, comment additions did.

- [ ] **Step 6: Commit.**

```bash
git add project/ant-adventure.sb3 transcripts/
git commit -m "$(cat <<'EOF'
transcript: comment-site sweep + full test-matrix results

Why:
  - Spec §7 comment policy requires a comment at every custom
    block hat, every broadcast, every non-obvious conditional,
    every variable, and every list. This task audits all sites
    in the .sb3 and confirms each carries a Scratch block-comment
    that matches the transcripts.
  - Recording the full test-matrix pass (spec §6) in
    transcripts/stage.md gives the reviewer a single place to
    audit "does the implementation meet the spec" without running
    the game.

Where:
  - project/ant-adventure.sb3: added/verified Scratch block
    comments at every site in the Task 10 audit checklist (full
    list in the plan).
  - transcripts/stage.md: appended a Test Matrix Results section
    with all 12 checks PASS; consistency scan across sprites.
  - transcripts/*.md: status-column cleanup, ADR link verification.

When: 2026-04-22 (Section: Task 10 comment sweep)

Verification:
  - All 12 spec §6 test-matrix checks PASS in the current .sb3.
  - Every required comment site has a block-comment in Scratch
    AND a matching // comment in the transcript.
  - All transcripts reference locked scene integers (no 9b/9c).
EOF
)"
```

**Done when:**
- Every comment site in the §7 policy is present in both Scratch and the transcript.
- All 12 test-matrix checks PASS.
- Commit on `main`.

---

## Task 11: Publish, README Update, Final Commit

**Files:**
- Modify: `README.md` — add the Scratch project URL once published
- Modify: `project/ant-adventure.sb3` — no changes, but re-export to refresh any metadata

- [ ] **Step 1: Publish to scratch.mit.edu.**

In Scratch (with your account logged in), open the project, then **Share** at the top-right. This makes the project visible at its public URL. Copy the URL (format: `https://scratch.mit.edu/projects/<PROJECT_ID>/`).

(If the executor is non-interactive or has no Scratch account, skip this step and update the README to say "Local .sb3 — load in Scratch by File → Load from your computer.")

- [ ] **Step 2: Update `README.md`.**

Open `README.md`. Find the line:

```markdown
**Play:** *link to scratch.mit.edu project goes here once published*
```

Replace with (if published):

```markdown
**Play:** https://scratch.mit.edu/projects/<PROJECT_ID>/
```

Or (if not published):

```markdown
**Play:** Load `project/ant-adventure.sb3` in Scratch (https://scratch.mit.edu → File → Load from your computer). Then click the green flag.
```

Additionally, append a "Status" section to the README:

```markdown

---

## Status

- [x] Spec approved (2026-04-22)
- [x] Plan approved (2026-04-22)
- [x] Assets created (Task 1)
- [x] Scratch project scaffolded (Task 2)
- [x] Scene table + initialize (Task 3)
- [x] Render path (Task 4)
- [x] Interpreter + keyboard input (Task 5)
- [x] Choice-C flag gate (Task 6)
- [x] Ending resolver — all four paths pass (Task 7)
- [x] Click input (Task 8)
- [x] Beetle reactor (Task 9)
- [x] Comment sweep + test matrix (Task 10)
- [x] Published / README updated (Task 11)

All spec §8 requirements met. See `transcripts/stage.md` §"Test Matrix Results" for the final verification log.
```

- [ ] **Step 3: Re-export `.sb3`.**

Open the `.sb3` in Scratch (if not already). File → Save to your computer → overwrite. This refreshes the embedded project metadata (project name, last-saved date) without changing logic.

- [ ] **Step 4: Commit.**

```bash
git add README.md project/ant-adventure.sb3
git commit -m "$(cat <<'EOF'
docs: publish to scratch.mit.edu and finalize README

Why:
  - Share the project on Scratch so the assignment reviewer can
    play it in-browser via the public URL. Author-added commit
    requirement: the code must be on GitHub AND the game must be
    playable at scratch.mit.edu.
  - README status checklist gives reviewers a one-page sign-off
    of completed tasks and a pointer to the test-matrix results
    in transcripts/stage.md.

Where:
  - README.md: Play link updated to scratch.mit.edu URL;
    status section added with checklist of completed tasks.
  - project/ant-adventure.sb3: re-exported to refresh metadata
    (no logic changes).

When: 2026-04-22 (Section: Task 11 publish)

Verification:
  - scratch.mit.edu URL opens the project and green flag walks
    all four paths successfully.
  - README renders correctly on GitHub.
  - git log --oneline shows 11 commits in task order from spec
    through assets through engine through polish.
EOF
)"
```

**Done when:**
- README points to the playable URL (or local-load instructions if not published).
- `git log --oneline` shows the full task progression.
- Commit on `main`.

---

## Review Handoff Checklist

When tasks 0-11 are complete, the reviewer (you-the-author) should be able to:

1. `git log --oneline` — see a clean progression: spec → plan → assets → skeleton → init → render → interpreter → gate → endings → clicks → reactor → polish → publish.
2. Open `transcripts/stage.md` — read the full Stage model top-to-bottom and see every variable, list, script, and custom block documented with purpose paragraph, contract, block listing, and matching inline comments.
3. Open `transcripts/narrator-scroll.md`, `worker-ant.md`, `beetle.md` — see the same structure for each sprite.
4. Open `project/ant-adventure.sb3` in Scratch — spot-check by walking one or two of the four paths to confirm the transcript matches reality.
5. Read `transcripts/stage.md` "Test Matrix Results" — confirm all 12 checks PASS.
6. Review any single commit's diff in isolation — the commit body's Why/Where/When/Verification tells you what that commit accomplishes without needing to trace backward.

If all six are true, the implementation is ready to ship.

---

## Spec Coverage Self-Check (run during plan writing)

| Spec item | Plan task(s) |
|---|---|
| §1 learning frame (data-driven, state machine, MVC, flags) | Task 3 (scene table), 5 (interpreter), 6 (flag gate), 7 (resolver), 2 (sprite roles) |
| §2 MVC split | Task 2 (sprite placement) + Tasks 4-9 (scripts by role) |
| §2 central loop | Task 5 (CB2, S2, C1) + Task 4 (S3, V1) |
| §2 broadcast-and-wait race fix | Task 4 (S3) |
| §3 variables (all 6) | Task 2 |
| §3 scratchpad `next_id` | Task 5 |
| §3 lists (all 8) | Task 2 |
| §3 14/15-scene table with exact content | Task 3 |
| §3 parallel-list invariant | Task 3 (via CB1) |
| §3 numeric flag ids | Task 6 (CB3) |
| §4 S1 initialize | Task 3 |
| §4 S2 × 3 input hats | Task 5 |
| §4 S3 render dispatcher | Task 4 |
| §4 V1 render_text | Task 4 (basic) + Task 6 (C gate) |
| §4 V2 click routing | Task 8 |
| §4 C1 keyboard | Task 5 |
| §4 R1 beetle reactor | Task 9 |
| §4 CB1 add_scene | Task 3 |
| §4 CB2 apply_choice (locked contract) | Task 5 |
| §4 CB3 is_flag_set | Task 6 |
| §4 CB4 run_scene_side_effects | Task 5 (flag sets + defensive endings) + Task 7 (CB5 calls) |
| §4 CB5 resolve_ending | Task 7 |
| §5 E1/E2/E3 guards | Task 5 (E2, E3 in CB2; ready in C1) + Task 8 (ready in V2) |
| §5 invariants | Task 3 (list lengths via CB1), Task 5 (current_scene/ending_code in CB2), Task 7 (ending_code in CB5) |
| §6 test matrix | Task 10 (full run + results recorded) |
| §7 repo layout | Existing + Task 2 + Task 11 |
| §7 .sb3 problem + transcripts | Every task with sb3 changes also updates transcripts |
| §7 commit template | Every commit in every task |
| §7 comment policy | Tasks 3-9 (inline) + Task 10 (audit sweep) |
| §8 assignment requirements cross-check | Met by the above tasks collectively; README Status section (Task 11) is the public record |
| §9 out-of-scope | Observed — no save/load, accessibility, i18n, time_remaining, or runtime validation anywhere in the plan |
| §10 review trail | Existing (in spec) |

**No gaps.** Every spec requirement is mapped to at least one task. Several tasks (10, 11) exist purely to consolidate and verify rather than add logic — an intentional choice for reviewability.

---

## Self-Review (ran during plan writing)

**Placeholder scan:** ✓ No TBD / TODO / "implement later" strings. Every task step contains the actual blocks the executor needs to place. Scene-table content is spelled out fully in Task 3.

**Type / name consistency:**
- Variables named identically in every task: `current_scene`, `food_carried`, `scout_trail_known`, `has_orders`, `ending_code`, `ready`, `next_id`. ✓
- Lists named identically: `scene_text`, `scene_choice_a`, `scene_choice_b`, `scene_next_a`, `scene_next_b`, `scene_flag_required`, `scene_choice_c`, `scene_next_c`. ✓
- Custom blocks: CB1 `add_scene`, CB2 `apply_choice`, CB3 `is_flag_set`, CB4 `run_scene_side_effects`, CB5 `resolve_ending`. ✓
- Broadcasts: `render_scene`, `render_text`, `update_reactors`, `choose_a`, `choose_b`, `choose_c`. ✓
- Scene id assignment: 1-15 as documented; no 9b/9c labels in code. ✓

**Commit discipline:** Every task provides a ready-to-paste commit with the template's five mandatory sections. ✓

**Reviewability:** Every task produces an independently reviewable commit that either (a) adds a bounded chunk of logic with a verifiable spec path, or (b) polishes documentation with a verifiable audit. ✓

**Single responsibility per file:** `stage.md` owns Model + interpreter; each sprite transcript owns that sprite's scripts. No cross-file state. ✓

Plan is ready to execute.
