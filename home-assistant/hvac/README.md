# Presence-Aware HVAC Control

## Overview

This case study documents a Home Assistant based HVAC control design that combines predictable weekly scheduling with dynamic presence handling.

The physical thermostat remains connected through Alarm.com. Home Assistant uses Versatile Thermostat as the control layer and Scheduler Component for routine temperature transitions.

## Design goals

* Maintain a comfortable occupied temperature.
* Raise the cooling setpoint during longer weekday absences.
* Avoid changing the HVAC setpoint for short errands or neighborhood walks.
* Restore the comfort setpoint before the resident reaches home.
* Preserve deterministic morning, evening, and overnight schedules.
* Keep the implementation understandable and easy to roll back.

## Architecture

```text
Phone location
    |
    v
Home Assistant person entity
    |
    v
Neighborhood zone
    |
    +--> inside / re-entered ---------> 72 F
    |
    +--> outside continuously 30 min -> 78 F
                                      |
Weekly Scheduler ---------------------+
                                      v
                             Versatile Thermostat
                                      |
                                      v
                                  Alarm.com
                                      |
                                      v
                            Honeywell thermostat
```

## Control strategy

The fixed Scheduler configuration remains responsible for normal daily transitions such as wake, away, return home, and bedtime targets.

The presence automation only operates during the normal weekday work window. A neighborhood-sized zone is used instead of the smaller Home zone so routine dog walks and nearby activity do not immediately indicate a meaningful absence.

A 30 minute continuous absence requirement acts as a debounce mechanism. If the resident returns to the neighborhood before 30 minutes elapse, the away transition never executes.

When the resident re-enters the neighborhood during the work window, the thermostat immediately returns to the comfort target.

## Example operating states

| Situation | Result |
|---|---|
| Normal weekday commute | 78 F after 30 minutes outside the neighborhood |
| Unexpected work-from-home day | Remains at the comfort target |
| Short store trip | No change if absence is under 30 minutes |
| Neighborhood dog walk | No change while within the neighborhood zone |
| Early return home | 72 F when the neighborhood zone is re-entered |
| Evening or weekend departure | Presence automation does not run |

## Validation

The control path was validated end to end by confirming a Home Assistant schedule change propagated through Versatile Thermostat, Alarm.com, and the physical thermostat.

The presence automation should additionally be reviewed through Home Assistant automation traces after real-world use to verify zone accuracy and absence timing.

## Security notes

The public automation example uses generic person and zone identifiers. Exact geofence boundaries, account information, private endpoints, and precise location data are intentionally excluded.

## Rollback

Disable the presence automation to return to fixed Scheduler behavior. Versatile Thermostat and the underlying Alarm.com climate entity continue to operate independently of the presence automation.
