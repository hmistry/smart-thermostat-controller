# Why Smart Thermostats Still Struggle — and How Room-Level Sensing Finally Fixed My Home

Most residential forced-air HVAC systems struggle with uneven heating and cooling — not because of bad equipment, but because they lack the sensing and control needed to adapt to real-world conditions inside a home.

After years of dealing with hot and cold rooms in a home with two-zone forced-air HVAC system, I built a custom room-sensing thermostat controller to address the root causes with a budget-friendly cost structure. After ten months of daily use, the home has been noticeably more comfortable, with only a modest increase in energy usage.

This post explains why most thermostat designs fall short and what worked better in practice.

## Problem: Thermal airflow, poor sensing, and no dynamic control

#### Scenario

![HVAC House Heat Flow Example][hvacHouseHeatFlow]

Consider above example of a two-zone home:

- One thermostat controls the downstairs (T1)
- Another controls the upstairs (T2)
- Zone dampers control hot/cold airflow between zones
- A single HVAC unit serves both zones
- One-sided solar heat gain

Factors contributing to uneven heat distribution in this example are:

1. Open floor plan zones lose effectiveness as heated or cooled air quickly flows to the return vent, limiting heat transfer to the space.
2. One-sided solar heat gain creates uneven temperatures within a zone, making control decisions ambiguous.
3. Inter-zone thermal coupling causes heating in one zone to affect thermostats in another (i.e. heating downstairs affects T2 thermostat).
4. Poor thermostat placement results in measurements that do not reflect actual living spaces.
5. Room door and window states vary constantly, changing the room's airflow and heat loss.
6. Vent adjustments are static and cannot adapt to weather, occupancy, or airflow changes.
7. Room-sensor averaging can mask cold rooms, delaying heating.
8. Lack of historical data makes it difficult to understand thermal behavior in the home and tune the HVAC system.

## Design Solution: Control Where Comfort is Actually Needed

The key insight was simple:

1. Thermostat control decisions should be based on room-level temperature data, not hallway averages. In practice, temperature does not equalize quickly enough to rely on averaging.
2. The thermostat control strategy must be customizable and allow algorithms beyond simple averaging.

#### Architecture

![HVAC Design Solution][hvacDesignSolution]

The solution consists of:

- Low-cost Bluetooth temperature sensors placed in each room
- Home Assistant running on a Raspberry Pi 5 handles all the control logic
- An ESP32-based HVAC zone controller managing dampers, HVAC furnace, and A/C Condenser
- Custom algorithms to optimize heating and cooling for comfort

How it works:

- Bluetooth Temperature sensors can be placed in ideal locations in each room, sending temperature and humidity data to Home Assistant
- The ESP32 HVAC Zone Controller is connected to zone dampers, HVAC furnace, and A/C Condenser. 
- HVAC Zone Controller is setup to run the following:
  - Cycles: Off, Heat, Cool, Fan, Purge
  - Zones: All, Upstairs, Downstairs
  - Safe operations: ensure correct sequencing and timing requirements between cycles and zones are met
- Created a HVAC Dashboard in Home Assistant with all the necessary thermostat controls and data per zone.
- Use automations to ensure selected room temperatures in a zone stay within a range, e.g. 70-73°F during the winter (heat only) and 80-83°F in the summer (cool only). Instead of averaging room temperatures, the automation evaluates the minimum and maximum room temperatures in a zone and calls for heating or cooling when any prioritized room moves below or above the defined range respectively*.

**There are some edge cases that require additional logic to the algorithm. However, for the purposes of this article, this is the core strategy.*

#### Hardware

![Xiaomi Sensors + Raspberry Pi + ESP32 Controller][xiaomiRpiEsp]

#### Home Assistant Thermostat UI

![HA Thermostats][haThermostats]

## Results

#### Room Temperature Charts

![HA HVAC Temperature Charts][haTempCharts]

#### HVAC Runtime Charts

![HA HVAC Usage Charts][haUsageCharts]

I implemented the above solution with 9 Temperature sensors including 1 outside and it's been running for 10 months now. Here's the results so far:

1. Much Better Comfort
   - Prioritized room temperature variations dropped from several degrees to roughly 3°F during regulated periods. Cold and hot bedrooms largely disappeared.
   - Sometimes large temperature differences between rooms from solar heat gain occur that require some user intervention (i.e. close/open room doors & windows) to help close the gap quicker and allow the system to operate normally for rest of the night.
2. Inter-Zone Coupling Eliminated
   - Heating downstairs no longer interferes with upstairs heating. Bedroom temperatures now drive decisions instead of hallway temperature.
3. Data-Driven HVAC Tuning
   - Temperature data helped identify airflow imbalances and adjusted vents accordingly
   - Runtime data helped optimize energy consumption by heating one zone at a time
4. Stable HVAC Operation
   - No short cycling of HVAC furnace or A/C condenser
   - Purge cycles happen after each run
   - Correct sequencing, timing, and safeguards enforced
5. Modest Energy Impact
   - Energy usage increased by approximately 10–20%, but comfort improved significantly. For the first time, the home maintained consistent, comfortable temperatures throughout the day.
6. Cheaper than commercial solutions
   - 9 Temperature sensors ($36) + Raspberry Pi 5 ($140) + ESP32 x8 Relay Board ($44) = $220
   - Compared to an Ecobee (2x$120) + 6 Room sensors ($240) + Zone Controller ($150) = $630
   - Adding temperature sensors is cheap at $4 each

## Conclusion

Many of the comfort issues I had lived with for years—hot and cold rooms, overheated hallways, and slow recovery from afternoon solar heat gain—turned out to be artifacts of poor sensing and overly simple control logic. I was surprised by how effective room-level sensing and a window-based control strategy turned out to be, and the experience has changed how I think about residential HVAC systems. The biggest gains did not come from adding AI, machine learning, or fancy hardware, but from giving the system better inputs and using the right control logic to respond to real conditions instead of averaged data. The secondary benefit is a better cost structure by unifying the controller and for scaling sensing in a home.

This solution is not a silver bullet. It still requires occasional user intervention when there are large temperature differences between rooms, as well as some technical knowledge to build and maintain. But after ten months of use, it has delivered a level of comfort that commercial thermostats never achieved in my home, and it reinforces a simple idea: comfort improves when control decisions are made closer to where people actually live.

## References

- For in-depth details on implementation and results, see [Smart Thermostat Controller Design and Results][stcDesignGithubLink]
- Other project information is available in [Smart Thermostat Controller Github repository][stcGithubLink]

[stcGithubLink]: https://github.com/hmistry/smart-thermostat-controller
[stcDesignGithubLink]: https://github.com/hmistry/smart-thermostat-controller/blob/main/smart_thermostat_controller_design.md

[hvacHouseHeatFlow]: <images/HVAC-Home-Heat-Flow.png>
[hvacDesignSolution]: <images/HVAC-Architecture-Proposed.png>

[xiaomiRpiEsp]: <images/Xiaomi-RPi-ESP.jpg>

[haTempCharts]: <images/HA-Charts-Temp.png>
[haThermostats]: <images/HA-Thermostats.png>
[haUsageCharts]: <images/HA-Usage-Charts.png>
