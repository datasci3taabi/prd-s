# Coasting Feature — Database Schema Documentation

## 1. Introduction

This document outlines the detailed database schema designed to support the Coasting driver
behaviour module. It supports independent engine ON cycle tracking as a foundational element and
dedicated tables for coasting cycle metadata, event data, and per-vehicle daily summaries.
Timestamp fields for creation and modification are included to enable Change Data Capture (CDC)
and auditing capabilities.

## 2. Core Concepts

- **Engine ON Cycle:** Time intervals when the engine runs, used as the primary reference for coasting data. Shared with all vehicle health features — no new table required.

- **CoastingCycleDetails:** Aggregated coasting statistics per engine ON cycle — coasting_pct, distance and fuel breakdowns, neutral and clutch coasting split, and critical event count.

- **CoastingEvents:** Individual coasting event intervals detected within each cycle, with behavioural severity, kinematics, dual fuel data, and critical packet count. Normal driving events are also stored to support accurate per-cycle totals. DQM flags are not stored at event level.

- **CoastingVehicleDayLevel:** Per-vehicle daily summary produced by a nightly cron job aggregating all cycle rows for that vehicle on that date. Used directly by fleet-overview and vehicle-overview endpoints.

- **Timestamps:** created_at and updated_at fields added for all tables support CDC and data auditing.

## 3. Tables and Structure

### 3.1 CoastingCycleDetails

Stores aggregated coasting behaviour statistics per engine ON cycle. One row per cycle.

| Column | Type | Description |
|---|---|---|
| coasting_cycle_id | UUID / PK | Unique identifier for this coasting cycle record |
| cycle_id | UUID / FK | Foreign key to EngineOnCycles |
| coasting_pct | Float | Percentage of total driving distance in coasting |
| overall_distance_km | Float | Total distance driven in the cycle |
| overall_consumption_liters | Float | Total fuel consumed in the cycle |
| coasting_distance_km | Float | Total coasting distance (neutral and clutch coasting combined) |
| coasting_consumption_liters | Float | Fuel consumed during all coasting events |
| neutral_coasting_distance_km | Float | Distance in neutral gear coasting specifically |
| clutch_coasting_distance_km | Float | Distance in clutch-held coasting specifically |
| normal_distance_km | Float | Distance driven normally (non-coasting) |
| normal_consumption_liters | Float | Fuel consumed during normal driving |
| normal_mileage_kmpl | Float | Fuel efficiency during normal driving |
| critical_event_count | Integer | Count of coasting events where vehicle speed was increasing (risky) |
| **created_at** | Timestamp | Record creation timestamp (for CDC) |
| **updated_at** | Timestamp | Last modification timestamp (for CDC) |

### 3.2 CoastingEvents

Stores individual coasting and normal driving event intervals per cycle.

| Column | Type | Description |
|---|---|---|
| event_id | UUID / PK | Unique identifier for the event |
| coasting_cycle_id | UUID / FK | Foreign key to CoastingCycleDetails |
| cycle_id | UUID / FK | Foreign key to EngineOnCycles |
| event_type | String | "neutral_coasting", "clutch_coasting", or "normal" |
| severity | String | Good / Normal / Bad / Critical |
| start_ts | Timestamp | Start time of event interval |
| end_ts | Timestamp | End time of event interval |
| duration_sec | Float | Event duration in seconds |
| distance_km | Float | Distance covered during the event |
| avg_speed_kmh | Float | Mean of per-packet vehicle speeds during the event |
| max_speed_kmh | Float | Maximum vehicle speed recorded during the event |
| avg_rpm | Float | Mean of per-packet RPM values during the event |
| fuel_from_consumption_liters | Float | Fuel consumed from cumulative consumption signal (nullable) |
| fuel_from_rate_liters | Float | Fuel consumed from instantaneous rate signal (nullable) |
| mileage_kmpl | Float | Fuel efficiency during the event; 0 if outside plausible range (1–15 km/L) |
| n_packets | Integer | Number of OBD packets that make up the event |
| critical_packets | Integer | Count of consecutive packet pairs where speed was increasing |
| start_lat | Float | Latitude at event start (nullable) |
| start_lng | Float | Longitude at event start (nullable) |
| end_lat | Float | Latitude at event end (nullable) |
| end_lng | Float | Longitude at event end (nullable) |
| **created_at** | Timestamp | Record creation timestamp (for CDC) |
| **updated_at** | Timestamp | Last modification timestamp (for CDC) |

### 3.3 CoastingVehicleDayLevel

Stores per-vehicle daily coasting summaries. One row per vehicle per date, written nightly by the cron job. The coasting_pct is computed as a distance-weighted ratio across all cycles for that date.

| Column | Type | Description |
|---|---|---|
| day_id | UUID / PK | Unique identifier for this day-level record |
| uniqueid | UUID / FK | Vehicle unique identifier |
| date | Date | The calendar date this record represents |
| total_cycles | Integer | Number of engine ON cycles driven on this date |
| total_distance_km | Float | Total distance driven across all cycles on this date |
| total_consumption_liters | Float | Total fuel consumed across all cycles on this date |
| coasting_distance_km | Float | Total coasting distance across all cycles |
| coasting_consumption_liters | Float | Fuel consumed during coasting across all cycles |
| coasting_mileage_kmpl | Float | Fuel efficiency during coasting |
| neutral_coasting_distance_km | Float | Neutral gear coasting distance across all cycles |
| clutch_coasting_distance_km | Float | Clutch-held coasting distance across all cycles |
| normal_distance_km | Float | Normal driving distance across all cycles |
| normal_consumption_liters | Float | Fuel consumed during normal driving across all cycles |
| normal_mileage_kmpl | Float | Fuel efficiency during normal driving |
| coasting_pct | Float | Distance-weighted coasting percentage for the day |
| behaviour | String | Good / Normal / Bad / Critical based on coasting_pct |
| **created_at** | Timestamp | Record creation timestamp (for CDC) |
| **updated_at** | Timestamp | Last modification timestamp (for CDC) |

## 4. Relationships Summary

| Primary Table | Related Table | Relationship Type |
|---|---|---|
| EngineOnCycles | CoastingCycleDetails | One-to-one (1 cycle → 1 cycle detail record) |
| CoastingCycleDetails | CoastingEvents | One-to-many (1 cycle record → many events) |
| uniqueid | CoastingVehicleDayLevel | One-to-many (1 vehicle → many daily summary rows) |
