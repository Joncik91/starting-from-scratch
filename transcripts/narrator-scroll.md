# Narrator Scroll — View

Role: the View in the MVC split. Reads `current_scene` and the scene-table lists on the Stage and renders the current scene's text + choice labels. Also routes clicks on the scroll into `choose_a/b/c` broadcasts (shared downstream with the Worker Ant's keyboard input).

Costume: `project/assets/narrator-scroll.svg`.

---

## Scripts

| Section | Name | Status |
|---|---|---|
| V1 | `when I receive render_text` — paint scene | **Task 4** (choice C gate in Task 6) |
| V2 | `when this sprite clicked` — click routing | **Task 8** |
