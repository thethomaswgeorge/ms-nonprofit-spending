# Mississippi Non-Profit Spending Audit

An interactive re-cut of the Mississippi Office of the State Auditor's **Non-Profit Spending** dashboard
(FY2021–FY2026), built to surface inconsistencies and peculiarities the official view hides.

- **`index.html`** — the audit dashboard: filters, charts, sixteen signal tests (name drift, duplicate vendor
  numbers, excluded public institutions, individuals as payees, single-year windfalls, year-over-year spikes,
  expense-coding flips, round-number lump sums, and more), an entity ledger and a per-EIN dossier.
- **`data/ms_nonprofit_spending_fy2021_2026.csv`** — all 36,326 payment lines
  (fiscal year, budget period, agency, EIN, vendor number, payee, state, expense item, expense type, amount).
  Total: $4,226,181,988.29, matching the OSA report to the cent.

## Source

Data was extracted from the OSA Power BI report embedded at
`msi.ms.gov/osa/reports/non-profit-spending`, which the Auditor's office compiles from the
DFA Transparency (MAGIC) system and supplements with IRS records. Figures are current as of October 1, 2026.
The report excludes payments to Mississippi public universities, community colleges, public school districts
and MPACT tuition; several entities matching those exclusions nevertheless appear and are flagged in the dashboard.

## Caveats

Signals are tests on the recorded data, not findings. The "looks like an individual" and
"government or co-op" signals in particular are pattern matches on vendor names and should be verified
against the vendor record before being relied on.

The dashboard is a single file with no build step; open `index.html` directly or serve the folder with any static host.
