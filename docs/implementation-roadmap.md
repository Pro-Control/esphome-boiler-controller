# Implementation Roadmap

This document describes the proposed path from design concept to a working ESPHome external component, and later to official ESPHome and Home Assistant proposals.

> Status: Planning document  
> This project is currently a design proposal and proof-of-concept plan.

## Project goal

Build a **Profile-Based Dual-Point Water Heater Controller** for ESPHome and Home Assistant.

The controller should provide a real boiler-control model inside the `water_heater` domain, including:

- user-defined profiles
- heat-only dual-point target range
- multiple temperature sensors
- interlocks / future permissive conditions
- real physical output feedback
- explicit runtime actions
- clear waiting and fault reasons

## Target platform name

Proposed ESPHome platform:

```yaml
water_heater:
  - platform: profile_dual_point
```

Repository name:

```text
esphome-boiler-controller
```

Project title:

```text
Profile-Based Dual-Point Water Heater Controller
```

## High-level phases

| Phase | Target | Output |
|---:|---|---|
| 1 | Design documentation | Clear proposal, YAML API, state machine, examples |
| 2 | ESPHome external component | Working proof of concept outside ESPHome core |
| 3 | Local testing | Validate logic with simulated and real hardware |
| 4 | ESPHome feature request | Present the idea to ESPHome maintainers |
| 5 | ESPHome pull request | Add official platform if accepted |
| 6 | ESPHome docs pull request | Add official documentation |
| 7 | Home Assistant architecture discussion | Propose missing water_heater domain features |
| 8 | Home Assistant Core pull request | Add accepted domain attributes/features |
| 9 | Home Assistant Frontend pull request | Improve official water_heater card display |

## Phase 1: Design documentation

Goal: make the idea understandable before writing production code.

Required files:

```text
README.md
docs/problem-statement.md
docs/yaml-api-concept.md
docs/state-machine.md
docs/implementation-roadmap.md
examples/minimal.yaml
examples/advanced-multi-stage-boiler.yaml
```

Expected outcome:

- The problem is clearly explained.
- The proposed YAML API is readable.
- The state machine is deterministic.
- The examples show minimal and advanced usage.
- Naming avoids confusion between profile, relay, feedback, interlock, and heater stage.

Completion criteria:

- A new reader can understand the purpose of the component without extra explanation.
- The YAML concept clearly separates:
  - profile
  - action
  - temperature input
  - monitored output
  - command relay
  - feedback condition
  - interlock
  - expected output state

## Phase 2: ESPHome external component proof of concept

Goal: build a working component that can be imported by ESPHome through `external_components`.

Planned structure:

```text
components/
└─ profile_dual_point/
   ├─ __init__.py
   ├─ water_heater.py
   ├─ profile_dual_point_water_heater.h
   └─ profile_dual_point_water_heater.cpp
```

Expected ESPHome usage:

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/Pro-Control/esphome-boiler-controller
    components:
      - profile_dual_point

water_heater:
  - platform: profile_dual_point
    name: "Boiler"
```

Main implementation tasks:

1. Define YAML schema validation.
2. Parse temperature sensors.
3. Parse target range configuration.
4. Parse timing options.
5. Parse interlocks.
6. Parse monitored outputs.
7. Parse user-defined profiles.
8. Implement controller state machine.
9. Store and restore runtime target/profile/power values.
10. Expose entity state and attributes to Home Assistant.
11. Add debug logging.
12. Add safety-oriented defaults.

Completion criteria:

- The component compiles inside ESPHome.
- Minimal example validates.
- Advanced example validates.
- Runtime state changes are visible in logs.
- The component can run with simulated switches and binary sensors.

## Phase 3: Simulated testing

Goal: test logic without real heater hardware.

Suggested simulated entities:

- template temperature sensor
- template switches for command relays
- template binary sensors for feedback
- template binary sensors for interlocks
- Home Assistant UI or ESPHome Web Server controls

Test cases:

| Test | Expected result |
|---|---|
| No target range configured | `action = waiting_for` |
| Temperature below low target | Heat demand starts |
| Temperature above high target | Heat demand stops |
| Interlock false during heat demand | `action = waiting_for` |
| Feedback confirms required outputs | `action = heating` |
| Required feedback missing | `action = error` |
| Unexpected output active | `action = error` |
| User disables heater | `action = off` |
| User changes profile | Old profile idles before new profile starts |
| Sensor becomes invalid | `action = error` |

Completion criteria:

- Every runtime action can be reproduced:
  - `off`
  - `idle`
  - `heating`
  - `waiting_for`
  - `error`
- Waiting reasons are exposed.
- Fault reasons are exposed.
- Fault reset policy works as expected.

## Phase 4: Real hardware proof of concept

Goal: validate the model on safe, controlled hardware.

Important safety rule:

This phase must not bypass mechanical or electrical safety devices.

Recommended hardware testing order:

1. Low-voltage relay simulation.
2. Low-voltage load simulation.
3. Current sensor / feedback simulation.
4. One real heater stage in a controlled setup.
5. Multi-stage controlled test only after earlier tests pass.

Hardware checks:

- relay command works
- feedback changes after relay command
- feedback timeout detects failure
- idle_action turns off outputs
- interlocks block heat_action
- manual fault reset works

Completion criteria:

- One-stage test passes.
- Multi-stage test passes.
- No unsafe automatic restart after feedback fault when manual reset is configured.
- Logs clearly explain every state transition.

## Phase 5: ESPHome feature request

Goal: present the project to ESPHome maintainers before opening a large pull request.

Feature request title:

```text
Add Profile-Based Dual-Point Water Heater Controller
```

Feature request summary:

```text
Add a new ESPHome water_heater platform for electric boilers and multi-stage water heaters. The platform provides heat-only dual-point control, user-defined profiles, interlocks with waiting reasons, real output feedback validation, and explicit runtime actions while keeping the entity in the water_heater domain.
```

Include links to:

- README
- problem statement
- YAML API concept
- state machine
- minimal example
- advanced example
- external component proof of concept

Do not ask to replace existing water heater platforms.

Position this as:

```text
A new optional platform:
water_heater:
  - platform: profile_dual_point
```

## Phase 6: ESPHome official pull request

Goal: merge the platform into ESPHome core if maintainers agree.

Recommended PR scope:

```text
Add profile_dual_point water_heater platform
```

Keep the PR focused on ESPHome only.

Do not include Home Assistant frontend changes in this PR.

Implementation expectations:

- schema validation
- code generation
- C++ component
- tests / example config
- clean logs
- safe defaults
- backwards compatibility

Possible review concerns:

| Concern | Response |
|---|---|
| Too complex | Keep minimal config simple and advanced features optional |
| Water heater domain limitations | Implement what ESPHome can expose now, document future HA needs |
| User-defined profiles | Keep them as controller profiles, not built-in HA operation modes |
| Safety | Use conservative defaults, feedback checks, and manual fault reset |
| Naming | Keep terms consistent and industrial: profile, interlock, stage, feedback |

## Phase 7: ESPHome documentation pull request

Goal: add official documentation after or alongside the ESPHome platform PR.

Documentation should include:

- overview
- minimal example
- advanced multi-stage example
- configuration variables
- runtime actions
- profile behavior
- target range behavior
- interlock behavior
- feedback behavior
- fault reset behavior
- safety notes

Suggested documentation path:

```text
components/water_heater/profile_dual_point.rst
```

## Phase 8: Home Assistant architecture proposal

Goal: propose improvements to the `water_heater` domain so Home Assistant can display the controller properly.

This should happen after the ESPHome concept is clear and preferably after a working proof of concept exists.

Proposed Home Assistant concepts:

```text
WaterHeaterAction:
  - off
  - idle
  - heating
  - waiting_for
  - error
```

Suggested attributes:

```yaml
profile: "Balanced"
action: "heating"
target_temperature_low: 55.0
target_temperature_high: 65.0
temperature_calculation: "average"
active_outputs:
  - primary_heater_stage
waiting_reasons: []
fault_reasons: []
```

Important design position:

- `profile` is not the same as operation mode.
- `action` is calculated runtime state.
- `waiting_for` is not a fault.
- `error` is a real fault.
- `off` is power/disabled state, not a user-defined profile.

## Phase 9: Home Assistant Core pull request

Goal: add accepted domain-level features or attributes after architecture approval.

Keep this PR small and focused.

Possible Core changes:

- water heater action property
- profile attribute support
- waiting reasons
- fault reasons
- low/high target range handling if accepted
- feature flags if required

Do not change frontend card behavior in this PR unless explicitly requested.

## Phase 10: Home Assistant Frontend pull request

Goal: improve the official water heater card display.

Possible frontend display:

- selected profile
- runtime action badge
- target low/high range
- current calculated temperature
- waiting reason section
- fault reason section
- active heater stages
- secondary temperature sensor display

Example UI intent:

```text
Boiler
Profile: Balanced
Action: Heating
Target: 55–65 °C
Current: 52.7 °C
Outputs: Primary, Assist
```

Waiting example:

```text
Boiler
Profile: Fast Recovery
Action: Waiting for
Reason: Waiting for high load permission
```

Fault example:

```text
Boiler
Profile: Balanced
Action: Error
Fault: Assist heater stage did not start
```

## Repository milestones

Suggested GitHub milestones:

| Milestone | Goal |
|---|---|
| `v0.1-design` | Documentation, YAML concept, examples |
| `v0.2-poc-schema` | ESPHome schema validation compiles |
| `v0.3-poc-state-machine` | Runtime state machine works in simulation |
| `v0.4-poc-feedback` | Feedback/interlocks/faults implemented |
| `v0.5-hardware-test` | Controlled hardware test complete |
| `v1.0-external-component` | Stable external component release candidate |
| `upstream-esphome` | ESPHome feature request / PR |
| `upstream-ha` | HA architecture / Core / Frontend proposals |

## Recommended GitHub labels

Suggested labels:

| Label | Purpose |
|---|---|
| `design` | YAML/API/state-machine discussion |
| `schema` | ESPHome YAML validation |
| `state-machine` | Controller runtime logic |
| `feedback` | Output feedback/readback |
| `interlocks` | Waiting conditions |
| `safety` | Safety behavior and defaults |
| `ha-domain` | Home Assistant water_heater model |
| `frontend` | UI/card discussion |
| `docs` | Documentation work |
| `poc` | Proof-of-concept implementation |

## Open design questions

These should be resolved before upstream PRs:

1. Should fault reset default to `manual` only, or allow `auto_when_clear`?
2. Should `start_permissive` stop active heating if it becomes false, or only block new starts?
3. Should all temperature sensors be required by default?
4. Should output feedback be mandatory or optional?
5. Should user-defined profiles map to HA operation modes, or remain as a separate `profile` attribute?
6. How should target low/high be exposed through the current Home Assistant water_heater domain?
7. Should the component support one profile only for very simple boilers?
8. Should expected output faults be latched by default?
9. Should the component expose active outputs as attributes only, or also as diagnostic entities?
10. What is the minimum Home Assistant frontend support needed for a useful first release?

## Recommended first implementation strategy

Start small.

Implement this order:

1. One temperature sensor.
2. One profile.
3. One monitored output.
4. Basic dual-point logic.
5. Runtime action: `off`, `idle`, `heating`.
6. Add `waiting_for` for missing target range.
7. Add run_interlocks.
8. Add feedback validation.
9. Add `error` state.
10. Add multiple profiles.
11. Add multiple temperature sensors.
12. Add advanced timing.
13. Add Home Assistant attributes.

This avoids building the full advanced version before the core logic is proven.

## Summary

The safest path is:

```text
Design docs
→ examples
→ external component
→ simulated tests
→ controlled hardware tests
→ ESPHome feature request
→ ESPHome PR
→ ESPHome docs PR
→ Home Assistant architecture proposal
→ Home Assistant Core PR
→ Home Assistant Frontend PR
```

The most important rule:

```text
Do not start with a large upstream PR.
Start with a clear design and a working external component.
```
