# Engine Braking Feature — Database Schema Documentation

## 1. Introduction

This document outlines the detailed database schema designed to support the Engine Braking driver
behaviour module. It supports independent engine ON cycle tracking as a foundational element and
dedicated tables for engine braking cycle metadata, event data, and per-vehicle daily summaries.
Engine braking is a positive driving behaviour — higher engine_braking_pct indicates the driver is
decelerating safely using the engine rather than relying solely on friction brakes. Timestamp
fields for creation and modification are included to enable Change Data Capture (CDC) and auditing
capabilities.

## 2. Core Concepts

- **Engine ON Cycle:** Time intervals when the engine runs, used as the primary reference for engine braking data. Shared with all vehicle health features — no new table required.

- **EngineBrakingCycleDetails:** Aggregated engine braking statistics per engine ON cycle — engine_braking_pct and distance and fuel breakdowns for engine braking and normal driving.

- **EngineBrakingEvents:** Individual engine braking event intervals detected within each cycle, with gear, kinematics, engine load, brake switch data, and dual fuel data. No severity labels are applied — engine braking duration does not indicate a problem; any sustained engine braking is positive behaviour. DQM flags are not stored at event level.

- **EngineBrakingVehicleDayLevel:** Per-vehicle daily summary produced by a nightly cron job aggregating all cycle rows for that vehicle on that date. Used directly by fleet-overview and vehicle-overview endpoints. Behaviour thresholds are inverted — higher engine_braking_pct is better.

- **Timestamps:** created_at and updated_at fields added for all tables support CDC and data auditing.

## 3. Tables and Structure

### 3.1 EngineBrakingCycleDetails

Stores aggregated engine braking behaviour statistics per engine ON cycle. One row per cycle.

| Column | Type | Description |
|---|---|---|
| engine_braking_cycle_id | UUID / PK | Unique identifier for this engine braking cycle record |
| cycle_id | UUID / FK | Foreign key to EngineOnCycles |
| engine_braking_pct | Float | Percentage of total driving distance under engine braking |
| overall_distance_km | Float | Total distance driven in the cycle |
| overall_consumption_liters | Float | Total fuel consumed in the cycle |
| engine_braking_distance_km | Float | Distance covered during engine braking events |
| engine_braking_consumption_liters | Float | Fuel consumed during engine braking events |
| normal_distance_km | Float | Distance driven without engine braking |
| normal_consumption_liters | Float | Fuel consumed during normal driving |
| normal_mileage_kmpl | Float | Fuel efficiency during normal driving |
| **created_at** | Timestamp | Record creation timestamp (for CDC) |
| **updated_at** | Timestamp | Last modification timestamp (for CDC) |

### 3.2 EngineBrakingEvents

Stores individual engine braking and normal driving event intervals per cycle. Normal driving events are also stored to support accurate per-cycle distance and fuel totals.

| Column | Type | Description |
|---|---|---|
| event_id | UUID / PK | Unique identifier for the event |
| engine_braking_cycle_id | UUID / FK | Foreign key to EngineBrakingCycleDetails |
| cycle_id | UUID / FK | Foreign key to EngineOnCycles |
| event_type | String | "engine_braking" or "normal" |
| duration_sec | Float | Event duration in seconds |
| n_packets | Integer | Number of OBD packets that make up the event |
| start_ts | Timestamp | Start time of event interval |
| end_ts | Timestamp | End time of event interval |
| start_speed_kmh | Float | Vehicle speed at event start |
| end_speed_kmh | Float | Vehicle speed at event end |
| avg_speed_kmh | Float | Mean of per-packet vehicle speeds during the event |
| max_speed_kmh | Float | Maximum vehicle speed recorded during the event |
| gear | Integer | Gear engaged during the event |
| avg_rpm | Float | Mean of per-packet RPM values during the event |
| max_rpm | Float | Maximum RPM recorded during the event |
| avg_engine_load | Float | Mean engine load percentage during the event |
| distance_km | Float | Distance covered during the event |
| fuel_from_consumption_liters | Float | Fuel consumed from cumulative consumption signal (nullable) |
| fuel_from_rate_liters | Float | Fuel consumed from instantaneous rate signal (nullable) |
| mileage_kmpl | Float | Fuel efficiency during the event; 0 if outside plausible range (1–15 km/L) |
| brake_applied_packets | Integer | Count of packets where brake switch was active during the event |
| brake_during_event | Boolean | True if brake was applied at any point during the event |
| start_lat | Float | Latitude at event start (nullable) |
| start_lng | Float | Longitude at event start (nullable) |
| end_lat | Float | Latitude at event end (nullable) |
| end_lng | Float | Longitude at event end (nullable) |
| **created_at** | Timestamp | Record creation timestamp (for CDC) |
| **updated_at** | Timestamp | Last modification timestamp (for CDC) |

### 3.3 EngineBrakingVehicleDayLevel

Stores per-vehicle daily engine braking summaries. One row per vehicle per date, written nightly by the cron job. The engine_braking_pct is computed as a distance-weighted ratio across all cycles for that date. Behaviour thresholds are inverted — a higher percentage means better driving behaviour.

| Column | Type | Description |
|---|---|---|
| uniqueid | UUID / FK | Vehicle unique identifier |
| date | Date | The calendar date this record represents |
| total_cycles | Integer | Number of engine ON cycles driven on this date |
| total_distance_km | Float | Total distance driven across all cycles on this date |
| total_consumption_liters | Float | Total fuel consumed across all cycles on this date |
| engine_braking_distance_km | Float | Distance covered under engine braking across all cycles |
| engine_braking_consumption_liters | Float | Fuel consumed during engine braking across all cycles |
| engine_braking_mileage_kmpl | Float | Fuel efficiency during engine braking |
| normal_distance_km | Float | Normal driving distance across all cycles |
| normal_consumption_liters | Float | Fuel consumed during normal driving across all cycles |
| normal_mileage_kmpl | Float | Fuel efficiency during normal driving |
| engine_braking_pct | Float | Distance-weighted engine braking percentage for the day |
| behaviour | String | Good / Normal / Bad / Critical based on engine_braking_pct (inverted thresholds) |
| **created_at** | Timestamp | Record creation timestamp (for CDC) |
| **updated_at** | Timestamp | Last modification timestamp (for CDC) |

## 4. Relationships Summary

| Primary Table | Related Table | Relationship Type |
|---|---|---|
| EngineOnCycles | EngineBrakingCycleDetails | One-to-one (1 cycle → 1 cycle detail record) |
| EngineBrakingCycleDetails | EngineBrakingEvents | One-to-many (1 cycle record → many events) |
| uniqueid | EngineBrakingVehicleDayLevel | One-to-many (1 vehicle → many daily summary rows) |
