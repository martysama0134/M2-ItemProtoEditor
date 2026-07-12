# M2 Item Proto Editor

A browser editor for Metin2 `item_proto.txt` and `item_names.txt`.
No build step, no server — open `index.html` in a browser and you are running
it (all app source is in that one file; the first load fetches React from a
CDN, so it needs internet once).

- [Using the editor](#using-the-editor)
- [File format](#file-format)
- [Modifying the editor](#modifying-the-editor) — how to add/change struct
  columns, item types, applies, flags, wear slots, limits, and locales
  (**agents: read this section before touching `index.html`**)

## Using the editor

### Loading files

- **↑ Load proto** — load an `item_proto.txt` (tab-separated text format, first
  line is the header row).
- **↑ Load names** — load an `item_names.txt` (`VNUM<TAB>LOCALE_NAME`, first
  line header). Names are matched to proto rows by vnum. Re-loading overlays
  matching vnums; it does not clear names for vnums missing from the new file.
- **Sample** — restore the built-in demo items (useful for testing changes to
  the editor itself).

### Browsing and editing

- Search accepts a vnum (`4012`), a vnum range (`100-200`), or a name substring
  (matches both LOCALE_NAME and NAME(K)).
- Filter by item type with the dropdown; **⌀ untranslated** shows only rows
  with an empty LOCALE_NAME.
- Click a row to open the edit panel. Fields are grouped (Identity, Economy,
  Requirements, Bonuses, Values, Flags & Wear, Misc). Enum fields are
  dropdowns, flag fields are toggle chips, everything else is a text/number
  input.
- **+ New** creates an item at `max(vnum)+1`. **Duplicate** clones the open
  item to a new vnum. **Delete** removes it.

### Names, charsets, and byte preservation

Metin2 protos are legacy single-byte/EUC-KR encoded, so the editor works at
the byte level:

- **LOCALE_NAME charset** selects the codepage used to *decode* loaded
  `item_names.txt` bytes and to drive the ⚠ encodability warnings
  (CP1250/1252/1253/1254/1256, or UTF-8 = never warn). It does **not**
  re-encode text on export — see the caveat below.
- **NAME(K)** is the throwaway Korean source-name column; its decode/warning
  charset toggles independently between CP949 and CP1252.
- **Byte preservation:** a name you never edit is exported with its *original
  bytes*, untouched. Only rows whose name you actually edited are re-encoded.
  (This covers the name payload bytes — the files themselves are still
  rebuilt: header regenerated, LF line endings, fields trimmed, columns
  beyond the known set dropped, names without a matching proto row omitted.)
- A **⚠** marker means the current text contains characters not encodable in
  the selected codepage.
- **Export encoding caveat:** edited text is written byte-wise as
  ASCII → 1 byte, U+0080–U+00FF → the raw low byte (correct for CP1252
  accents, *not* re-mapped for other codepages), U+0100 and above → UTF-8
  bytes (counted in the export toast). There is no real CP949/CP1250/…
  encoder — for those codepages keep edited names ASCII, or rely on byte
  preservation and fix non-ASCII names outside the editor.

### Exporting

**↓ Export** downloads both files: `item_proto.txt` (all rows, all columns)
and `item_names.txt` (only rows that have a LOCALE_NAME).

## File format

`item_proto.txt` is tab-separated with one header line. Columns are
**positional** — the editor reads and writes them strictly in this order:

| # | Header | Internal key | Editor field |
|---|--------|--------------|--------------|
| 0 | ITEM_VNUM~RANGE | `vnum` | number |
| 1 | ITEM_NAME(K) | `nameK` | text (byte-backed) |
| 2 | ITEM_TYPE | `type` | enum (`ITEMTYPES`) |
| 3 | SUB_TYPE | `subType` | text |
| 4 | SIZE | `size` | number |
| 5 | ANTI_FLAG | `antiFlag` | flags (`ANTI`) |
| 6 | FLAG | `flag` | flags (`FLAGS`) |
| 7 | ITEM_WEAR | `wear` | enum (`WEAR`) |
| 8 | IMMUNE | `immune` | text |
| 9 | GOLD | `gold` | number |
| 10 | SHOP_BUY_PRICE | `shopBuy` | number |
| 11 | REFINE | `refine` | number |
| 12 | REFINESET | `refineSet` | number |
| 13 | MAGIC_PCT | `magicPct` | number |
| 14–17 | LIMIT_TYPE0/VALUE0, LIMIT_TYPE1/VALUE1 | `limitType0`… | enum (`LIMIT`) + number |
| 18–23 | ADDON_TYPE0–2 / ADDON_VALUE0–2 | `addonType0`… | enum (`APPLY`) + number |
| 24–29 | VALUE0–VALUE5 | `value0`–`value5` | number |
| 30 | Specular | `specular` | number |
| 31 | SOCKET | `socket` | number |
| 32 | ATTU_ADDON | `attuAddon` | number |

`item_names.txt` is `VNUM<TAB>LOCALE_NAME` with a header line.

Flag columns hold `|`-separated names (`ANTI_DROP | ANTI_SELL`). Scalar
values are kept as strings internally; unedited names are kept as byte arrays
(`bytes` / `localeBytes`) with `nameK`/`locale` set to `null`.

## Modifying the editor

### Architecture

The entire app is `index.html`:

- The `<x-dc>` block is the HTML template (dc-runtime templating: `{{ expr }}`
  interpolation, `<sc-for>` loops, `<sc-if>` conditionals).
- The `<script type="text/x-dc" data-dc-script>` block holds
  `class Component extends DCLogic` — all state, parsing, export, and the
  static tables that define the item proto schema.

**`support.js` is a generated runtime bundle (dc-runtime). Never edit it.**

There is no build step. Edit `index.html`, reload the browser, click
**Sample** to test.

### Where everything is defined

All schema knowledge lives in static tables on `Component` in `index.html`:

| Table | What it defines |
|-------|-----------------|
| `COLS` | Internal field keys; **array order = column order in item_proto.txt** |
| `HEADERS` | Header row written on export; **must stay index-aligned with `COLS`** |
| `DEFAULTS` | Default value per field for new/parsed rows |
| `FIELD` | Per-field editor metadata: `label`, `kind`, `options`, `span` |
| `GROUPS` | Edit-panel sections; each lists the field keys it shows |
| `ITEMTYPES` | Options for the `type` enum |
| `LIMIT` | Options for `limitType0/1` |
| `WEAR` | Options for `wear` |
| `APPLY` | Options for `addonType0/1/2` (bonuses) |
| `ANTI`, `FLAGS` | Chip options for the two flag fields |
| `LOCALES` | `[code, label, codepage]` triples for the charset dropdown |
| `EXTRA` | Per-codepage string of encodable non-ASCII characters |
| `SAMPLE` | Built-in demo rows (they inherit `DEFAULTS` for missing keys) |

Field metadata in `FIELD` — `kind` is one of:

- `num` — text input with numeric input mode
- `text` — plain text input
- `enum` — dropdown; needs `options: Component.SOMETABLE`
- `flags` — toggle chips joined as `A | B | C`; needs `options`

Other keys: `label` is the panel caption; `span: 2` makes the field take the
full panel width.

### Add a proto column (new field)

Five places must change, and **`COLS`/`HEADERS` must get the new entry at the
same index**. Parsing and export are positional, so put the column where your
server's file actually has it — for a new trailing column, append to both.

Example — add a trailing `MASK_VNUM` column:

1. `COLS` — append `'maskVnum'`
2. `HEADERS` — append `'MASK_VNUM'`
3. `DEFAULTS` — add `maskVnum:'0'`
4. `FIELD` — add `maskVnum:{label:'Mask vnum',kind:'num'}`
5. `GROUPS` — add `'maskVnum'` to a group's `keys` (e.g. Misc), or it won't
   appear in the edit panel

Notes:

- Columns 0 (`vnum`) and 1 (`nameK`) are read by fixed position in
  `parseProto()`, and `nameK` is special-cased (raw bytes) in `export()`.
  Insert new columns only at index ≥ 2.
- Rows with missing *trailing* columns load fine — those fields get
  `DEFAULTS` (a row still needs a vnum plus at least one more field).
  Interior columns cannot be omitted; tabs are positional. Export always
  writes every column in `COLS`; extra input columns beyond it are dropped.
- `SAMPLE` does not need updating; sample rows inherit `DEFAULTS`.

### Add an item type

Append the name to `ITEMTYPES` (e.g. `'ITEM_AURA'`). It appears in the
edit-panel type dropdown immediately; the list's type filter only offers
types present in the loaded rows. Optional: give it a list-badge color in
`typeColor()`.

### Add an apply / bonus type

Append to `APPLY` (e.g. `'APPLY_SUNGMA_STR'`). Shows up in all three Addon
dropdowns. Use the exact enum name your server source uses — the text proto
stores the name, not the number.

### Add a flag, anti-flag, wear slot, or limit type

Append to `FLAGS`, `ANTI`, `WEAR`, or `LIMIT` respectively. Flag chips render
automatically; toggled flags are exported `|`-joined in the order of the
options array.

### Add a locale / codepage

1. Add `['xx','Language (CPnnnn)','nnnn']` to `LOCALES`.
2. If the codepage is new to the editor:
   - add an `EXTRA['nnnn']` string listing every encodable non-ASCII character
     (used for the ⚠ encodability check),
   - add `'nnnn':'windows-nnnn'` to the map inside `decodeName()`.
3. Codepages whose repertoire can't be listed as a string (CJK etc.) get range
   logic in `encChar()` instead — see the CP949 and CP1256 branches there.

These steps add decoding and ⚠ warnings for the new codepage. Export still
writes low-byte/UTF-8 as described in the encoding caveat above — a true
encoder for the codepage would also require changing `_encField()`.

### Gotchas

- **Index alignment.** A `COLS`/`HEADERS` mismatch mislabels that column and
  every one after it in the exported header. Count twice — there are 33.
- **Flag chips rebuild the value.** `toggleFlag()` regenerates the column from
  the known options only — unknown/custom flag names (and the placeholder
  `NONE` default) are silently dropped on the first toggle. Add custom flags
  to the options array before using them.
- **Byte-backed names.** `nameK` and `locale` keep original file bytes
  (`bytes` / `localeBytes`) until edited; editing sets the string and nulls
  the bytes for that row (see `update()`). Don't "normalize" this — it is the
  no-corruption guarantee.
- **Scalars are strings.** Row values are strings even for numeric fields
  (comparisons/parsing use `parseInt` where needed); byte-backed names are
  `null` plus a byte array instead.
- **Static init order.** `FIELD` references `Component.ITEMTYPES` etc. at
  class-init time — option arrays must be declared before `FIELD`.
- **`support.js` is generated.** Repeat: never edit it.

### Testing a change

1. Open `index.html` in a browser (or reload).
2. Click **Sample**, open an item, check your field/option renders and edits.
3. **Export** and inspect the downloaded files — header count, column count,
   and tab positions must match your server's expectations.
4. Load a real `item_proto.txt`, spot-check a few rows, re-export, diff.
5. For encoding-sensitive changes, hex-compare unedited non-ASCII names
   across a load → export round trip (byte preservation must hold).
