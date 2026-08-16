# Weekly HVAC Scheduling Design

## Overview

This document describes the deterministic scheduling layer used by the Home Assistant HVAC control system.

The schedule is intentionally separated from presence-aware automation. Scheduler Component defines the expected daily temperature transitions, while presence logic handles deviations from the normal weekday routine.

## Design goals

* Maintain consistent occupied and sleeping temperatures.
* Reduce daytime cooling when the home is expected to be unoccupied.
* Preserve different Friday, Saturday, and Sunday routines.
* Keep predictable scheduling separate from dynamic presence handling.
* Allow Home Assistant to remain the primary scheduling interface while Alarm.com remains the transport path to the physical thermostat.

## Schedule model

The weekly schedule is divided into four logical profiles rather than using a single seven-day schedule with exceptions.

| Profile | Days | Purpose |
|---|---|---|
| Workdays | Monday through Thursday | Standard workday routine |
| Friday | Friday | Standard workday with later bedtime |
| Saturday | Saturday | Weekend routine with later wake and bedtime |
| Sunday | Sunday | Weekend wake time with normal bedtime |

## Temperature targets

| State | Cooling target |
|---|---:|
| Sleep | 69 F |
| Comfort | 72 F |
| Away | 78 F |

These values are configuration choices for this implementation, not universal recommendations.

## Workdays, Monday through Thursday

| Time | Target | Intent |
|---|---:|---|
| 12:00 AM | 69 F | Sleeping |
| 4:30 AM | 72 F | Wake period |
| 6:15 AM | 78 F | Expected absence |
| 3:30 PM | 72 F | Expected return home |
| 9:30 PM | 69 F | Sleep |

## Friday

| Time | Target | Intent |
|---|---:|---|
| 12:00 AM | 69 F | Sleeping |
| 4:30 AM | 72 F | Wake period |
| 6:15 AM | 78 F | Expected absence |
| 3:30 PM | 72 F | Expected return home |
| 10:30 PM | 69 F | Later sleep period |

## Saturday

| Time | Target | Intent |
|---|---:|---|
| 12:00 AM | 69 F | Sleeping |
| 6:30 AM | 72 F | Wake period |
| 10:30 PM | 69 F | Sleep |

## Sunday

| Time | Target | Intent |
|---|---:|---|
| 12:00 AM | 69 F | Sleeping |
| 6:30 AM | 72 F | Wake period |
| 9:30 PM | 69 F | Sleep |

## Interaction with presence-aware control

The schedule represents the expected routine. Presence-aware automation modifies daytime behavior when actual occupancy differs from the expected routine.

During the weekday work window:

* If the resident leaves the neighborhood zone and remains away for 30 minutes, the thermostat is set to the Away target.
* If the resident re-enters the neighborhood zone, the thermostat immediately returns to the Comfort target.
* Short trips that do not exceed the absence threshold do not change the setpoint.

This separation prevents the deterministic schedule from accumulating one-off exceptions for work-from-home days, errands, early returns, or other irregular events.

## Control flow

```text
Weekly Scheduler
      |
      +--> expected time transition
      |
      v
Versatile Thermostat
      |
      v
Alarm.com
      |
      v
Physical thermostat

Presence automation
      |
      +--> temporary daytime correction
      |
      +------------------------------> Versatile Thermostat
```

## Operational precedence

The system uses a simple last-command-wins model at the climate entity. Scheduler and presence automation can both set the target temperature, but they operate in intentionally different contexts.

Scheduler provides the baseline state at known transition times. Presence automation provides corrections during the weekday work window when actual occupancy differs from that baseline.

The design avoids running a separate cloud schedule in Alarm.com at the same time. This prevents competing schedulers from repeatedly overwriting each other's target temperatures.

## Validation

Validation should confirm that:

1. Each scheduled transition changes the Versatile Thermostat target at the expected time.
2. The underlying Alarm.com climate entity receives the same target.
3. The physical thermostat reflects the command.
4. Presence corrections operate only during the defined weekday work window.
5. Short departures do not trigger the Away target.
6. Evening, overnight, and weekend schedules remain unaffected by presence automation.

## Rollback

To roll back the scheduling layer:

1. Disable the Home Assistant Scheduler entries.
2. Restore the previous thermostat schedule in Alarm.com if desired.
3. Leave Versatile Thermostat and the Alarm.com integration in place if manual Home Assistant control should remain available.

Presence automation can be disabled independently of the schedule layer.
