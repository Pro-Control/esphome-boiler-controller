# Problem Statement

This document explains the problem that the **Profile-Based Dual-Point Water Heater Controller** is trying to solve.

> Status: Design proposal  
> This project is not implemented yet.

## Summary

ESPHome and Home Assistant already provide useful entities for temperature control, but there is currently a gap between:

- `water_heater`: the correct domain for boilers and water heaters
- `climate`: the stronger controller model for thermostat-like behavior

This project proposes a new ESPHome `water_heater` platform that keeps the correct water-heater domain while adding controller behavior designed for real electric boilers and multi-stage water heaters.

## Current gap

### Water heater domain

The `water_heater` domain is the correct semantic domain for boilers and domestic hot water systems.

However, current template-style water heater behavior is mostly entity-oriented:

- current temperature
- target temperature
- operation mode
- away mode
- on/off state
- set action

This is useful for representing a water heater, but it does not fully model a real boiler controller.

Missing concepts include:

- explicit runtime action: `heating`, `idle`, `waiting_for`, `error`, `off`
- heat-only dual-point low/high control
- user-defined operating profiles
- multi-stage heater output logic
- interlock-based waiting states
- physical output feedback validation
- clear waiting and fault reasons

### Climate thermostat domain

The `climate` thermostat platform provides stronger controller behavior, such as:

- heating and idle actions
- target-based control
- anti-chatter timing
- thermostat-like state management

But it uses the `climate` domain, which is semantically designed for space temperature and HVAC devices, not domestic water heaters.

For a boiler, this creates a mismatch:

| Need | `climate` thermostat | `water_heater` |
|---|---|---|
| Correct water-heater domain | No | Yes |
| Heating / idle runtime behavior | Yes | Limited |
| Boiler-specific profiles | No | Limited |
| Heat-only dual-point water control | Not ideal | Not currently complete |
| Output feedback and interlocks | Custom logic required | Custom logic required |

## Real boiler requirements

A real electric boiler controller often needs more than a single target temperature.

Common requirements include:

### 1. Heat-only dual-point control

The controller should use a low/high target range:

| Condition | Result |
|---|---|
| Temperature <= target low | Start heating |
| Temperature >= target high | Stop heating |
| Temperature between low/high | Keep previous demand state |

The high target is the stop-heating point.

It is not a cooling point.

### 2. User-defined profiles

A boiler profile should describe a user-defined heating strategy.

Examples:

- Economy
- Balanced
- Fast Recovery
- Solar Only
- Cheap Power
- High Demand

The component should not hard-code profile names.

Each user should define their own profiles, icons, actions, interlocks, and expected outputs.

### 3. Multi-stage heater control

Many boilers may have more than one heating stage.

For example:

- primary heater stage
- assist heater stage
- boost heater stage

Different profiles may use different stages.

Example:

| Profile | Expected heating stages |
|---|---|
| Economy | Primary stage only |
| Balanced | Primary + Assist |
| Fast Recovery | Primary + Assist + Boost |

### 4. Physical output feedback

A relay command does not prove that a heater stage actually started.

The controller should support real-world feedback such as:

- contactor auxiliary contact
- relay feedback
- current sensor
- binary sensor
- power monitor

The controller should detect:

- a required stage did not start
- a stage did not stop
- an unexpected stage is on

### 5. Interlocks and waiting reasons

Some conditions should block heating without being treated as faults.

Examples:

- grid power unavailable
- water level not safe
- high-load permission not granted
- solar power insufficient
- cheap electricity window not active

When heating is blocked by these conditions, the controller should report:

```text
action = waiting_for
```

and expose a human-readable reason.

Example:

```yaml
waiting_reasons:
  - "Waiting for high load permission"
```

### 6. Explicit runtime actions

The controller should expose one clear runtime action:

```text
off
idle
heating
waiting_for
error
```

These actions are fixed by the component.

They are not user-defined.

### 7. Clear separation of concepts

This project separates concepts that are often mixed together:

| Concept | Meaning |
|---|---|
| Power | Whether the water heater is enabled |
| Profile | User-defined heating strategy |
| Action | Real runtime state calculated by the controller |
| Target range | Low/high temperature control range |
| Interlock | Temporary heating blocker |
| Feedback | Real-world output confirmation |
| Fault | Real failure requiring attention |

## Non-goals

This project does not aim to:

- replace all existing ESPHome water heater platforms
- replace the `climate` thermostat platform
- force built-in profiles like ECO or Performance
- treat `off` as a user-defined profile
- use dual-point heat/cool HVAC semantics
- hide real faults behind optimistic state updates
- replace mechanical safety devices

## Proposed solution

Add a new ESPHome water heater platform:

```yaml
water_heater:
  - platform: profile_dual_point
```

The platform provides:

- correct `water_heater` domain
- user-defined profiles
- heat-only dual-point control
- multiple temperature sensors
- interlocks with waiting reasons
- monitored heater outputs
- real output feedback validation
- explicit runtime action
- clear Home Assistant attributes

## Why this should be a water_heater platform

This controller is intended for boilers and domestic water-heater systems.

The controlled medium is water, not room air.

The correct semantic entity in Home Assistant is therefore `water_heater`, not `climate`.

The goal is to bring thermostat-like control behavior into the water-heater domain without misusing the climate domain.

## Expected outcome

A user should be able to create one water heater entity that clearly shows:

- current tank temperature
- target low/high range
- selected profile
- runtime action
- waiting reasons
- fault reasons
- active heater stages
- relevant temperature sensors

The end result should be a single official water-heater entity that can represent the real boiler state without requiring many separate helper entities.
