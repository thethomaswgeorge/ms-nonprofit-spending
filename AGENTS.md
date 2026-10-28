# AGENTS.md

Guidance for AI coding agents working in this repository. Human-facing overview: [README.md](README.md).

## What this is

A static audit dashboard for the Mississippi Office of the State Auditor's Non-Profit Spending data
(FY2021–FY2026, 36,326 payment lines, $4,226,181,988.29). It is published with GitHub Pages from
`main`, `/ (root)`. There is no build step, package manager, framework or test suite.

| Path | Role |
| --- | --- |
| `index.html` | The whole app: CSS, an embedded JSON dataset, and one vanilla-JS IIFE. About 2 MB. |
| `data/ms_nonprofit_spending_fy2021_2026.csv` | The same dataset as a flat CSV for download and analysis. |
| `.nojekyll` | Stops GitHub Pages from running Jekyll. Keep it. |

## The data

### CSV columns

`fiscal_year, budget_period, agency, ein, vendor_number, nonprofit, state, expense_item, expense_type, amount`

- One row is one payment line as recorded by the source. No two rows share all nine non-amount fields.
- `budget_period` is `YYYY-A1` or `YYYY-A2` and always starts with `fiscal_year`.
- `ein` is digits only, with no dash. 413 lines carry 8-digit EINs because the source stored EINs as numbers
  and lost the leading zero. Keep them as recorded. The dashboard flags them and pads them only for display.
- `nonprofit` is the payee name exactly as recorded, including truncation at 35 characters, typos and
  double spaces. Do not clean or normalise names: name drift is one of the things the dashboard surfaces.
- `amount` is in dollars, non-negative, with at most two decimals and no trailing zeros
  (`15000000`, `108647280.84`, `12.5`). 68 lines are exactly `0`.

### Embedded JSON (`<script id="data" type="application/json">` in `index.html`)

Dictionary-encoded to keep the file small:

- `bp`, `ag`, `np`, `st`, `ei`, `et` are sorted lookup arrays (budget periods, agencies, payee names, states,
  expense items, expense types).
- `rows` holds tuples `[fy, bp, ag, ein, vn, np, st, ei, et, amt]`. Positions 1, 2, 5, 6, 7 and 8 are
  indexes into the lookup arrays. `ein` and `vn` (vendor number) are raw integers.
- `person` lists the EINs of payees whose names read as individuals. It is curated by hand.
- `asof` is the data currency date shown in the header.

### Invariant: the CSV and the JSON must agree

Decoded, the embedded rows and the CSV rows must be the same multiset, and both must total
$4,226,181,988.29. Nothing generates one from the other, so **any data change has to be made in both
places**. Run this check after touching either:

```bash
python3 - <<'EOF'
import csv, json, re, collections as C
from decimal import Decimal
h = open('index.html').read()
d = json.loads(re.search(r'<script id="data" type="application/json">(.*?)</script>', h, re.S).group(1))
dec = lambda r: (str(r[0]), d['bp'][r[1]], d['ag'][r[2]], str(r[3]), str(r[4]), d['np'][r[5]],
                 d['st'][r[6]], d['ei'][r[7]], d['et'][r[8]], Decimal(str(r[9])))
A = C.Counter(dec(r) for r in d['rows'])
B = C.Counter(tuple(r.values())[:9] + (Decimal(r['amount']),)
              for r in csv.DictReader(open('data/ms_nonprofit_spending_fy2021_2026.csv')))
print('rows', sum(A.values()), sum(B.values()), '| match', A == B,
      '| total', sum(k[9] * v for k, v in B.items()))
EOF
```

Expected output: `rows 36326 36326 | match True | total 4226181988.29`.

When you edit the CSV, change only the lines you mean to change. Python's `csv` writer does not
reproduce the file byte for byte (quoting and number formatting differ), so check `git diff --stat` before
committing.

## The app (`index.html` script)

Everything runs in `run()`: `filterLines()` → `aggregate()` → `renderTiles()`, `renderCharts()`,
`renderSignals()`, `renderLedger()`, `renderLines()`. The state object `S` holds the filters (years, agency,
state, minimum line amount, search text, active signal), the sort orders and the page numbers.

- `aggregate()` builds one entity per EIN from the **filtered** lines and sets `e.flags` for each signal.
- `SIG` defines the 16 signals: key, severity (`crit`, `warn`, `info`), title, description `d` and method
  text `m`. The page shows `m` under "How the signals are defined", so **when you change a signal's logic,
  update its `m` text too.**
- Name-pattern signals use `RX_EXCL` (excluded public institutions, with affiliated foundations and
  associations excluded by `RX_AFFIL`), `RX_GOV` (government bodies, county hospitals, utilities, co-ops,
  credit unions) and `RX_TRUNC` (truncated names). These are heuristics. Before you widen one, list the payee
  names it newly matches and read them: broad terms such as `COUNTY` catch many ordinary non-profits.
- `HIST` holds each EIN's per-year totals over **all** lines. The single-year and year-over-year spike signals
  read `HIST`, not the filtered totals, so filters do not change them.
- The dossier (`openDossier`) always uses every line for the EIN and ignores filters.

## Conventions

- Keep it a single, dependency-free file. The only external request is Google Fonts.
- Match the existing dense, compact JS style: short names, minimal comments, no frameworks.
- Colours are CSS custom properties on `:root`, with dark-mode overrides under `prefers-color-scheme`
  and `[data-theme]`. Add new colours as tokens in all three blocks.
- Signals are tests on recorded data, not findings. Keep that framing in any text you add.

## Checking changes

- Syntax-check the script:
  `python3 -c "h=open('index.html').read();open('/tmp/app.js','w').write(h[h.rfind('<script>')+8:h.rfind('</script>')])" && node --check /tmp/app.js`
- Serve the folder and open it in a browser, since `file://` may be blocked in some environments:
  `python3 -m http.server 8000`, then visit `http://localhost:8000/`.
- Try a single-year filter, an agency filter, a search and a signal card, and confirm the tiles, charts,
  ledger and lines stay consistent.
