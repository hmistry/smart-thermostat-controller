# Smart Thermostat Controller: Product Requirements Document (PRD)

- Author: Hiren Mistry
- Created: 12/01/2025
- Updated: 12/26/2025

## Problem

Forced-air HVAC systems (HVAC - Heating, Ventilation, and Air-Conditioning) are very popular in the US, but they are commonly plagued with poor thermal distribution, especially in residential homes. Different rooms and floors end up at varying temperatures, leaving the residents in either cold, medium, or hot spaces. Sometimes it even causes thermostat wars between residents. We won't go into the pros/cons of forced-air HVAC compared to hydronic or other HVAC systems but focus on building a solution to improve the thermal distribution with the existing HVAC system.

We know some of the primary reasons for poor HVAC performance are poor thermostat locations, uneven hot/cold air delivery, and a lack of system adaptability to different conditions. Breaking it down, the issues are:

- Limited number of thermostats - typically 1 per house or 1 per zone.
- Thermostat location - not measuring at the right places e.g. it's in a cold hallway, but rooms are hot.
- Air ducts and vents do not dynamically control or divert hot/cold air flow to the right areas.
- HVAC systems typically has a single high-speed or 2-speed blower, which constrains how many vents need to stay open to avoid pressurizing the ducts and stressing the blower.
- Open or closed doors and windows greatly impact airflow and the effectiveness of heating or cooling the rooms.
- Varying levels of solar heat gain and air leakages/penetration in rooms impact the temperature.
- HVAC system components lack the ability for finer controls by measuring rooms in different locations and directing just the right amount of heated/cooled air to those areas.

## Context

A number of companies and DIY'ers over the years have tried to address this problem by making smart vents, zone controllers, and smart thermostats with room sensing. The smart vents never took off in the market, most likely because of installation costs and a lack of good integration with thermostats and HVAC systems. Zoning is done using zone controllers but requires an HVAC tech to redesign the duct system and can be costly to upgrade. Zoning can help in some scenarios, but rooms within a zone may still have uneven heat distribution due to thermostat location and air delivery. Smart thermostats with room sensing solve the problem of measuring temperature in the right locations but are costly, and their effectiveness is still limited by the algorithm used (typically averaging the room sensors), zoning in the house, and air delivery to the appropriate places. Ecobee is the leading smart thermostat with room sensing and offers either averaging temperature across sensors or using temperature sensors where motion is detected.

## Solution

Design a low-cost, scalable room-sensing thermostat and HVAC controller whose algorithm/rules can be customized to optimize an HVAC system depending on the home and residents' preferences. There can be limitations to achieving ideal thermal distribution due to the lack of fine control in the HVAC system components and the home, but the goal is to improve as much as we can with those limitations over existing solutions in the market. Having the ability to log various performance data will make it easier to optimize the system over time. If the system can integrate with the smart home network, then we can use existing smart home sensors for presence/temperature signals, thereby reducing duplication of sensors and cost to the homeowner.

## Use Cases

Here are residential use cases and which are targeted cases to solve for.

| Floors | Rooms | Target Use Case | Notes |
|--------|-------|-----------------|-------|
| 1   | 1-2 Bedrooms + Kitchen + Living | No | Overall area is small, and the system should be able to heat the home with acceptable variation. However, they may benefit from our solution but are not our main target. |
| 1   | >=3 Bedrooms + Kitchen + Dining + Living | Yes | May have zones or can benefit from shifting target temperatures in different sections of the house depending on time/occupancy/heating distribution. |
| >=2 | >=2 Bedrooms + Kitchen + Dining + Living | Yes | - May have zones (assume by floors) or can benefit from shifting target temperatures in different sections of the house depending on time/occupancy/heating distribution. <br> - Downstairs heat could rise and accumulate in upper floor hallways. |

## Value Proposition

The Smart Thermostat Controller solution's key value propositions are:

- Low cost (target < $200 for thermostat with zones and up to 6 room sensors).
- Integrated room sensing and zone controller.
- Customizable algorithm and automation rules.
- Smart home integration.
- Data logging to see heating/cooling patterns and optimize the system for thermal comfort and/or energy consumption.

## Goals / Success Criteria

Success is delivering a solution that will make a home more comfortable temperature-wise for the residents at a lower cost than any solution on the market.

## User Experience

**DIY Installer:**

- Install the hardware to interface with the HVAC furnace or zone controller.
- Install and set up a portal or Home Automation platform if required.
- Pair room temperature sensors to the appropriate hardware.
- Configure hardware using the portal with target temperatures, zones, algorithm, etc.
- Review historical performance data in the portal and adjust target temperatures, algorithm, automation, etc.

**User:**

Using the portal, I can:

- View the status of the HVAC system and historical performance data.
- Modify target temperatures, schedules, and automation.

## Competitor Analysis

Cost models for how much current market solutions would cost a homeowner:

- 1 story, 3 bedrooms.
- 2 story, 2 bedrooms (2nd floor), kitchen/living (1st floor), single zone.
- 2 story, 4 bedrooms (2nd floor), kitchen/living (1st floor), 2 zones by floor.
- Honeywell Zone Controller costs $150.

| Thermostat | Room Sensors | Temperature Read Algorithm | Cost (1s/3b) | Cost (2s/2b) | Cost (2s/2z/4b) |
|------------|--------------|----------------------------|--------------|--------------|-----------------|
| [Ecobee][ecobeeThermostat] | - Yes, up to 32, with motion <br> - $50/sensor | - Average of participating sensors in a mode <br> - Create modes and schedule modes, e.g. home, away, sleep <br> - Occupancy takes a few minutes to assume adding/removing a room in the algorithm to prevent false triggers for short activity | $150 + $50*2 = **$250** | $150 + $50*3 = **$300** | 2*$150 + $50*5 + $150 = **$700** |
| [Nest][nestThermostat]   | - Yes, up to 6/thermostat & 18/home <br> - $40/sensor | - Set target sensor on a schedule <br> - Takes average if multiple sensors | $150 + $40*2 = **$230** | $150 + $40*3 = **$270** | 2*$150 + $40*5 + $150 = **$650** |
| [Amazon][amazonThermostat] | - Yes, using Amazon Air Quality Sensor or Amazon Dot, up to 10 sensors <br> - $70/AQ sensor or $50/Dot | - Uses average of participating sensors in a mode <br> - Can use Alexa routines to add custom rules to include/exclude sensors based on time | $70 + $50*2 = **$170** | $70 + $50*3 = **$220** | 2*$70 + $50*5 + $150 = **$540** |

## Key Features and Roadmap

### v1.0

- Target my use case - single-speed blower HVAC with dual zones.
- Working prototype (MVP room sensing and interface to or integrated zone control with logging).
- Optimizing rules through an existing Home Automation platform, e.g. Home Assistant.

### v1.1

- Integrate zone controller.

### v1.2

- Integrate weather and other sensor integration, e.g. motion, presence.
- Alerts.

### v1.3

- Control interface panel with display (hardware).
- Work with other Home automation OS, e.g. Hubitat.

## User Stories and Requirements

| Scenario                | User Story |
|-------------------------| ---------- |
| Installation            | As a DIY user, I want to easily install the thermostat/controller hardware, similar to installing any thermostat and zone controller. I want to easily set up a Home Automation platform if required. |
| Setup                   | As a DIY user, I want to configure the room sensors, thermostat, and zone controller. I want to set up automations and schedules on the Home Automation platform. |
| Change temperature setting | As a user, I want to change the current temperature targets and/or schedule. |
| Review performance data | As a user and DIY user, I want to review the performance data, e.g. room temperatures, HVAC on/off times, zones on/off, etc. |
| Turn on/off             | As a user, I want to be able to turn on or off the HVAC. |
| Alerts                  | As a user, I want to be notified if there is an error or issue. |

## Out of Scope

For MVP:

1. Physical controller with display.
1. Any high-effort, complex integrations that aren't necessary for the core functionality as described in v1.0.

[ecobeeThermostat]: https://www.ecobee.com/
[nestThermostat]: https://store.google.com/us/category/nest_thermostats
[amazonThermostat]: https://www.amazon.com/Amazon-Smart-Thermostat/dp/B08J4C8871