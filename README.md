# Depot Startup MRP

A standalone tool to plan Day One stockholding for a new bus depot, scaling historical consumption from group-wide transaction history to a user-defined fleet mix and age profile.

Companion to the existing MRP system but answering a different question — not *"what should I reorder for an established depot"* but *"what should I hold on Day One for a depot that hasn't run yet, and what's the steady-state MIN/MAX once it's bedded in"*.

## Inputs

- **Fleet builder (in-app)** — bus type, quantity, average age, entered per row
- **Lead time** — single global value, default 5 days (configurable 4–7)
- **HxGN transaction export** — issues only, group-level, used as the source consumption dataset
- **Fleet reference file** — fleet number → model mapping, used both for direct lookup and to build the 3-digit prefix fallback

## Outputs

A single Excel workbook with three sheets:

1. **Day One Pack** — what to hold on opening day
2. **Steady-State MIN/MAX** — recommended ongoing replenishment levels
3. **Confidence Flags** — parts with sparse source data that need manual review

## Approach

- Read `METHODOLOGY.md` for the data processing rules (transaction filtering, prefix fallback, demand rate calculation, age weighting)
- Read `SPEC.md` for the UI and output requirements
- Read `CLAUDE.md` for project conventions and validation discipline

## Status

Greenfield. Validated transaction-handling logic is inherited from the parent MRP tool's single-part pilot and confirmed against the wireless bell push (part 265526) cross-model usage fingerprint.
