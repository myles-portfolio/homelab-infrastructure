# Command Center Dashboard

## Overview

The Command Center is a YAML-managed Home Assistant dashboard designed as an operational interface rather than a generic collection of entity cards.

The live dashboard uses a three-column layout that groups information by purpose:

* quick actions and home state
* climate, time, and weather
* energy production, storage, consumption, and grid activity

The public repository contains a sanitized representation of this design. Live household identifiers, entity names, device naming conventions, exact locations, and other sensitive values are replaced with generic examples.

## Design goals

The dashboard is intended to provide useful information within a few seconds of opening it.

Primary goals include:

* make frequently used actions immediately accessible
* surface abnormal states visually
* group related data into functional domains
* preserve useful historical context without overwhelming the overview
* maintain a consistent dark visual language
* use color to communicate state rather than decoration alone
* remain maintainable as YAML and version-controlled documentation

## Layout model

```text
Quick Actions          Climate and Time          Energy

Routines               Clock                     Current power
At a Glance            Thermostat                Battery state
Home Status            Indoor environment        Daily energy
Home Controls          Weather                   Energy distribution
                       Climate trends             Power trends
                       HVAC schedule
```

The dashboard uses Home Assistant Sections rather than relying on free-form card placement. This provides a predictable responsive grid while keeping the desktop layout visually consistent.

## Frontend components

The dashboard uses a combination of native Home Assistant cards and HACS frontend components.

### Mushroom

Used for compact state and climate cards with consistent typography and icon treatment.

### button-card

Used for highly styled action and summary cards that need more control over layout, state-driven backgrounds, labels, and icon presentation.

### card-mod

Used to apply consistent card surfaces, borders, transparency, and rounded corners where native styling is insufficient.

### ApexCharts Card

Used for time-series visualization such as indoor environmental trends and energy flow history.

### Scheduler Card

Used to expose the active climate scheduling model without requiring navigation into the scheduler configuration interface.

## Visual language

The live dashboard uses a custom dark theme with:

* near-black background surfaces
* translucent cards over a subtle astronomical image
* 18 px card radius
* low-contrast borders
* minimal shadows
* state-aware accent colors

Typical semantic colors include:

| State | Visual treatment |
|---|---|
| Normal / closed / available | green or muted neutral |
| Active lighting | warm amber |
| Cooling / informational | blue or cyan |
| Warning / transitional | amber or orange |
| Open / fault / abnormal | red |
| Secondary routines | purple |

## Information architecture

### Quick Actions

Provides direct access to frequently used household routines such as morning, departure, arrival, and sleep modes.

### At a Glance

Surfaces lightweight context such as dusk, dawn, and grid state without competing with deeper monitoring cards.

### Home Status

Shows presence, security state, work-from-home mode, door state, and garage state.

### Climate

Combines the current time, thermostat control, indoor and outdoor environmental measurements, weather forecast, trend history, and climate schedule.

### Energy

Separates instantaneous power from cumulative and daily energy data.

Instantaneous values answer what is happening now, while utility-meter-derived daily values answer what has happened during the current day.

Examples include:

* solar production
* household load
* grid use
* battery activity
* state of charge
* daily solar generation
* daily household consumption
* grid import and export
* battery charge and discharge energy

## Sanitization

The public dashboard configuration should replace or remove:

* real person entities
* precise household names or addresses
* device-specific identifiers
* internal naming conventions that reveal infrastructure details
* alarm system identifiers
* real image media IDs
* private URLs or endpoints
* exact location or zone information

Generic entity names should communicate purpose, for example:

```yaml
entity: person.resident
```

instead of a real household member entity.

Similarly:

```yaml
entity: sensor.solar_production
```

is preferred over vendor-specific or live device naming in public examples.

## Publication model

The repository version is intended to demonstrate dashboard engineering patterns and information architecture. It is not designed to be copied directly into another Home Assistant environment without entity mapping and installation of the required custom cards.
