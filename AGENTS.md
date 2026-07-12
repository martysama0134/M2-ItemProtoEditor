# AGENTS.md

Read `README.md` before working in this repo — especially the
[Modifying the editor](README.md#modifying-the-editor) section for **any**
change to item proto structs, types, fields, flags, or locales. It lists the
exact tables to touch and the cookbook for each kind of change.

Hard invariants:

1. **Never edit `support.js`.** It is a generated dc-runtime bundle; the whole
   app lives in `index.html`.
2. **`COLS` and `HEADERS` must stay index-aligned.** A new field also needs
   `DEFAULTS`, `FIELD`, and a `GROUPS` entry — five places total (see
   README § Add a proto column). New columns only at index ≥ 2; columns 0–1
   (`vnum`, `nameK`) are special-cased in parse/export.
3. **No build step.** Verify changes by opening `index.html` in a browser and
   clicking **Sample**, then Export and inspect the output columns.
