# Worker Ant — Controller

Role: the Controller in the MVC split. Listens for player keyboard input and emits `choose_a/b/c` broadcasts. Does not apply the choice itself; that's the Stage's job (CB2 `apply_choice`).

The Worker Ant sprite is the protagonist visible during play but holds no story state. It is deliberately minimal.

Costume: `project/assets/worker-ant.svg`.

---

## Scripts

| Section | Name | Status |
|---|---|---|
| C1 | `when [1] key pressed` → broadcast `choose_a` | Implemented |
| C1 | `when [2] key pressed` → broadcast `choose_b` | Implemented |
| C1 | `when [3] key pressed` → broadcast `choose_c` | Implemented |

All three include a `ready = 1` guard.

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
    broadcast [choose_b v]                        // S2b on Stage
  end

when [3 v] key pressed
  if <(ready) = (1)> then
    broadcast [choose_c v]                        // S2c on Stage
  end
```

**Verification (Task 5):**
- Green flag + press `1` → advances to scene 2; `has_orders` becomes 1. ✓
- From scene 2, pressing `2` or `3` is a no-op (E2). ✓
- Keys pressed during S1's initial render are ignored (`ready=0`). ✓
