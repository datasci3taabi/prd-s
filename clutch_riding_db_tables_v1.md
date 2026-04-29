# Clutch Riding Feature — Database Schema Documentation

## 1. Introduction

This document outlines the detailed database schema designed to support the Clutch Riding driver
behaviour module. It supports independent engine ON cycle tracking as a foundational element and
dedicated tables for clutch riding cycle metadata, event data, and per-vehicle daily summaries.
The shared DriverBehaviourCycleDQM table stores data quality flags used to validate cycle results
across all driver behaviour modules. Timestamp fields for creation and modification are included
to enable Change Data Capture (CDC) and auditing capabilities.

## 2. Core Concepts

- **Engine ON Cycle:** Time intervals when the engine runs, used as the primary reference for clutch riding data. Shared with all vehicle health features — no new table required.

- **DriverBehaviourCycleDQM:** A single shared DQM table storing 7 data quality flags per engine ON cycle, computed from the raw OBD packet set. Populated from the clutchRiding API cycle-level dqm_flags response. One row per cycle, shared across all three driver behaviour modules.

- **ClutchRidingCycleDetails:** Aggregated clutch riding statistics per engine ON cycle — clutch_riding_pct and distance and fuel breakdowns.

- **ClutchRidingEvents:** Individual clutch riding and normal riding event intervals detected within each cycle, with behavioural severity, kinematics, and dual fuel data. DQM flags are not stored at event level.

- **ClutchRidingVehicleDayLevel:** Per-vehicle daily summary produced by a nightly cron job aggregating all cycle rows for that vehicle on that date. Used directly by fleet-overview and vehicle-overview endpoints.

- **Timestamps:** created_at and updated_at fields added for all tables support CDC and data auditing.

## 3. Tables and Structure

### 3.1 DriverBehaviourCycleDQM

Stores 7 data quality flags per engine ON cycle. One row per cycle, shared across all three driver behaviour modules. Populated from the clutchRiding API dqm_flags response.

| Column | Type | Description |
|---|---|---|
| dqm_id | UUID / PK | Unique identifier for the DQM record |
| cycle_id | UUID / FK | Foreign key to EngineOnCycles |
| fuel_missing | Boolean | fuel_consumption column absent or entirely NaN — fuel metrics unreliable |
| dist_missing | Boolean | obddistance column absent or entirely NaN — distance metrics unreliable |
| rpm_missing | Boolean | rpm absent, all NaN, or all zero — RPM averages unreliable |
| clutch_missing | Boolean | clutch_switch_status absent or all NaN — detection unreliable |
| gear_missing | Boolean | current_gear absent or all NaN |
| large_time_gap_exists | Boolean | Any packet gap exceeds 60 seconds — distance and fuel deltas inflated |
| impossible_speed_detected | Boolean | Any speed reading at or above 250 km/h — faulty sensor or data corruption |
| **created_at** | Timestamp | Record creation timestamp (for CDC) |
| **updated_at** | Timestamp | Last modification timestamp (for CDC) |

### 3.2 ClutchRidingCycleDetails

Stores aggregated clutch riding behaviour statistics per engine ON cycle. One row per cycle.

| Column | Type | Description |
|---|---|---|
| clutch_cycle_id | UUID / PK | Unique identifier for this clutch riding cycle record |
| cycle_id | UUID / FK | Foreign key to EngineOnCycles |
| clutch_riding_pct | Float | Percentage of total driving distance with clutch held |
| overall_distance_km | Float | Total distance driven in the cycle |
| overall_consumption_liters | Float | Total fuel consumed in the cycle |
| clutch_riding_distance_km | Float | Distance driven with clutch held |
| clutch_riding_consumption_liters | Float | Fuel consumed during clutch riding events |
| normal_riding_distance_km | Float | Distance driven normally (clutch released) |
| normal_riding_consumption_liters | Float | Fuel consumed during normal riding |
| normal_riding_mileage_kmpl | Float | Fuel efficiency during normal riding |
| **created_at** | Timestamp | Record creation timestamp (for CDC) |
| **updated_at** | Timestamp | Last modification timestamp (for CDC) |

### 3.3 ClutchRidingEvents

Stores individual clutch riding and normal riding event intervals per cycle.

| Column | Type | Description |
|---|---|---|
| event_id | UUID / PK | Unique identifier for the event |
| clutch_cycle_id | UUID / FK | Foreign key to ClutchRidingCycleDetails |
| cycle_id | UUID / FK | Foreign key to EngineOnCycles |
| event_type | String | "clutch_riding" or "normal_riding" |
| severity | String | Good / Normal / Bad / Critical for clutch_riding; Short / Medium / Good for normal_riding |
| event_start_ts | Timestamp | Start time of event interval |
| event_end_ts | Timestamp | End time of event interval |
| duration_sec | Float | Event duration in seconds |
| distance_km | Float | Distance covered during the event |
| avg_speed_kmh | Float | Mean of per-packet vehicle speeds during the event |
| avg_rpm | Float | Mean of per-packet RPM values during the event |
| fuel_from_consumption_liters | Float | Fuel consumed from cumulative consumption signal (nullable) |
| fuel_from_rate_liters | Float | Fuel consumed from instantaneous rate signal (nullable) |
| mileage_kmpl | Float | Fuel efficiency during the event; 0 if outside plausible range (1–15 km/L) |
| start_lat | Float | Latitude at event start (nullable) |
| start_lng | Float | Longitude at event start (nullable) |
| end_lat | Float | Latitude at event end (nullable) |
| end_lng | Float | Longitude at event end (nullable) |
| **created_at** | Timestamp | Record creation timestamp (for CDC) |
| **updated_at** | Timestamp | Last modification timestamp (for CDC) |

### 3.4 ClutchRidingVehicleDayLevel

Stores per-vehicle daily clutch riding summaries. One row per vehicle per date, written nightly by the cron job. The clutch_riding_pct is computed as a distance-weighted ratio across all cycles for that date.

| Column | Type | Description |
|---|---|---|
| day_id | UUID / PK | Unique identifier for this day-level record |
| uniqueid | UUID / FK | Vehicle unique identifier |
| date | Date | The calendar date this record represents |
| total_cycles | Integer | Number of engine ON cycles driven on this date |
| total_distance_km | Float | Total distance driven across all cycles on this date |
| total_consumption_liters | Float | Total fuel consumed across all cycles on this date |
| clutch_riding_distance_km | Float | Distance driven with clutch held across all cycles |
| clutch_riding_consumption_liters | Float | Fuel consumed during clutch riding across all cycles |
| clutch_riding_mileage_kmpl | Float | Fuel efficiency during clutch riding |
| normal_riding_distance_km | Float | Distance driven normally across all cycles |
| normal_riding_consumption_liters | Float | Fuel consumed during normal riding across all cycles |
| normal_riding_mileage_kmpl | Float | Fuel efficiency during normal riding |
| clutch_riding_pct | Float | Distance-weighted clutch riding percentage for the day |
| possible_saving_liters | Float | Estimated fuel saving if clutch riding distance had been driven at normal mileage |
| behaviour | String | Good / Normal / Bad / Critical based on clutch_riding_pct |
| data_anomaly_flag | Boolean | True when clutch mileage exceeds normal mileage — indicates corrupt fuel data |
| **created_at** | Timestamp | Record creation timestamp (for CDC) |
| **updated_at** | Timestamp | Last modification timestamp (for CDC) |

## 4. Relationships Summary

| Primary Table | Related Table | Relationship Type |
|---|---|---|
| EngineOnCycles | DriverBehaviourCycleDQM | One-to-one (1 cycle → 1 DQM record) |
| EngineOnCycles | ClutchRidingCycleDetails | One-to-one (1 cycle → 1 cycle detail record) |
| ClutchRidingCycleDetails | ClutchRidingEvents | One-to-many (1 cycle record → many events) |
| uniqueid | ClutchRidingVehicleDayLevel | One-to-many (1 vehicle → many daily summary rows) |
