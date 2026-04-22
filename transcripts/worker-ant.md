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
