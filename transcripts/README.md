# Transcripts

These files are the **human-readable source of truth** for every script in the Scratch project. The `.sb3` file is the authoritative runtime; these transcripts are what you read and review.

Every transcript follows the same shape:

```
Script / Custom Block: <name>
Purpose: <what this does and why it exists>
Contract (if applicable): <ordering, pre/postconditions, invariants>

Blocks:
  <pseudocode rendering of the actual Scratch blocks, using Scratch's block
  names but typeset for readability>
  // inline comments match the comments attached to the corresponding blocks
  // inside Scratch itself
```

When reviewing a change to the project: the commit touches BOTH the `.sb3` and the relevant transcript(s). If they disagree, the transcript is wrong and needs updating — but the reviewer has enough signal to ask.

## File map

| File | Covers |
|---|---|
| `stage.md` | Stage scripts, variables, lists (the Model + interpreter) |
| `narrator-scroll.md` | Narrator Scroll sprite scripts (the View) |
| `worker-ant.md` | Worker Ant sprite scripts (the Controller) |
| `beetle.md` | Beetle sprite scripts (the Reactor) |

These files will be populated during the implementation phase; the implementation plan ( `docs/superpowers/plans/`) specifies what each should contain.
