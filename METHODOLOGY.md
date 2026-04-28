# Methodology

This file encodes the validated data processing rules. Mirror them exactly. Do not deviate without re-validating against the wireless bell push fingerprint described in `CLAUDE.md`.

---

## 1. Transaction filtering

Source: HxGN EAM transaction export, issues only (`TransactionType = 'I'`).

### Include in demand

- **Genuine bus issues**: `TransactionType = 'I'`, `TransactionQty > 0`, `FleetNumber` populated and resolvable to a model
- **Negative STTK rows**: `TransactionType = 'I'`, `TransactionQty < 0`. These represent unrecorded consumption found at stocktake. Take the **absolute value** and add to demand for the relevant model.

### Exclude from demand

- Issues with no `FleetNumber` (direct-to-bus deliveries or non-bus consumption — can't attribute to a model)
- Issues to fleet numbers whose 3-digit prefix doesn't map to any known bus type (i.e. non-bus support vehicles like vans, forklifts, steam cleaners)

---

## 2. Fleet number → model resolution

Resolve in this order:

1. **Direct lookup** — match `FleetNumber` against the fleet reference file
2. **Prefix fallback** — if no direct match, take the first 3 digits and look up the model from the prefix-to-model mapping

### Building the prefix-to-model mapping

From the fleet reference file, group by 3-digit prefix and identify the most common model for each prefix.

#### Documented exceptions (one prefix → multiple models)

These prefixes resolve to more than one bus model in the fleet reference. Use the most common model for the prefix and log the row as ambiguous in a debug file (don't surface in the user-facing output unless it materially affects results):

| Prefix | Models |
|---|---|
| 351 | Wrightbus Streetdeck MH2 / MH3 |
| 355 | Wrightbus Streetdeck MH2 / MH3 |
| 474 | Wrightbus Streetlite NMH / MH1 |
| 476 | Wrightbus Streetlite MH1 / MH2 |
| 538 | Optare Solo / Solo SR |
| 631 | Wrightbus Streetlite NMH / MH1 |
| 632 | Wrightbus Streetlite MH1 / MH2 |
| 686 | BMC 220 SLF / 225 SLF |

The exceptions are mostly close variants of the same chassis, so misclassification within a prefix has limited impact on demand rate.

#### Non-bus prefixes — exclude entirely

Prefixes 905, 907, 908, 909, 930, 932, 933, 934, 935, 936, 940, 960, 962, 965, 970, 971, 990 are vans, forklifts, steam cleaners, and EVs — not buses. Exclude transactions on these from per-bus demand calculations.

---

## 3. Per-bus-per-year demand

For each (Part, Model) pair:

```
demand_per_bus_per_year = total_qty_issued_to_model / active_population_for_model
```

Where:

- `total_qty_issued_to_model` — sum of TransactionQty (positive issues + absolute value of negative STTKs) across all transactions resolved to that model, **for this part**
- `active_population_for_model` — count of distinct fleet numbers of that model appearing **anywhere in the transaction file** (across all parts), resolved via direct lookup or prefix fallback

### Why "active population in transactions" not "fleet reference count"?

The fleet reference is regional and incomplete. The transaction history is group-wide. Using the regional fleet count as the denominator would over-state the per-bus rate by treating group-wide consumption as if it came from a smaller fleet.

Counting distinct fleet numbers actually appearing in the transactions gives a true group-wide active-population proxy.

### Low-confidence flag

If a model has fewer than **3** distinct fleet numbers in the transaction history, flag any per-bus rate derived from it as low-confidence. The rate is too volatile to be reliable.

---

## 4. Scaling to the user's fleet

For each (Part, Model) pair where the user is operating that model:

```
annual_demand_for_part_from_model = (
    demand_per_bus_per_year
    × user_fleet_quantity_for_model
    × age_weighting(material_group, avg_age)
)
```

Then sum across all models the user operates to get total annual demand for the part:

```
total_annual_demand_for_part = sum(annual_demand_for_part_from_model) for all models
```

---

## 5. Age weighting

A multiplier applied to per-bus demand to reflect that the source transactions come from a mixed-age fleet, while the user's depot may be newer or older.

### Material group classification (initial heuristic)

- **Consumables** — filters, brake pads, wiper blades, bulbs, fuses, bell pushes, etc. Minimal age effect, weighting = 1.0 across all ages.
- **Wear items** — clutches, alternators, body panels, radiators, etc. Heavy age effect, see curve below.
- **Scheduled service** — oils, coolants, scheduled parts. Minimal age effect, weighting = 1.0.

Classification can be inferred from `MtlGroupDesc` — keep the mapping in a config file so it can be tuned without code changes.

### Wear-item age weighting curve (linear, tunable)

| Avg Fleet Age | Weighting |
|---|---|
| 0–2 yrs | 0.5 |
| 3–5 yrs | 0.8 |
| 6–9 yrs | 1.0 (matches the source-data baseline) |
| 10–14 yrs | 1.3 |
| 15+ yrs | 1.6 |

These values are **starting estimates only**. Surface them in a config file. Do not hard-code in the calculation logic.

---

## 6. Day One holding calculation

For each part with at least one model contributing to the user's fleet:

```
day_one_holding = ceil(
    monthly_demand                   # = total_annual_demand_for_part / 12
    + lead_time_safety_buffer        # = (lead_time_days / 365) × total_annual_demand
    + launch_contingency             # = 0.5 × monthly_demand
)
```

- **Monthly demand** — covers the first month before reorder cycles stabilise
- **Lead-time safety buffer** — covers expected consumption during one supplier lead time
- **Launch contingency (50%)** — extra cushion for the early weeks where you have no live consumption signal to react to

Round up to the next whole unit.

---

## 7. Steady-state MIN/MAX

Mirror the parent MRP tool's validated logic. **Lift the existing implementation rather than reinventing it.** Confirm the source repo location with the user before coding.

---

## 8. Confidence column

For each part recommended in either Day One or Steady-State sheets:

| Confidence | Criteria |
|---|---|
| **High** | 10+ issues across the source data, AND present in 2+ models matching the user's fleet, AND every contributing model has 3+ active fleet numbers |
| **Medium** | 3–9 issues, OR present in only 1 model matching the user's fleet, OR contributing models have between 3 and 5 active fleet numbers |
| **Low** | Fewer than 3 issues, OR derived only from low-confidence model rates (<3 fleet numbers) |

Always include low-confidence parts in the **Confidence Flags** sheet so they get manual review — never silently drop them.

---

## 9. Validation gate

After any change to the calculation pipeline, regenerate the per-model demand table for the wireless bell push (part 265526) and verify:

- Streetdeck MH2 rate ≈ 1.0 per bus / yr
- Streetdeck MH3 rate ≈ 0.4 per bus / yr
- Volvo B9TL rate < 0.05 per bus / yr
- Cross-model spread is preserved

If these don't hold, stop and investigate before continuing.
