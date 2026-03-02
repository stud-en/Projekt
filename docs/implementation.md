# Implementation Plan: Train Operator Simulation

This plan is based on the approved architecture in [docs/concepts.md](concepts.md). It is documentation-only and phase-gated so you can review each step before implementation.

## Phase 1: Minimal Working Example (Single Agent, No MQTT)

**Goal:** Build one runnable operator notebook with basic route movement and closed-loop validation, without network communication.

**New Files:**
- Create [notebooks/agent_operator.ipynb](../notebooks/agent_operator.ipynb) (notebook)
- Modify [docs/implementation.md](implementation.md) (documentation checklist updates)

**Implementation Details:**
- Implement local state variables in the notebook:
  - `run_id`
  - `start_station`
  - `current_station`
  - `completed_trips`
  - `target_trips`
- Use route assumptions from [docs/concepts.md](concepts.md) and move one segment per step.
- Enforce closure rule: simulation succeeds only if `completed_trips == target_trips` and `current_station == start_station`.
- Keep outputs simple (table/log print), no MQTT and no map yet.

**Dependencies:**
- No new packages expected.

**Verification:**
- Run `python scripts/verify_setup.py`
- Run `python -m pytest`
- Manual:
  - Open [notebooks/agent_operator.ipynb](../notebooks/agent_operator.ipynb)
  - Run all cells
  - Confirm trip counter updates correctly
  - Confirm the run ends only on closed-loop condition

**Investigation:**
- Validate the route edge assumptions from [docs/concepts.md](concepts.md).
- Confirm which default loops (4, 6, 8 trips) should be treated as baseline regression cases.

---

## Phase 2: Add Configuration-Driven Runtime

**Goal:** Move station/route/timing/limits into `config.yaml` and load settings through the library config loader.

**New Files:**
- Modify [config.yaml](../config.yaml) (config file)
- Modify [src/simulated_city/config.py](../src/simulated_city/config.py) (library module, only if parser changes are needed)
- Modify [notebooks/agent_operator.ipynb](../notebooks/agent_operator.ipynb) (notebook)
- Modify [docs/config.md](config.md) and [docs/implementation.md](implementation.md) (documentation)

**Implementation Details:**
- Read runtime settings via `simulated_city.config.load_config()`.
- Add/align simulation parameters with [docs/concepts.md](concepts.md):
  - locations
  - routes
  - `target_trips`
  - threshold limits
  - timing values
- Keep secrets referenced by env var names, not hardcoded credentials.

**Dependencies:**
- No new packages expected.

**Verification:**
- Run `python scripts/verify_setup.py`
- Run `python -m pytest`
- Manual:
  - Change one config value (for example `target_trips`)
  - Re-run notebook
  - Confirm behavior changes without notebook logic edits

**Investigation:**
- Decide whether routes are modeled as adjacency-only, ordered paths, or both.
- Confirm parser support for all simulation keys needed by the notebook.

---

## Phase 3: Add MQTT Publishing (Operator Agent)

**Goal:** Publish operator state, movement, and lifecycle events to MQTT topics while preserving Phase 2 behavior.

**New Files:**
- Modify [notebooks/agent_operator.ipynb](../notebooks/agent_operator.ipynb) (notebook)
- Modify [src/simulated_city/mqtt.py](../src/simulated_city/mqtt.py) (library module, only if helper changes are required)
- Modify [docs/mqtt.md](mqtt.md) and [docs/implementation.md](implementation.md) (documentation)

**Implementation Details:**
- Publish messages to topics defined in [docs/concepts.md](concepts.md):
  - `operator/state`
  - `operator/movement`
  - `simulation/lifecycle`
- Prefix topics with configured `base_topic`.
- Emit JSON payloads with required fields (`run_id`, timestamp, state/movement values).

**Dependencies:**
- No new packages expected (`paho-mqtt` already included).

**Verification:**
- Run `python scripts/verify_setup.py`
- Run `python -m pytest`
- Manual:
  - Start local broker profile
  - Run operator notebook
  - Verify published topic traffic (for example with `mosquitto_sub`)

**Investigation:**
- Resolve MQTT API naming consistency between docs and implementation (`connect_mqtt`/`publish_json_checked` vs class methods).
- Confirm QoS/retain defaults by topic category.

---

## Phase 4: Add Second Agent with MQTT Subscription

**Goal:** Add an observer agent that subscribes to operator topics, calculates metrics, and republishes observer outputs.

**New Files:**
- Create [notebooks/agent_observer.ipynb](../notebooks/agent_observer.ipynb) (notebook)
- Optional reusable helpers in [src/simulated_city](../src/simulated_city) (library modules)
- Modify [docs/implementation.md](implementation.md) (documentation)

**Implementation Details:**
- Subscribe to:
  - `operator/state`
  - `operator/movement`
- Compute and publish:
  - `observer/metrics`
- Start with minimal derived metrics:
  - `completion_ratio`
  - `remaining_trips`
  - `closure_feasible`
  - warning list

**Dependencies:**
- No new packages expected.

**Verification:**
- Run `python scripts/verify_setup.py`
- Run `python -m pytest`
- Manual:
  - Run operator and observer notebooks simultaneously
  - Confirm observer receives messages and publishes metrics updates

**Investigation:**
- Decide strict vs tolerant schema validation for incoming events.
- Determine restart behavior for observer state cache.

---

## Phase 5: Add Dashboard Visualization (anymap-ts)

**Goal:** Create a dedicated dashboard notebook that visualizes live operator movement and observer metrics.

**New Files:**
- Create [notebooks/dashboard.ipynb](../notebooks/dashboard.ipynb) (notebook)
- Modify [docs/maplibre_anymap.md](maplibre_anymap.md) and [docs/implementation.md](implementation.md) (documentation)

**Implementation Details:**
- Subscribe to:
  - `operator/state`
  - `operator/movement`
  - `observer/metrics`
- Render station nodes and route lines from config.
- Move operator marker as movement events arrive.
- Optionally publish UI heartbeat to `dashboard/ack`.
- Use `anymap-ts` for live map rendering (project rule).

**Dependencies:**
- No new packages expected (`anymap-ts[all]` already in notebook extras).

**Verification:**
- Run `python scripts/verify_setup.py`
- Run `python -m pytest`
- Manual:
  - Run operator, observer, and dashboard notebooks in parallel
  - Confirm live map updates and metrics panel refresh

**Investigation:**
- Validate refresh cadence vs simulation timestep.
- Check reconnect behavior after notebook restart.

---

## Phase 6+: Hardening, Orchestration, and Consistency

**Goal:** Stabilize the full multi-agent workflow and align code/docs/tests for instructor-ready review.

**New Files:**
- Create [notebooks/agent_dispatcher.ipynb](../notebooks/agent_dispatcher.ipynb) (notebook)
- Add or extend tests in [tests](../tests) (test modules)
- Update [README.md](../README.md), [docs/exercises.md](exercises.md), [docs/testing.md](testing.md), [docs/mqtt.md](mqtt.md), [docs/implementation.md](implementation.md) (documentation)

**Implementation Details:**
- Add control commands (`start`, `pause`, `resume`, `stop`, `reset`).
- Add focused tests for config/topic/schema helpers.
- Standardize MQTT API usage across docs and code.
- Add operational checklists for restart/recovery.

**Dependencies:**
- Add new dependencies only if clearly required and justified.

**Verification:**
- Run `python scripts/verify_setup.py`
- Run `python scripts/validate_structure.py`
- Run `python -m pytest`
- Manual:
  - Full end-to-end run across all notebooks
  - Confirm all required MQTT topics/messages are observed

**Investigation:**
- Decide if MVP remains single-operator or expands to multi-operator runs.
- Define final acceptance criteria for instructor review.

---

## Standard Phase Gate (Run After Every Phase)

- `python scripts/verify_setup.py`
- `python -m pytest`
- `python -m jupyterlab`
- Run the new/changed notebook(s) end-to-end and compare behavior with [docs/concepts.md](concepts.md)

## Notes

- Keep one notebook per agent.
- Do not install packages inside notebooks.
- Keep mapping on `anymap-ts` only for live visualization.
- Load all runtime parameters from `config.yaml` through `load_config()`.
