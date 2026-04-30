# Driver Behaviour Dashboard — Open Issues & Data Gaps

> Living document. Add issues per module as design is reviewed.
> Status: `OPEN` | `NEEDS CLARIFICATION` | `RESOLVED`

---

## CROSS-MODULE (affects all 3 features)

| # | Issue | Status | Notes |
|---|-------|--------|-------|
| C1 | **Fuel Price Constant** — no fuel price stored anywhere in ClickHouse. Needed for: Total Cost Impact, AI Insight costs, Impact column, Est Fuel Savings. | NEEDS CLARIFICATION | Confirm INR/liter value (₹90? configurable per fleet?) |
| C2 | **Idle Time %** in Vehicle Rankings (Engine Braking + Free Running) — not in any of the 3 event tables. | OPEN | Need separate idling data source. Does a ClickHouse idling table exist? Or do we derive from engine cycle gaps? |
| C3 | **Summary label inconsistency** — Summary card says "Last 30 days" but page filter shows "Last 15 days". | NEEDS CLARIFICATION | Which window drives the 3 summary KPIs? Should it always mirror the page date filter? |
| C4 | **Trend % direction convention** — Engine Braking shows -3.5% in green (fewer incidents = good). Clutch Riding + Free Running show +3.5% in green. Confirm: is green always "improvement" (meaning direction of good is inverted per feature)? | NEEDS CLARIFICATION | Define per-feature "good direction" so trend arrow/color logic is consistent |
| C5 | **Rate / Per 1000 Km** denominator — requires total distance driven, not just event distance. Source: SUM of all event_type rows per vehicle (including 'normal_riding', 'normal') from the respective event table. Confirm this is the correct denominator. | OPEN | Alternative: use OBD total odometer. Decide once. |
| C6 | **AI Insight pareto copy** — "9 of 10 Vehicles drive 80% of incidents" is a computed sentence. Need: total fleet vehicle count AND a configurable pareto threshold (80%?) to determine N. | OPEN | Confirm pareto threshold is fixed at 80% or user-adjustable |
| C7 | **Summary Total Events vs module-specific Events** — in Free Running, the KPI section shows Events: 1,234 while Summary shows 12,450. Confirm: Summary = fleet-wide all-features count, or only this module's count? | NEEDS CLARIFICATION | |
| C8 | **DQM-failed rows** — schema has `dqm_status` and `quality_score` on all tables. Confirm: should dashboards exclude `dqm_status = 'FAIL'` rows? Or show all with a data quality warning? | NEEDS CLARIFICATION | |

---

## MODULE 1 — Clutch Riding (`clutch_riding_events_sm`)

### Schema reference
`event_type`: `clutch_riding` | `normal_riding`
Key columns: `duration_sec`, `distance_km`, `avg_speed_kmh`, `avg_rpm`, `severity`, `fuel_consumed_liters`, `mileage_kmpl`, `event_start_epoch`

### Component Mapping

| Component | Field | Source | Status |
|-----------|-------|--------|--------|
| Summary | Total Events | `COUNT(*) WHERE event_type='clutch_riding'` | ✅ Direct |
| Summary | Total Cost Impact | `SUM(fuel_consumed_liters) × fuel_price` | ⚠️ Needs C1 |
| Summary | Trend % | Current vs prior period count ratio | ✅ Derivable |
| AI Insight | Worst Vehicle / Incidents / Cost | GROUP BY uniqueid, ORDER BY count DESC | ✅ Direct |
| AI Insight | Potential Saving | Fleet cost × pareto concentration | ⚠️ Needs C6 |
| KPI Bar | Tot Distance Covered | `SUM(distance_km)` all event_types | ✅ Direct |
| KPI Bar | Clutch riding Distance | `SUM(distance_km) WHERE event_type='clutch_riding'` | ✅ Direct |
| KPI Bar | Overall Mileage (8.4 Kmpl) | `SUM(distance_km) / SUM(fuel_consumed_liters)` all events | ✅ Direct |
| KPI Bar | Clutch riding Mileage (6.4 Kmpl) | Same but WHERE event_type='clutch_riding' | ✅ Direct |
| KPI Bar | Mileage Drop % (28.3%) | `(overall_mileage - clutch_mileage) / overall_mileage × 100` | ✅ Derived |
| KPI Bar | Fuel Lost (48 L) | See issue CR1 below | ⚠️ Open |
| KPI Bar | Cost Impact (₹1,23,456) | `fuel_lost × fuel_price` | ⚠️ Needs CR1 + C1 |
| Chart | Distance Covered Trend | `GROUP BY toDate(toDateTime(event_start_epoch))`: SUM(distance_km) | ✅ Direct |
| Chart | Cost Impact Trend | Daily SUM(fuel) × price | ⚠️ Needs C1 |
| Rankings | Incidents | `COUNT(*) WHERE event_type='clutch_riding'` per vehicle | ✅ Direct |
| Rankings | Rate/1000 Km | `incidents / total_distance_km × 1000` | ⚠️ Needs C5 |
| Rankings | Mileage Drop | Per-vehicle mileage drop % | ✅ Derived |
| Rankings | Impact | Per-vehicle fuel_lost × price | ⚠️ Needs CR1 + C1 |
| Breakdown | Clutch Riding % | `clutch_distance / total_distance × 100` per vehicle | ✅ Derived |
| Breakdown | Overall Mileage | Per-vehicle overall mileage | ✅ Direct |
| Breakdown | Clutch Mileage | Per-vehicle clutch-only mileage | ✅ Direct |
| Breakdown | Mileage Drop % | Per-vehicle drop | ✅ Derived |

### Open Issues

| # | Issue | Status |
|---|-------|--------|
| CR1 | **Fuel Lost formula undefined** — Is it: `(clutch_distance / clutch_mileage) - (clutch_distance / overall_mileage)`? Or total excess fuel vs baseline? The 48L in design needs a precise definition before this can be coded. | NEEDS CLARIFICATION |
| CR2 | **Clutch Riding % denominator** — uses SUM(distance_km) of ALL rows (clutch_riding + normal_riding) per vehicle. Confirm: `normal_riding` events are correctly captured in the same table so total distance is accurate. | OPEN |
| CR3 | **Two trend charts** (Distance + Cost Impact) share the same x-axis date range. Confirm they should always be in sync with the page date filter, with no independent zoom/pan. | NEEDS CLARIFICATION |

---

## MODULE 2 — Free Running / Coasting (`coasting_events_sm`)

### Schema reference
`event_type`: `neutral_coasting` | `clutch_coasting` | `normal`
Key columns: `duration_sec`, `distance_km`, `avg_speed_kmh`, `max_speed_kmh`, `fuel_consumed_liters`, `fuel_from_rate_liters`, `n_packets`, `critical_packets`, `start_lat/lng`, `end_lat/lng`, `start_ts`

### Component Mapping

| Component | Field | Source | Status |
|-----------|-------|--------|--------|
| Summary | Total Events | `COUNT(*) WHERE event_type IN ('neutral_coasting','clutch_coasting')` | ✅ Direct |
| Summary | Total Cost Impact | `SUM(fuel_consumed_liters) × fuel_price` | ⚠️ Needs C1 |
| Summary | Trend % | Period comparison | ✅ Derivable |
| AI Insight | All fields | Same pareto logic as CR | ✅ / ⚠️ |
| KPI | Events: 1,234 | COUNT coasting events | ✅ Direct |
| KPI | Cost Impact | SUM × price | ⚠️ Needs C1 |
| KPI | Per 1000 Km: 12 | `events / total_distance × 1000` | ⚠️ Needs C5 |
| KPI | +6% badge | See issue FR3 | ⚠️ Open |
| Chart | Event Count trend | `GROUP BY toDate(toDateTime(start_ts))`: COUNT | ✅ Direct |
| Rankings | Incidents | COUNT per vehicle | ✅ Direct |
| Rankings | Idle Time % | **NOT IN SCHEMA** | ❌ Needs C2 |
| Rankings | Fuel Wasted | `SUM(fuel_consumed_liters)` coasting events per vehicle | ✅ Direct |
| Rankings | Impact | SUM × price | ⚠️ Needs C1 |
| Breakdown | Free Running (Hrs) | `SUM(duration_sec) / 3600` per vehicle | ✅ Direct |
| Breakdown | Free running Time % | See issue FR1 | ❌ Open |
| Breakdown | Cost Impact | SUM(fuel) × price | ⚠️ Needs C1 |

### Open Issues

| # | Issue | Status |
|---|-------|--------|
| FR1 | **Free running Time %** — requires total engine-on time per vehicle. `coasting_events_sm` does not store this. Options: (a) derive from `engine_braking_cycles_sm.cycle_end_ts - cycle_start_ts` joined on `cycle_id`; (b) query PostgreSQL `public.engineoncycles` at dashboard time; (c) add total_cycle_duration to a summary table. Which approach? | OPEN |
| FR2 | **"Free Running" vs "Coasting"** — Design calls this feature "Free Running" but schema event_type uses `neutral_coasting` and `clutch_coasting`. Confirm: does "Free Running" = neutral_coasting only, or both neutral + clutch coasting combined? This affects all counts and calculations. | NEEDS CLARIFICATION |
| FR3 | **+6% badge** next to "Free Running" section heading — unclear what this % represents. Options: (a) WoW/MOM trend of event count; (b) % of total driving time that is free running; (c) something else. | NEEDS CLARIFICATION |
| FR4 | **`critical_packets`** column (speed increasing during coasting = dangerous) — not surfaced in any Figma component. Should this appear as a separate flag/column in the breakdown or event log? | OPEN — potential future addition |
| FR5 | **GPS columns** (`start_lat/lng`, `end_lat/lng`) in `coasting_events_sm` — not shown in any Figma component. Confirm: is there a map view planned for coasting events? | OPEN — potential future addition |

---

## MODULE 3 — Engine Braking (`engine_braking_events_sm` + `engine_braking_cycles_sm`)

### Schema reference
Events: `event_type`: `engine_braking` | `normal` — with gear, rpm, engine_load, speed range, brake_during_event
Cycles: `engine_braking_pct`, `engine_braking_mileage_kmpl`, `normal_mileage_kmpl`, `overall_distance_km`, etc.

### Component Mapping

| Component | Field | Source | Status |
|-----------|-------|--------|--------|
| Summary | Total Events | `COUNT(*) WHERE event_type='engine_braking'` on events table | ✅ Direct |
| Summary | Total Cost Impact | `SUM(fuel_from_consumption_liters) × fuel_price` | ⚠️ Needs C1 |
| Summary | Trend % | Period comparison | ✅ Derivable |
| AI Insight | All fields | Pareto on events table | ✅ / ⚠️ |
| KPI | Fleet Avg Adoption % | `AVG(engine_braking_pct)` from cycles table | ✅ Direct |
| KPI | Mileage at Current Adpt | `SUM(engine_braking_distance_km) / SUM(engine_braking_consumption_liters)` from cycles | ✅ Direct |
| KPI | Baseline Mileage | `SUM(normal_distance_km) / SUM(normal_consumption_liters)` from cycles | ✅ Direct |
| KPI | Est Fuel Savings | See issue EB1 | ⚠️ Open |
| KPI | Potential Fuel Saving | See issue EB2 | ⚠️ Open |
| KPI | Annual Potential Fuel Savings | Monthly extrapolated × 12 | ⚠️ Needs EB2 |
| Chart | Adoption Trend (%) | `AVG(engine_braking_pct)` grouped by day from cycles table | ✅ Direct |
| Chart | WoW/MOM/QoQ toggle | Same query, different GROUP BY granularity | ✅ Derivable |
| Rankings | Incidents | COUNT events per vehicle | ✅ Direct |
| Rankings | Idle Time % | **NOT IN SCHEMA** | ❌ Needs C2 |
| Rankings | Fuel Wasted | `SUM(fuel_from_consumption_liters)` EB events per vehicle | ✅ Direct |
| Rankings | Impact | SUM × price | ⚠️ Needs C1 |
| Event Log | Vehicle | `uniqueid` | ✅ Direct |
| Event Log | Date & Time | `toDateTime(start_ts)` | ✅ Direct |
| Event Log | Duration | `duration_sec` | ✅ Direct |
| Event Log | Speed | `avg_speed_kmh` | ✅ Direct |
| Event Log | Deceleration Rate | See issue EB3 | ⚠️ Derived |
| Event Log | Gear No | `gear` | ✅ Direct |
| Monthly Table | Adoption rate | `AVG(engine_braking_pct)` per vehicle per month | ✅ Direct |
| Monthly Table | Event Count | COUNT per vehicle per month | ✅ Direct |
| Monthly Table | Est Fuel Savings | See issue EB1 | ⚠️ Open |
| Monthly Table | Potential Savings (100%) | See issue EB2 | ⚠️ Open |
| Monthly Table | Est Brake Savings | See issue EB4 | ❌ Open |

### Open Issues

| # | Issue | Status |
|---|-------|--------|
| EB1 | **Est Fuel Savings formula** — Design shows Mileage at Current Adpt = Baseline Mileage = 6.4 Kmpl (no difference), yet Est Fuel Savings = ₹28,500. The savings can't be from mileage difference alone. Is this based on: wear cost avoided? reduced brake pad consumption? or a different mileage benchmark (manufacturer spec vs actual)? | NEEDS CLARIFICATION |
| EB2 | **Potential Fuel Saving (100% adoption)** — how is this extrapolated? Formula: `current_savings / current_adoption_pct × 100`? Or scenario modelling? | NEEDS CLARIFICATION |
| EB3 | **Deceleration Rate** not stored in schema — must be computed at query time: `(start_speed_kmh - end_speed_kmh) / duration_sec × 0.2778` → m/s². Confirm display format is `−0.47 m/s²` (negative = deceleration). | OPEN — derivable, just confirm sign convention |
| EB4 | **Est Brake Savings** (Monthly table) — no brake wear model in schema at all. Requires either: (a) a cost-per-km-of-braking constant; (b) a separate brake wear table. Not implementable without additional data. | OPEN — blocked, need business input |
| EB5 | **`brake_during_event`** (UInt8 flag in `engine_braking_events_sm`) — not surfaced in any Figma component. Should this appear in the event log as an indicator (was brake also applied during EB)? | OPEN — potential future addition |
| EB6 | **Worst 20% / Top 20% toggle** in Vehicle Rankings — "Top 20%" would rank by highest adoption / best behaviour. Confirm: top = highest `engine_braking_pct` from cycles table? | NEEDS CLARIFICATION |

---

## Resolved Issues

_Move items here once confirmed._

| # | Issue | Resolution |
|---|-------|------------|
| — | — | — |

---

## Decisions Log

_Record any business/product decisions made during review._

| Date | Decision |
|------|----------|
| — | — |
