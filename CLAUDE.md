# Claude Code Project Context

This is a greenfield build. Before writing any code, read these files in order:

1. `README.md` — project purpose and high-level shape
2. `SPEC.md` — functional requirements
3. `METHODOLOGY.md` — HxGN data processing rules **(this is the most important file — it encodes validated transaction logic)**

## Project conventions

- **Single self-contained HTML file** — all logic, UI, and processing runs in the browser with no backend server required
- All data processing (transaction filtering, fleet resolution, demand rates, age weighting) implemented in JavaScript within the HTML file
- Excel output generated client-side (e.g. using SheetJS / xlsx.js)
- All ingestion logic must follow `METHODOLOGY.md` exactly. The transaction filtering rules were validated against a real pilot. Do not silently change them.

## Validation discipline

Before adding any new processing rule, validate it against the **wireless bell push (part 265526)**. Expected fingerprint from real data:

- Heavy use on Wrightbus Streetdeck MH2 (~1.0 per bus / yr)
- Moderate use on Streetdeck MH3, HEV96, Optare Versa, Optare Solo SR
- **Negligible** use on Volvo B9TL (~0.02 per bus / yr — older buses use the wired version, not this part)

If a code change disrupts this fingerprint, the change is wrong, not the data.

## What's known to be tricky

- **HxGN lead times are broken** — do NOT pull lead times from the data. Use the user-entered value only.
- Some transactions have no `FleetNumber` (direct-to-bus or non-bus). Exclude these from per-bus rate calculations.
- Some `FleetNumber`s in the transaction file won't match the fleet reference (out-of-region buses). Use the **3-digit prefix fallback** — see `METHODOLOGY.md`.
- The 3-digit prefix rule is mostly reliable but has 16 documented exceptions. Handle them explicitly.
- **Negative `TransactionQty` rows are STTK adjustments** — include their absolute value in demand. They represent unrecorded consumption found at stocktake.

## Out of scope (for v1)

- Per-supplier or per-part lead times
- Authoritative parts ↔ bus-type catalogue (this will come from a separate "bus breakdown" project later)
- Anything beyond a one-month Day One horizon
