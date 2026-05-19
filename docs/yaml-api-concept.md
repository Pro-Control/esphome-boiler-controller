# YAML API Concept

This document describes the proposed YAML API for the **Profile-Based Dual-Point Water Heater Controller**.

> Status: Design concept  
> This YAML is not currently supported by ESPHome as-is.

## Purpose

The goal is to provide an ESPHome `water_heater` platform that behaves like a real boiler controller, while staying in the correct Home Assistant `water_heater` domain.

The controller is designed for electric boilers and multi-stage water heaters that need:

- heat-only dual-point control
- user-defined heating profiles
- multiple temperature sensors
- interlocks and permissive conditions
- physical output feedback
- explicit runtime actions for Home Assistant
- predictable safety-oriented state transitions

## Proposed platform

```yaml
water_heater:
  - platform: profile_dual_point
    name: "Boiler"
```

## Design principles

### Profiles are user-defined only

The component does not provide built-in profiles such as `ECO`, `Normal`, or `Performance`.

Users define their own profiles, names, icons, actions, interlocks, and expected output states.

### Runtime action is fixed by the component

The runtime action is not user-defined. It is calculated internally by the controller.

Supported runtime actions:

| Action | Meaning |
|---|---|
| `heating` | Heating is required and the physical outputs are confirmed active |
| `idle` | The controller is enabled, but no heating is currently required |
| `waiting_for` | Heating is required, but blocked by an interlock or missing configuration |
| `error` | A real fault exists, such as invalid sensor data or failed output feedback |
| `off` | The water heater is disabled by the user |

### Interlocks are not faults

Interlocks describe temporary conditions that block heating.

Examples:

- grid power not available
- water level not safe
- high-load permission not granted
- solar power not sufficient
- cheap power window not active

When an interlock blocks heating, the controller should report:

```text
action = waiting_for
```

and expose one or more waiting reasons.

### Feedback validates real physical output state

A relay command is not enough to confirm that a heater stage really started or stopped.

The controller should support output feedback/readback, such as:

- relay auxiliary contact
- contactor feedback
- current sensor
- binary sensor confirming the stage state

The component should be able to detect both:

- a required heater stage did not start
- an unexpected heater stage is still on

## Concept YAML

```yaml
# ============================================================
# Profile-Based Dual-Point Water Heater Controller
# Concept YAML - not currently supported by ESPHome as-is
# ============================================================

water_heater:
  - platform: profile_dual_point
    name: "Boiler"
    id: boiler_water_heater
    icon: mdi:water-boiler

    # --------------------------------------------------------
    # Target temperature range
    # --------------------------------------------------------
    # Low/high target values are runtime values.
    #
    # The user sets them from Home Assistant / Web Server UI.
    # The component restores the last saved values internally.
    #
    # No default target low/high values are required in YAML.
    #
    # If no saved target range exists:
    # action = waiting_for
    # reason = "Target temperature range not configured"
    # --------------------------------------------------------
    target:
      require_configured_range: true
      minimum_span: 3.0


    # --------------------------------------------------------
    # Temperature input section
    # --------------------------------------------------------
    # These are measurement sensors, not heater outputs.
    #
    # The UI may show each sensor in a defined card slot.
    # The controller uses the calculated temperature for logic.
    # --------------------------------------------------------
    temperature:
      sensors:
        - id: tank_top_temperature
          sensor: Boiler_Tank_Top_Temperature
          name: "Tank Top Temperature"
          icon: mdi:thermometer-chevron-up
          ui_slot: top_left

        - id: tank_middle_temperature
          sensor: Boiler_Tank_Middle_Temperature
          name: "Tank Middle Temperature"
          icon: mdi:thermometer
          ui_slot: top_right

        - id: tank_bottom_temperature
          sensor: Boiler_Tank_Bottom_Temperature
          name: "Tank Bottom Temperature"
          icon: mdi:thermometer-chevron-down
          ui_slot: bottom_left

      calculation:
        # Which temperature value controls the boiler logic.
        method: average
        # allowed:
        # - primary
        # - average
        # - minimum
        # - maximum

        # Used only when method: primary.
        # primary_sensor: tank_middle_temperature

        # If any configured temperature sensor fails,
        # stop heating and report action = error.
        unavailable_behavior: error


    # --------------------------------------------------------
    # UI limits only
    # --------------------------------------------------------
    # This defines the allowed UI range.
    # It does NOT set default target values.
    # --------------------------------------------------------
    visual:
      min_temperature: 10.0
      max_temperature: 85.0
      target_temperature_step: 0.5


    # --------------------------------------------------------
    # Anti-chatter / safety timing
    # --------------------------------------------------------
    # These options prevent rapid start/stop cycles.
    # --------------------------------------------------------
    timing:
      min_heat_run_time: 60s
      min_heat_off_time: 60s
      min_idle_time: 10s


    # --------------------------------------------------------
    # Feedback timing
    # --------------------------------------------------------
    # After heat_action or idle_action, wait this time before
    # checking if the physical outputs really changed state.
    # --------------------------------------------------------
    feedback:
      settle_time: 10s


    # --------------------------------------------------------
    # Fault behavior
    # --------------------------------------------------------
    # Feedback faults are real faults, not temporary waiting states.
    #
    # For water heaters, manual reset is usually safer because
    # the user should notice and inspect real output failures.
    # --------------------------------------------------------
    faults:
      reset: manual
      # allowed:
      # - manual
      # - auto_when_clear


    # --------------------------------------------------------
    # Monitored heater outputs
    # --------------------------------------------------------
    # This section defines the physical heating stages.
    #
    # Important naming:
    # - heat_action / idle_action command the relay/switch outputs.
    # - on_condition/off_condition are real feedback/readback checks.
    # - the stage id is what profiles reference in expected_outputs.
    # --------------------------------------------------------
    monitored_outputs:
      - id: primary_heater_stage
        name: "Primary Heater Stage"
        icon: mdi:heat-wave

        on_condition:
          binary_sensor.is_on: Boiler_Primary_Heater_Feedback

        off_condition:
          binary_sensor.is_off: Boiler_Primary_Heater_Feedback

        on_fault_reason: "Primary heater stage did not start"
        off_fault_reason: "Primary heater stage did not stop"


      - id: assist_heater_stage
        name: "Assist Heater Stage"
        icon: mdi:heat-wave

        on_condition:
          binary_sensor.is_on: Boiler_Assist_Heater_Feedback

        off_condition:
          binary_sensor.is_off: Boiler_Assist_Heater_Feedback

        on_fault_reason: "Assist heater stage did not start"
        off_fault_reason: "Assist heater stage did not stop"


      - id: boost_heater_stage
        name: "Boost Heater Stage"
        icon: mdi:flash

        on_condition:
          binary_sensor.is_on: Boiler_Boost_Heater_Feedback

        off_condition:
          binary_sensor.is_off: Boiler_Boost_Heater_Feedback

        on_fault_reason: "Boost heater stage did not start"
        off_fault_reason: "Boost heater stage did not stop"


    # --------------------------------------------------------
    # Interlocks / permissives
    # --------------------------------------------------------
    # These are not faults.
    # They are temporary reasons that block heating.
    #
    # If an active interlock is false:
    # action = waiting_for
    # heat_action is blocked
    # idle_action of the current profile is applied
    # --------------------------------------------------------
    interlocks:
      - id: grid_power_available
        name: "Grid Power Available"
        icon: mdi:transmission-tower

        type: run_interlock
        # v0.1 examples use run_interlock only.
        # start_permissive remains an open design question.

        condition:
          - binary_sensor.is_on: Boiler_Grid_Power_Available

        waiting_reason: "Waiting for grid power"


      - id: safe_water_level
        name: "Safe Water Level"
        icon: mdi:water-alert

        type: run_interlock

        condition:
          - binary_sensor.is_on: Boiler_Water_Level_OK

        waiting_reason: "Waiting for safe water level"


      - id: high_load_permission
        name: "High Load Permission"
        icon: mdi:meter-electric

        type: run_interlock

        condition:
          - binary_sensor.is_on: Boiler_High_Load_Allowed

        waiting_reason: "Waiting for high load permission"


    # --------------------------------------------------------
    # User-defined profiles
    # --------------------------------------------------------
    # Profiles are user-defined only.
    #
    # The component does not provide built-in Economy, Balanced,
    # Fast Recovery, or any other fixed profile names.
    #
    # Each profile defines:
    # - name/icon shown in UI
    # - interlocks required by this profile
    # - heat_action
    # - idle_action
    # - expected output states during heating/idle
    # --------------------------------------------------------
    profiles:

      - id: economy_profile
        name: "Economy"
        icon: mdi:leaf

        interlocks:
          - grid_power_available
          - safe_water_level

        heat_action:
          - switch.turn_on: Boiler_Primary_Heater_Relay

        idle_action:
          - switch.turn_off: Boiler_Primary_Heater_Relay

        expected_outputs:
          heating:
            on:
              - primary_heater_stage
            off:
              - assist_heater_stage
              - boost_heater_stage

          idle:
            off:
              - primary_heater_stage
              - assist_heater_stage
              - boost_heater_stage


      - id: balanced_profile
        name: "Balanced"
        icon: mdi:water-boiler

        interlocks:
          - grid_power_available
          - safe_water_level

        heat_action:
          - switch.turn_on: Boiler_Primary_Heater_Relay
          - switch.turn_on: Boiler_Assist_Heater_Relay

        idle_action:
          - switch.turn_off: Boiler_Primary_Heater_Relay
          - switch.turn_off: Boiler_Assist_Heater_Relay

        expected_outputs:
          heating:
            on:
              - primary_heater_stage
              - assist_heater_stage
            off:
              - boost_heater_stage

          idle:
            off:
              - primary_heater_stage
              - assist_heater_stage
              - boost_heater_stage


      - id: fast_recovery_profile
        name: "Fast Recovery"
        icon: mdi:flash

        interlocks:
          - grid_power_available
          - safe_water_level
          - high_load_permission

        heat_action:
          - switch.turn_on: Boiler_Primary_Heater_Relay
          - switch.turn_on: Boiler_Assist_Heater_Relay
          - switch.turn_on: Boiler_Boost_Heater_Relay

        idle_action:
          - switch.turn_off: Boiler_Primary_Heater_Relay
          - switch.turn_off: Boiler_Assist_Heater_Relay
          - switch.turn_off: Boiler_Boost_Heater_Relay

        expected_outputs:
          heating:
            on:
              - primary_heater_stage
              - assist_heater_stage
              - boost_heater_stage

          idle:
            off:
              - primary_heater_stage
              - assist_heater_stage
              - boost_heater_stage
```

## State machine summary

```text
if water_heater is disabled:
    action = off

else if target low/high range is not configured:
    action = waiting_for
    reason = "Target temperature range not configured"
    apply current profile idle_action

else if any configured temperature sensor is invalid:
    action = error
    apply current profile idle_action

else if heat demand exists and any active interlock is false:
    action = waiting_for
    expose waiting_reason
    apply current profile idle_action

else if heat demand exists:
    apply current profile heat_action
    wait feedback.settle_time

    if expected heating outputs are confirmed:
        action = heating
    else:
        action = error
        expose fault_reason

else:
    apply current profile idle_action
    wait feedback.settle_time

    if expected idle outputs are confirmed:
        action = idle
    else:
        action = error
        expose fault_reason
```

## Expected Home Assistant attributes

Example when heating:

```yaml
profile: "Balanced"
action: "heating"
target_temperature_low: 55.0
target_temperature_high: 65.0
current_temperature: 52.7
temperature_calculation: "average"
active_interlocks: []
waiting_reasons: []
fault_reasons: []
active_outputs:
  - primary_heater_stage
  - assist_heater_stage
```

Example when waiting:

```yaml
profile: "Fast Recovery"
action: "waiting_for"
waiting_reasons:
  - "Waiting for high load permission"
```

Example when in fault:

```yaml
profile: "Balanced"
action: "error"
fault_reasons:
  - "Assist heater stage did not start"
```

## Naming glossary

| Term | Meaning |
|---|---|
| `profile` | User-defined heating strategy |
| `runtime action` | Internal controller state exposed to Home Assistant |
| `temperature sensor` | Measurement input used for control/display |
| `monitored output` | A physical heater stage that can be checked by feedback |
| `heat_action` / `idle_action` | ESPHome actions that command relay/switch outputs |
| `feedback` | Real-world readback confirming the physical stage state |
| `interlock` | Temporary condition that blocks heating |
| `waiting_reason` | Human-readable reason for `waiting_for` |
| `fault_reason` | Human-readable reason for `error` |
| `expected_outputs` | Required ON/OFF state of heater stages for a profile |
