# ESPHome Boiler Controller

Profile-based dual-point water heater controller concept for ESPHome and Home Assistant.

> Status: Design proposal / proof-of-concept planning  
> This component is not implemented yet.

## Goal

This project proposes a new ESPHome `water_heater` platform for real boiler and electric water-heater control.

The proposed controller is designed for systems that need:

- heat-only dual-point control
- user-defined operating profiles
- multiple temperature sensors
- physical output feedback
- interlocks / permissive conditions
- clear runtime actions for Home Assistant

## Proposed ESPHome platform

```yaml
water_heater:
  - platform: profile_dual_point
    name: "Boiler"
```

## Core idea

The controller separates concepts that are often mixed together:

| Concept | Meaning |
|---|---|
| Profile | User-defined heating strategy, such as Economy, Balanced, Fast Recovery |
| Action | Runtime state: heating, idle, waiting_for, error, off |
| Temperature | One or more tank sensors used for control and display |
| Interlock | Temporary condition that blocks heating but is not a fault |
| Feedback | Real-world confirmation that a heater stage actually started or stopped |
| Expected outputs | The required ON/OFF state of heater stages during heating and idle |

## Runtime actions

The controller exposes a fixed set of runtime actions:

| Action | Meaning |
|---|---|
| `heating` | Heating is required and the physical outputs are confirmed active |
| `idle` | The controller is enabled but no heating is currently required |
| `waiting_for` | Heating is required but blocked by an interlock or missing configuration |
| `error` | A real fault exists, such as invalid sensor data or failed output feedback |
| `off` | The water heater is disabled by the user |

## Profiles are user-defined

The component does not provide built-in modes such as ECO, Normal, or Performance.

Instead, users define their own profiles:

```yaml
profiles:
  - id: economy_profile
    name: "Economy"
    icon: mdi:leaf

  - id: balanced_profile
    name: "Balanced"
    icon: mdi:water-boiler

  - id: fast_recovery_profile
    name: "Fast Recovery"
    icon: mdi:flash
```

## Why this project exists

The existing `water_heater` domain is appropriate for representing boilers and water heaters, but current ESPHome water heater platforms do not provide full thermostat-like boiler control.

The `climate` thermostat platform provides stronger control behavior, but it uses the `climate` domain rather than the `water_heater` domain.

This project explores a controller that keeps the correct `water_heater` domain while adding boiler-specific control logic.

## Documentation

| Document | Purpose |
|---|---|
| [`docs/problem-statement.md`](docs/problem-statement.md) | Explains the gap this project is trying to solve |
| [`docs/yaml-api-concept.md`](docs/yaml-api-concept.md) | Describes the proposed YAML API |
| [`docs/state-machine.md`](docs/state-machine.md) | Defines the runtime controller state machine |
| [`docs/implementation-roadmap.md`](docs/implementation-roadmap.md) | Outlines the path from concept to external component and upstream proposals |

## Examples

| Example | Purpose |
|---|---|
| [`examples/minimal.yaml`](examples/minimal.yaml) | Minimal one-sensor / one-stage boiler concept |
| [`examples/advanced-multi-stage-boiler.yaml`](examples/advanced-multi-stage-boiler.yaml) | Advanced multi-sensor / multi-stage boiler concept |

## Planned features

- Profile-based boiler control
- Heat-only dual-point target range
- Multiple temperature sensor support
- Average / minimum / maximum / primary sensor calculation
- Interlocks with waiting reasons
- Physical output feedback validation
- Expected ON/OFF output states per profile
- Manual or automatic fault reset policy
- Home Assistant-friendly state attributes
- ESPHome external component proof of concept

## Repository status

Current milestone:

```text
v0.1-design
```

This milestone focuses on:

- problem definition
- YAML API concept
- state machine
- minimal example
- advanced example
- implementation roadmap

The next milestone will be:

```text
v0.2-poc-schema
```

This milestone will focus on an ESPHome external component skeleton and YAML schema validation.

## Roadmap

1. Define the YAML API concept
2. Define the controller state machine
3. Create minimal and advanced examples
4. Build an ESPHome external component proof of concept
5. Open an ESPHome feature request
6. Propose Home Assistant water_heater domain improvements
7. Prepare official ESPHome and Home Assistant pull requests if accepted

## Safety notice

This project is intended for research, design discussion, and controlled testing.

Water heaters and electric boilers can involve high voltage, high current, pressure, and hot water. Any real installation must use proper electrical protection, certified components, thermal safety limits, and local safety regulations.

This controller concept should not replace mechanical thermostats, thermal cutoffs, pressure relief valves, or other required safety devices.

## License

MIT
