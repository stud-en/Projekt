# Concepts: Train Operator Circulation Simulation (Zealand)

This document defines the architecture before implementation. It translates the project idea into an agent-based, MQTT-driven design with one notebook per agent and a dashboard notebook for visualization.

## 1. System Overview

### Trigger (Who/What is moving)
- The moving entity is one operator-train agent that executes discrete trips.
- One trip equals movement from the current station to one adjacent station on a predefined route graph.
- The trigger for each state transition is a simulation tick.

### Observer (What the city sees)
- The observer layer tracks system state at each tick:
  - current station
  - start station
  - completed trips
  - target trip count (`x`)
  - route segment just traversed
- The observer publishes normalized state events for dashboard and diagnostics.

### Control Center (Decision logic)
- The control layer selects the next valid adjacent station from the route graph.
- It enforces the closure constraint: after `x` trips, the final state must return to the start station.
- If closure is impossible under current path choices, control applies a deterministic fallback path-to-start policy.

### Response (System outcome)
- While `completed_trips < x`, the operator advances one segment per tick.
- The dashboard updates marker position and route trace after every movement event.
- The simulation ends in a success state only when both conditions are true:
  - `completed_trips == x`
  - `current_station == start_station`

### Geographic and route context
- The geographic scope is Zealand intercity rail links with one optional bridge connection toward Odense.
- The topology should be modeled as a graph of station nodes with explicit bidirectional edges.
- Based on the provided route sketch, the core station set is:
  - `Helsingor`
  - `Kobenhavn_H`
  - `CPH_Lufthavn`
  - `Roskilde`
  - `Ringsted`
  - `Slagelse`
  - `Kalundborg`
  - `Naestved`
  - `Nykobing_F`
  - `Odense` (optional extension)
- Core line intent from the sketch:
  - north-south branch from `Kobenhavn_H` to `Helsingor`
  - east-west trunk from `Kobenhavn_H` through `Roskilde` toward `Kalundborg`
  - airport branch from `Kobenhavn_H` to `CPH_Lufthavn`
  - southwest trunk through `Ringsted` and `Naestved` to `Nykobing_F`

### Route assumptions table (planning baseline)

| Line ID | From | To | Segment role | Bidirectional |
|---|---|---|---|---|
| `L1` | `Kobenhavn_H` | `Helsingor` | North branch | yes |
| `L2` | `Kobenhavn_H` | `Roskilde` | West trunk (inner) | yes |
| `L3` | `Roskilde` | `Kalundborg` | West trunk (outer) | yes |
| `L4` | `Kobenhavn_H` | `CPH_Lufthavn` | Airport branch | yes |
| `L5` | `Roskilde` | `Ringsted` | South trunk (inner) | yes |
| `L6` | `Ringsted` | `Naestved` | South trunk (middle) | yes |
| `L7` | `Naestved` | `Nykobing_F` | South trunk (outer) | yes |
| `L8` | `Ringsted` | `Odense` | Optional extension | yes |

Planned graph edge set for MVP route validation:
- `Kobenhavn_H <-> Helsingor`
- `Kobenhavn_H <-> Roskilde`
- `Roskilde <-> Kalundborg`
- `Kobenhavn_H <-> CPH_Lufthavn`
- `Roskilde <-> Ringsted`
- `Ringsted <-> Naestved`
- `Naestved <-> Nykobing_F`
- `Ringsted <-> Odense` (only when optional extension is enabled)

## 2. MQTT Architecture

Topic names are relative to `base_topic` from configuration (default: `simulated-city`).

### Topic catalog

| Topic | Published by | Subscribed by | Purpose |
|---|---|---|---|
| `operator/command` | `agent_dispatcher.ipynb` | `agent_operator.ipynb` | Start/pause/stop/reset commands |
| `operator/state` | `agent_operator.ipynb` | `agent_observer.ipynb`, `dashboard.ipynb` | Current operator state each tick |
| `operator/movement` | `agent_operator.ipynb` | `agent_observer.ipynb`, `dashboard.ipynb` | Per-trip movement event |
| `observer/metrics` | `agent_observer.ipynb` | `dashboard.ipynb` | Derived KPIs and validation checks |
| `simulation/lifecycle` | `agent_dispatcher.ipynb`, `agent_operator.ipynb` | All notebooks | Run lifecycle events (initialized, running, completed, failed) |
| `dashboard/ack` | `dashboard.ipynb` | `agent_dispatcher.ipynb` (optional) | UI/session heartbeat or acknowledgments |

### Message schema by topic

#### `operator/command`
- Publisher: `agent_dispatcher.ipynb`
- Subscribers: `agent_operator.ipynb`
- JSON fields:
  - `run_id` (string)
  - `timestamp` (ISO-8601 string)
  - `command` (string; one of `start`, `pause`, `resume`, `stop`, `reset`)
  - `start_station` (string)
  - `target_trips` (integer)
  - `route_id` (string)

#### `operator/state`
- Publisher: `agent_operator.ipynb`
- Subscribers: `agent_observer.ipynb`, `dashboard.ipynb`
- JSON fields:
  - `run_id` (string)
  - `timestamp` (ISO-8601 string)
  - `operator_id` (string)
  - `current_station` (string)
  - `start_station` (string)
  - `completed_trips` (integer)
  - `target_trips` (integer)
  - `is_closed_loop` (boolean)
  - `status` (string; `idle`, `running`, `paused`, `completed`, `failed`)

#### `operator/movement`
- Publisher: `agent_operator.ipynb`
- Subscribers: `agent_observer.ipynb`, `dashboard.ipynb`
- JSON fields:
  - `run_id` (string)
  - `timestamp` (ISO-8601 string)
  - `trip_index` (integer)
  - `from_station` (string)
  - `to_station` (string)
  - `segment_id` (string)
  - `duration_s` (number)

#### `observer/metrics`
- Publisher: `agent_observer.ipynb`
- Subscribers: `dashboard.ipynb`
- JSON fields:
  - `run_id` (string)
  - `timestamp` (ISO-8601 string)
  - `completion_ratio` (number, 0-1)
  - `remaining_trips` (integer)
  - `closure_feasible` (boolean)
  - `warnings` (array of strings)

#### `simulation/lifecycle`
- Publisher: `agent_dispatcher.ipynb`, `agent_operator.ipynb`
- Subscribers: all notebooks
- JSON fields:
  - `run_id` (string)
  - `timestamp` (ISO-8601 string)
  - `phase` (string; `initialized`, `running`, `paused`, `completed`, `failed`, `stopped`)
  - `reason` (string, optional)

#### `dashboard/ack`
- Publisher: `dashboard.ipynb`
- Subscribers: `agent_dispatcher.ipynb` (optional)
- JSON fields:
  - `run_id` (string)
  - `timestamp` (ISO-8601 string)
  - `session_id` (string)
  - `state` (string; `connected`, `ready`, `viewing`)

## 3. Configuration Parameters

All runtime parameters should be stored in `config.yaml` and loaded through `simulated_city.config.load_config()`.

### MQTT broker settings
- `mqtt.active_profiles`: list of active profile names. Default: `["default"]`
- `mqtt.profiles.default.host`: broker host. Default: `localhost`
- `mqtt.profiles.default.port`: broker port. Default: `1883`
- `mqtt.profiles.default.tls`: TLS enabled. Default: `false`
- `mqtt.profiles.default.username_env`: env-var name for username. Default: `MQTT_USERNAME`
- `mqtt.profiles.default.password_env`: env-var name for password. Default: `MQTT_PASSWORD`
- `mqtt.profiles.default.client_id_prefix`: client ID prefix. Default: `simcity`
- `mqtt.profiles.default.keepalive_s`: keepalive seconds. Default: `60`
- `mqtt.profiles.default.base_topic`: topic root. Default: `simulated-city`

### Network and map data (Zealand)
- `simulation.locations`: station coordinate list (id + lat/lon)
- Suggested default stations:
  - `Kobenhavn_H` (`55.6726`, `12.5646`)
  - `Helsingor` (`56.0365`, `12.6136`)
  - `CPH_Lufthavn` (`55.6280`, `12.6492`)
  - `Roskilde` (`55.6415`, `12.0803`)
  - `Ringsted` (`55.4427`, `11.7902`)
  - `Slagelse` (`55.4028`, `11.3546`)
  - `Kalundborg` (`55.6797`, `11.0886`)
  - `Naestved` (`55.2299`, `11.7609`)
  - `Nykobing_F` (`54.7691`, `11.8746`)
  - `Odense` (`55.4038`, `10.4024`) (optional extension)
- `simulation.routes`: adjacency or ordered route definitions connecting station IDs.
- Recommended route IDs:
  - `zealand_north`: `Kobenhavn_H -> Helsingor`
  - `zealand_west`: `Kobenhavn_H -> Roskilde -> Kalundborg`
  - `zealand_airport`: `Kobenhavn_H -> CPH_Lufthavn`
  - `zealand_south`: `Kobenhavn_H -> Roskilde -> Ringsted -> Naestved -> Nykobing_F`
  - `zealand_bridge_extension` (optional): `Ringsted -> Odense`

### Thresholds and limits
- `simulation.target_trips`: required number of trips `x`. Default: `24`
- `simulation.max_trips`: hard safety cap to avoid runaway loops. Default: `200`
- `simulation.min_route_nodes`: minimum stations in a route. Default: `2`
- `simulation.closure_required`: enforce return to start station. Default: `true`
- `simulation.max_closure_attempts`: retries for fallback return path. Default: `3`
- `simulation.require_bidirectional_edges`: each route edge must be reversible. Default: `true`
- `simulation.allow_optional_extension`: allow entering optional bridge route (`Odense`). Default: `false`

### Timing parameters
- `simulation.timestep_s`: seconds per simulation tick. Default: `5`
- `simulation.trip_duration_s`: fixed travel time per segment. Default: `180`
- `simulation.publish_interval_s`: interval for periodic state publication. Default: `5`
- `simulation.dashboard_refresh_s`: dashboard update cadence. Default: `1`
- `simulation.random_seed`: optional deterministic seed. Default: `42`

## 4. Architecture Decisions

### Notebooks to create (one notebook per agent)
- `notebooks/agent_dispatcher.ipynb`
  - Starts/stops runs, publishes lifecycle and command messages.
- `notebooks/agent_operator.ipynb`
  - Owns operator state, applies movement logic, publishes state/movement events.
- `notebooks/agent_observer.ipynb`
  - Subscribes to operator topics, computes metrics, publishes observer summaries.
- `notebooks/dashboard.ipynb`
  - Subscribes to state/movement/metrics, renders map and run status with `anymap-ts`.

### Library code (`src/simulated_city/`)

#### Data models (dataclasses)
- `Station`: `id`, `lat`, `lon`, optional display name.
- `Route`: `route_id`, ordered station IDs or adjacency mapping.
- `OperatorState`: run ID, current/start station, trip counters, status.
- `MovementEvent`: from/to station, trip index, duration, timestamp.
- `LifecycleEvent`: phase, reason, timestamp.

#### Utility functions
- Topic builders using configured `base_topic`.
- JSON serialization/deserialization and payload validation.
- Config validators for route consistency and station existence.

#### Calculation helpers
- Next-station selection from route adjacency.
- Closure feasibility check for remaining trips.
- Deterministic fallback path-to-start computation.

### Classes vs Functions

#### Model as classes
- Stateful notebook agents (dispatcher/operator/observer) should be class-based.
- Config and payload/data-model objects should be dataclasses.
- Long-lived MQTT session wrappers should be class-based.

#### Keep as functions
- Stateless calculations (closure checks, adjacency lookup).
- Pure transformations (payload normalization, route parsing).
- Coordinate transformations (reuse existing helpers in `simulated_city.geo`).

## 5. Open Questions

- Should command handling support multiple simultaneous operators now, or keep a strict single-operator scope for MVP?
- Should route definitions be stored as ordered line paths, graph adjacency, or both?
- Should the `Odense` extension be enabled in MVP runs, or reserved for phase 2?
- Is `target_trips` fixed per run command, or can it be changed during a paused run?
- Should lifecycle failures terminate immediately, or allow recovery/retry from last valid station?
- Which station subset is mandatory for MVP (minimum route set on Zealand)?
- Should dashboard acknowledge messages (`dashboard/ack`) be mandatory or optional?
- The docs currently reference helper names `connect_mqtt()` and `publish_json_checked()`, while current library code exposes `MqttConnector` and `MqttPublisher.publish_json()`. Confirm the canonical API before implementation.

## Assumptions used in this design
- The architecture stays distributed: one notebook per agent plus one dashboard notebook.
- MQTT topic root comes from `base_topic` and defaults to `simulated-city`.
- Simulation uses fixed segment travel time (no live delay model in MVP).
- The map layer uses `anymap-ts` for live visualization.
- All credentials come from environment variables referenced by `config.yaml`.