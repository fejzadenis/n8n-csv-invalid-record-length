# Invalid Record Length: columns length is 7, got 8 — n8n CSV fix

The row is not corrupt and no data is lost. The parser read the header, decided the file has 7 columns, then reached a row that splits into 8 fields on the delimiter you selected — so it refused that row instead of guessing.

This repo contains the free diagnosis path and a copy-paste Code node that repairs the rows before they reach the CSV node. No upload, no account, no dependency.

## What the numbers mean

| Message | What it actually says |
|---|---|
| `columns length is 7, got 8` | One field too many — a trailing delimiter, or an unquoted value containing the delimiter |
| `columns length is 8, got 7` | One field too few — the exporter omitted the last empty cell, or a value was lost upstream |
| `got 1` | The delimiter is wrong: the file is semicolon-separated and the node is set to comma, so every row is one field |
| `got 13` when the header has 7 | A field contains the delimiter and is not quoted, so one cell got split into several |

## Which row is it? Twenty seconds

The number after `on line` counts the header as line 1. Open the CSV in a plain text editor with line numbers (VS Code, Notepad++) and look at that line and the one before it:

- Line ends with the delimiter (`...;R;` when the header has 7 fields) → trailing empty cell from a spreadsheet export. Most common cause by a wide margin.
- Previous line has an unclosed quote → the parser is still inside that quoted field when it hits the next line; everything after shifts.
- Line ends mid-address → an unquoted line break inside a field.

Excel opens the file because Excel is lenient: it pads short rows, tolerates a trailing delimiter, and re-guesses the separator. The n8n `Extract from File` node uses the delimiter you selected and refuses any row whose field count differs from the header. Both observations are true at once.

## Fix 1 — two settings that turn a hard stop into a diagnosis

1. **Set the delimiter explicitly.** A file exported on a European Windows is usually `;` while the node defaults to `,`. If the error said `got 1`, this is the whole problem.
2. **Turn `Skip records with errors` off while diagnosing.** With it on, the node drops every affected row and reports nothing — a 4,000-row import quietly becomes 3,880 rows.

## Fix 2 — a Code node that pads instead of failing

Use this when you cannot control the export (a supplier, a client, a nightly FTP drop). Run Once for All Items, input one item holding the file as text.

```js
// Code node, mode: Run Once for All Items
// input: one item { csv: "<file contents as text>" }
const DELIM = ';';
const text = $input.first().json.csv.replace(/^\uFEFF/, '');
const lines = text.split(/\r?\n/).filter(l => l.trim() !== '');

const header = lines.shift().split(DELIM).map(h => h.trim() || 'unnamed');
const rows = lines.map((line, i) => {
  const cells = line.split(DELIM);
  // a trailing delimiter produces empty overflow cells - drop them
  while (cells.length > header.length && cells[cells.length - 1].trim() === '') cells.pop();
  // a short row gets the missing cells as empty strings
  while (cells.length < header.length) cells.push('');
  const out = { __source_line: i + 2 };
  header.forEach((h, j) => { out[h] = cells[j] === undefined ? '' : cells[j].trim(); });
  return { json: out };
});
return rows;
```

Two honest limits that decide whether this is enough:

- It does not understand quoted delimiters. A cell containing `;` inside quotes still breaks. If your file has quoted separators, a real parser has to do the work.
- It repairs shape, not meaning. A row that lost a value upstream stays wrong; `__source_line` is there so you can go back and check those rows in the original file.

## Fix 3 — repair it in the browser, free

[Free CSV repair](https://fejzadenis.github.io/leadflow/free) removes trailing delimiters, re-pads short rows to the header, normalises quoting, dedupes by email and repairs wrongly decoded characters (question marks, `Ã©`, UTF-8 read as GB18030). It hands back the fixed file plus a report of every row it changed. The file is read with the browser's own file API and repaired in the tab: no upload, no account, no analytics.

## If you would rather hand it to a person

One row class, one file: a human opens the CSV, repairs it and sends it back with a report naming every dropped row and every value rewritten. **$49 for up to 5,000 rows, back within 24 hours, 14-day refund** if the delivered file does not match that sentence. Same person, same fix, when the damage is wider or the file is bigger — [the full set of 15 import failures and the repair offer](https://fejzadenis.github.io/leadflow/).

Not ready to spend anything: mail the broken file and the first look is free — someone reads it and tells you what is wrong before you decide. perun.intelligence@gmail.com

## Scope note

This was reported to n8n as issue #20524; maintainers routed it as an enhancement rather than a bug, so the node is unlikely to become lenient. Repair the rows yourself rather than waiting.
