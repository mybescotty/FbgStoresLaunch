# Functional Spec

## User journey

1. User lands on the **depot builder** page
2. User adds rows to the fleet table — one row per bus type they're planning to operate
3. User enters lead time (single value, default 5 days)
4. User clicks **Generate Plan**
5. Tool processes the loaded reference data and outputs a downloadable Excel workbook

## Front page (depot builder)

A table with the following columns and an **Add Row** button:

| Column | Type | Notes |
|---|---|---|
| Bus Type | Dropdown | Populated from distinct chassis variants in the fleet reference file |
| Quantity | Integer | How many of this type the depot will operate |
| Average Age | Decimal (years) | Drives the age weighting curve in `METHODOLOGY.md` |

Plus a single field above or below the table:

- **Lead Time (days)** — default 5

And a **Generate Plan** button.

### UX notes

- Bus Type dropdown should be searchable — there are 70+ variants
- Allow rows to be removed individually
- Show a running total ("Total fleet: N buses") below the table
- Validate before submitting: at least one row, all quantities ≥ 1, ages ≥ 0

## Reference data

Reference data is uploaded once (admin function), not per session:

- **Fleet reference file** — used to populate the bus type dropdown and build the prefix-to-model mapping
- **Transaction history** — used as the source consumption dataset

Show the upload date and basic stats (rows, distinct parts, date range) on a small admin page so the user knows how fresh the data is.

## Output workbook

A single Excel workbook with three sheets.

### Sheet 1 — Day One Pack

| Column | Notes |
|---|---|
| Part | Part number |
| Description | Part description |
| Material Group | From HxGN |
| Recommended Holding | Whole units, rounded up |
| Confidence | High / Medium / Low |
| Notes | Free text — e.g. "Used only on 1 model in your fleet", "Sparse history" |

### Sheet 2 — Steady-State MIN/MAX

| Column | Notes |
|---|---|
| Part | Part number |
| Description | Part description |
| Material Group | From HxGN |
| MIN | |
| MAX | |
| Confidence | High / Medium / Low |

MIN/MAX logic should mirror the parent MRP tool's validated approach — confirm with the user before implementing rather than reinventing.

### Sheet 3 — Confidence Flags

Lists every part where source data is too sparse to give a confident number.

| Column | Notes |
|---|---|
| Part | |
| Description | |
| Total Issues in Source | Across all models, all time |
| Models in User Fleet Using This Part | Count |
| Recommendation Quality | Why it's flagged (e.g. "Only 2 issues in source data") |

## Non-functional

- Plan generation should complete within 60 seconds for a typical input (5–10 bus types)
- Excel output must open cleanly in both Excel and LibreOffice
- All count values formatted as integers
- Use a consistent professional font in the output (Arial or similar)
