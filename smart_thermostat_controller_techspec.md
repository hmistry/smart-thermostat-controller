# Smart Thermostat Controller: Technical Specification

- Author: Hiren Mistry
- Created: 12/01/2025
- Updated: 12/26/2025

*Disclaimer: I'm no expert. The information in this repository is based on my research and what I built to solve the problem I had. It does not express or imply you should do the same nor guarantees it will solve your HVAC issues.  It is provided "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED as stated in the [License][license].*

## Overview

The objective is to develop a lower-cost, scalable room-sensing thermostat and HVAC controller whose algorithm/rules can be customized to optimize an HVAC system to the home’s enviroment and the residents’ comfort. This solution aims to significantly reduce thermal variations and enhance comfort within the constraints of the HVAC system and the home.

### Context

The [Product Requirements Document (PRD)][prdFile] delves into the challenges of poor heating and cooling performance in homes, examining current HVAC solutions and the obstacles in achieving comfortable thermal distribution within the homes. Based on research conducted in the PRD, current market solutions can range from $170 to $700 for a multiple room-monitoring solution over a single thermostat control.

Here's an example scenario to illustrate the causes of poor heating/cooling distribution in homes:

#### Example Setup

![HVAC House Heat Flow Example][hvacHouseHeatFlow]

Consider a home with a two-zone HVAC system, controlling the upstairs and downstairs zones. Each zone has its own thermostat: T1 for the downstairs and T2 for the upstairs. Initially, the zone dampers are set to the “off” position (open). When a zone requests heat, the corresponding damper is opened (off), while the other damper is closed (on). If both zones need heat, both dampers are opened.

Here are some notable characteristics of the home:

- The left side receives more solar heat gain than the right side.
- The right side experiences more cool breezes than the left side.
- The downstairs floor is an open floor plan, allowing free flow of air between the kitchen, hallway, living room, and upstairs hallway via the staircase.
- There’s a single large return air vent in the upstairs hallway leading to the HVAC unit.
- Closing the bedroom doors can restrict airflow from upstairs rooms to the upstairs hallway.
- The temperature of the heated air at the vents typically ranges from 85°F to 140°F, which mixes with the room air to heat the rooms. Similarly, the cooling air at the vents is usually targeted to be 14-20°F below the return air being cooled.

#### What factors contribute to uneven heat distribution in this example?

1. Heating an open floor plan zone: Heating the downstairs area loses some effectiveness because the air quickly flows through to the upstairs hallway and recirculates, limiting heat transfer to objects and walls downstairs that are away from the airflow path.

2. One-sided solar heat gain: If the kitchen is warmer due to afternoon solar heat gain and the living room is cold, heating the downstairs area will heat all areas because there’s no way to shut off the kitchen and only heat the living room. As a result, the temperature measured by the thermostat T1 will be somewhere in between the kitchen and the living room. HVAC system designers claim that the heat will eventually even out, but I’ve found that it can take several hours, leaving you to deal with hot and cold areas. Sometimes, the rooms won’t even out by the next day when the solar heating starts, resulting in the living room always being cooler than the kitchen.

3. Inter-zone thermal coupling: Heating the downstairs zone causes warm air to rise into the upstairs hallway to the return duct, warming thermostat T2. This means thermostat T2 is no longer representative of the temperature in the bedrooms. As a result, the bedrooms will cool further below the desired range until the upstairs hallway cools down and thermostat T2 calls for heat.

4. Thermostat/temperature sensor location: The location of the thermostat and temperature sensors (I’ll refer to both as sensors in this bullet point) is crucial to provide the system with the correct temperature data. If the sensors are located where the sun falls on them or are near a window (cool or warm breezes), the temperature measured will be different from the room, causing false heating, cooling, or turning off the HVAC system.

5. Room windows/doors: The upstairs zone also experiences a similar solar heat gain issue as the downstairs zone between bedrooms 1 and 2. However the temperature difference can be even greater if both room doors are closed, and the window in bedroom 1 is closed, trapping all the solar heat gain, while the window in bedroom 2 is open, allowing cold wind to blow into the room. To heat the upstairs zone in such a case, the user will need to open or close the bedroom windows and doors accordingly to heat the cold room and maintain the desired temperature in the warm room.

6. Averaging with room sensors: Most, if not all, solutions on the market use an average of room sensors (which may include the main thermostat). While this approach makes sense, it is not the most effective method. If the rooms are at different temperatures, and the average temperature is above the thermostat’s minimum target, the heater will not turn on to heat the room below the target range. The user will have to wait until the average temperature drops or intervene by opening the window in the warmer room for a short time to cool it down faster, triggering the heater to heat the zone. I found this process can take hours to stabilize large temperature differences and ensure consistent heating.

7. Closing vents to control heating: Partial closing of vents in the rooms can help even out the temperature distribution within a specific zone. However, this approach is more complex because the HVAC system designer has designed the system to deliver a specific volume of heated air per minute per room based on worst-case heat loss calculations for each room. In reality, I have observed that the amount of heated air flow required varies depending on weather conditions, the state of doors and windows, and the presence of heat-generating objects. This means in an ideal world the vents need more dynamic control. Furthermore, excessive airflow restriction by closing vents in a zone can create back pressure, potentially stressing the blower fan and causing the duct to blow.

These are some of the complex issues that contribute to uneven thermal distribution in homes using air-based HVAC systems. We will use this information to develop a more effective thermostat controller algorithm and solution to address these problems.

## Product/Technical Requirements

### In Scope

As per the [PRD][prdFile], our Smart Thermostat Controller's key value propositions are:

- Low cost: Targeted to be less than $200 for a thermostat with two zones and up to six room sensors.
- Scalable room sensing with remote temperature/humidity sensors
- Integrated zone controller
- Customizable controller algorithm and automation rules
- Smart Home platforms integration
- Data logging: Enables data logging to monitor heating/cooling patterns and optimize the system for thermal comfort and energy consumption.
- User interface: Provides a user-friendly interface for configuring, viewing settings, and reviewing/downloading data.

To meet the above product requirements, here are the key technical requirements for the solution:

1. Hardware Requirements

    1. Room Sensors

        - Low-cost temperature and humidity sensors.
        - Battery (battery life > 1 year) or USB-powered.
        - Wireless connectivity for easy installation.
        - Temperature resolution of 0.1°C and humidity of 1%.
        - Display (optional).

    2. Thermostat Unit

        - Interface with standard [4-5 wire (RWYGB) HVAC systems][hvacWiring] and zone controllers.
        - Supports RWYGB wire code: Red (R/Rc/Rh) is 24VAC (Rc for A/C condenser power, Rh for heater furnace power), White (W) is for turning on the heater, Yellow (Y) is for turning on the A/C condenser, Green (G) is for turning on the blower fan, and Black/Blue (B) is 24VAC Common (used to power thermostats).
        - Wireless connectivity to room sensors (supports up to 20 sensors).
        - User interface for installers and users to configure, control, and view the system’s state and settings.
        - Preferably powered by USB, DC power supply, or AC mains to reduce battery maintenance.
        - Built-in failsafes and timing requirements for HVAC systems.

    3. Zone Controller

        - Interface with standard 4-5 wire (RWYGB) HVAC systems and thermostats.
        - Relays need to support voltage (24VAC) and current (1A) ratings for HVAC systems and dampers.
        - Wireless connectivity to the main thermostat for easy installation location.
        - Built-in failsafes and timing requirements for HVAC systems and zone dampers.
        - Powered by USB, DC power supply, or AC mains, eliminating the need for a battery, which can be inconvenient to replace in locations like attics.
        - User interface designed for both installer and user configuration, control, and viewing of the state and settings.

    4. Thermostat and Zone Controller Failsafe and Timing requirements

        - A/C and heater cannot be running simultaneously.
        - Typically, A/C requires the blower fan to be on first, followed by the A/C after a specified delay.
        - When operating HVAC components to be on/off, delays need to be added and can vary from 10 seconds to minutes depending on the component and system. These delays are configured by the installer according to the manufacturer’s guidelines.
        - Gas heaters do not require the blower fan to be on as the HVAC Furnace controls the blower fan directly.
        - Blower fan must be turned off when turning on a gas heater to prevent issues with igniting the gas burner.
        - The system supports both normally open (NO) and normally closed (NC) Zone dampers. Normally open dampers are open when turned off, while normally closed dampers are closed when turned off. These settings are configured by the installer.
        - When switching dampers on and off, there should never be a state where all dampers are closed for an extended period, and the HVAC is running in any mode. This can lead to pressure buildup and potential duct damage.
        - The system should provide a configurable damper switch-settling delay for the installer to set, as dampers may take time to open or close. For example, my dampers take 10 seconds to open or close.
        - The system should have a configurable timeout of at least 2-3 minutes when switching from A/C to heater or heater to A/C (or a combination of both). This timeout can be used to purge the ducts and system after the heater or A/C turns off.
        - If the zone controller loses connection with the thermostat, it should gracefully turn off the HVAC and open the dampers unless it re-establishes connection with the thermostat.

2. Software Requirements

   - Runs on low-power IoT microcontroller platforms like ESP, Arduino, Raspberry Pi, and others, making it cost-effective.
   - Provides an interface to set up, view, and modify settings. UI can be web browser, application, or custom GUIs.
   - Supports wireless connectivity and over-the-air updates.
   - Has the capability to recover from power outages and wireless connection outages.
   - Enables data storage, charting, and export of various data points, such as temperature, humidity, and on/off times, for analysis and system optimization.
   - Supports smart home integration by supporting commercial platforms like HomeKit and Google Home, as well as open-source systems like Home Assistant and Hubitat.

### Out of Scope

Due to time and limited testing access, the following HVAC systems will not be supported in the MVP:

- Multi-stage gas furnaces and all-electric furnaces.
- Non-standard 4-5 wire (RWYGB) HVAC interfaces.
- Heat Pumps.
- Smart Home platforms other than HomeKit.

As per the PRD, we won’t prioritize features for version 1.2 and beyond for the MVP. These include:

- Weather compensation.
- Alerts.
- Multiple Smart Home platforms.

### Future Goals

Product and technical requirements for the future include:

- Out-of-scope features.
- Hydronic floor/ceiling heating/cooling systems.

## Solutions

### Current Solution Available

![HVAC Current System Design][hvacCurrentSystemDesign]

The diagram illustrates the wiring of typical current commercial solutions for forced-air HVAC systems. This particular system is designed for a two-zone system, which includes a second thermostat, zone controller, and two dampers to direct air to the selected zone. In contrast, a single-zone system only has a single thermostat connected directly to the heater furnace and A/C condenser (if present) or heat pump. The thermostat internally has relays that, depending on a heating or cooling request, connect the 24VAC on the Red wire to W for heating, G for fan, or Y+G for cooling (A/C condensers require the blower fan to be on). Additionally, there are safeguards to prevent turning on both the heater and A/C, along with some timing delays to ensure proper operation sequencing. Similarly, a zone controller uses relays to connect 24VAC to a power a zone damper there by closing the damper. When the power is disconnected from the zone damper, then the spring will open the damper.

Pros:

- Standard design in the HVAC industry, making it familiar to contractors for installation and repair.
- Widely available commercial products.
- Generally works to varying degrees of satisfaction.
- Some higher-end thermostats offer room temperature sensing.

Cons:

- All the issues discussed in the [Context][contextSection] section above.
- Multi-zone systems and room sensing are costly and have limited effectiveness.
- Algorithms used with room sensors are suboptimal, meaning they work well for smaller temperature variations across rooms but not as effectively for large or skewed variations.

### Proposed Solution Design

![HVAC Proposed Design Solution][hvacNewDesignSolution]

Based on my research of available hardware components, here's the proposed hardware and software solution:

1. Room Temperature Sensing

   - The [Xiaomi BT Temperature/humidity sensors (LYWSD03MMC)][xiaomiTempSensors] meet our technical requirements, including resolution, accuracy and Bluetooth wireless communication.
   - It has an average battery life of 2 years.
   - It has a clean, minimalist aesthetic, a display (which is a bonus), and is easily placed around the house.
   - Each sensor costs $4, making it highly cost-effective for scaling room sensing.
   - To pair the sensor with Home Assistant, the firmware will need to be updated to the [custom ATC_MiThermometer firmware][xiaomiCustomFirmware].
   - The sensor will send temperature and humidity data to Home Assistant.

2. Zone Controller

   - The [ESP32 Relay board (x8 relays)][espRelayBoard] meets our hardware requirements.
   - When combined with the [ESPHome platform][espHomePlatform], it simplifies development and has built-in OTA updates.
   - It has a total of 8 relays; 3 relays will be used to control a single-stage HVAC with an A/C condenser. The remaining 5 relays can be used to control up to 5 zone dampers.
   - The controller will be powered by a [12V DC power adapter][espPSU].
   - It handles all the logic and failsafes for controlling HVAC and zones.
   - The controller will have the following Cycle modes: `Off`, `Heat`, `Cool`, `Purge`, and `Fan`.
   - It will have the following Zone modes: `All`, `Upstairs` (Zone 1), and  `Downstairs` (Zone 2).
   - The `Purge` cycle will set the zone to `All` and run for X minutes (default 3 minutes) before automatically turning off.
   - In the `Off` cycle, the zone will be set to `All`.
   - On power up, the Cycle and Zone will default to `Off` and `All` respectively.
   - When switching from Zone 1 to Zone 2 or vice versa, the correct sequencing and timing will be observed to ensure that both zone dampers are never closed simultaneously.
   - The 8 relays will be wired as follows:
     - Relay 1: HVAC Heating
     - Relay 2: HVAC Cooling
     - Relay 3: HVAC Fan
     - Relay 4-6: Not used
     - Relay 7: Close Upstairs Zone Damper
     - Relay 8: Close Downstairs Zone Damper
   - Relay state per Zone modes:
     - Valid zones: `All`, `Upstairs`, `Downstairs`
     - `All`: Turn off relays 7 and 8
     - `Upstairs`: Turn on relay 8, turn off relay 7
     - `Downstairs`: Turn on relay 7, turn off relay 8
   - Relay state per Cycle modes:
     - Valid cycles: `Off`, `Heat`, `Cool`, `Purge`
     - `Off`: Set Zone to `All`, turn off relays 1-3
     - `Heat`: Turn on relay 1. Have a 5 minute heat-cool-lockout timeout before turning on when coming directly from `Cool` and ensure fan relay is turned off before turning on.
     - `Cool`: Turn on relay 3 for 30 seconds to start fan, then turn on relay 2. Have a 5 minute heat-cool-lockout timeout before turning on when coming directly from `Heat`.
     - `Purge`: Turn on relay 3 for 3 minutes, then switch to Off cycle mode
     - `Fan`: Turn on relay 3
  
3. Thermostat & Smart Home Platform

   - Selected the [Home Assistant (HA) platform][homeAssistantPlatform] as our platform of choice.
   - Offers the best zero-cost option for building custom integrations using [ESPHome][espHomePlatform] for the [ESP32 Relay board][espRelayBoard].
   - Integrates well with commercial Smart Home systems.
   - Provides a UI for configuring, viewing, and analyzing data via a mobile app, web browser, or a dedicated wall panel using a [tablet in kiosk mode][homeAssistantKiosk].
   - Manages all thermostat logic and interfaces with the ESP32 Zone Controller to set the appropriate HVAC cycle mode and zone.
   - Requires a Raspberry Pi to run, which adds to the cost.
   - Allows unification of all other smart home devices into a single platform for more hollistic custom automations.

4. Hardware cost breakdown

   - Cost models from PRD:
     - 2-story, 2 bedrooms (2nd floor), kitchen/living (1st floor), single zone.
     - 2-story, 4 bedrooms (2nd floor), kitchen/living (1st floor), 2 zones by floor.

   | Hardware Component | 1 Zone, 2 Story, 2 Bed | 2 Zones, 2 Story, 4 Bed |
   |--------------------|-----------------------------|-------------------------|
   | [Raspberry Pi 5 8GB][rpiCompute] | $82 | $82 |
   | [Raspberry Pi Heatsink][rpiHeatsink] + [27W USB-C PD Power Supply][rpiPSU] + [M.2 NVMe HAT][rpiNvmeHat] + [Raspberry Pi 5 Case][rpiCase] Bundle | $42 | $42 |
   | [128GB M.2 NVMe SSD][rpiSSD] | $15 | $15 |
   | [Xiaomi Temperature Sensors (LYWSD03MMC)][xiaomiTempSensors] | 3 x $4 = $12 | 6 x $4 = $24 |
   | [ESP32 x4 Relay board][espRelayBoardx4] | $18 | N/A |
   | [ESP32 x8 Relay board][espRelayBoard] | N/A | $19 |
   | [ESP32 12V DC Power Supply][espPSU] | $7 | $7 |
   | [24VAC 40VA HVAC Power Transformer (for Zone Dampers)][hvac24VacTransformer] | N/A | $18 |
   | Total Cost | $176 | $207 |

   - For the single zone, a 2-story/2-bedroom setup, the total cost of $176 beats our target of $200 and competitive solutions for 1-zone, 3-sensor setups costing $220 to $300.

   - For the 2-zone, a 2-story/4-bedroom setup, the total cost of $207 is slightly above the $200 target. However, competitive solutions for 2-zone, 6-sensor setups range from $540 to $700 (as per PRD), which translates to a significant savings of 60-70%.

   - If you plan to integrate other smart home devices into Home Assistant and build a comprehensive smart home system, consider getting the 16GB RPi. This upgrade will increase the overall cost by $44 and future-proofs the hardware as you install more add-ons.

5. Software Algorithm

   - To solve for the limitations of using average temperatures across selected rooms, we’ll adopt a temperature “window” approach. Each room temperature must fall within the specified range. Here’s how it works:
     - Calculate a composite temperature for all selected rooms in a zone. The composite temperature will be the minimum temperature when the minimum is below the target temperature, or the maximum temperature when the maximum is above the target temperature. For heating, it has a bias to take the minimum if a room is below the target temperature, while other rooms are above the target temperature. Conversely, for cooling, it will have a bias to take the maximum temperature.
     - Set a target temperature for heating and one for cooling, for instance, 71.5°F for heating and 81.0°F for cooling. Ensure that the gap between the heating and cooling target temperatures is wide enough to prevent any overlap.
     - Determine a window size around the target temperature that provides a comfortable temperature for the occupants, for example, a range of ±1.5°F.
     - If the composite temperature drops below the target temperature by 1.5°F, initiate heating. Conversely, when the composite temperature exceeds the target temperature by 1.5°F, stop heating. Have a similar but opposite process for cooling.
     - Implement a safeguard to prevent the room temperature from exceeding the maximum permissible temperature (which can be above the target range, for instance, 75°F). This safeguards against scenarios where there’s a significant difference between the minimum and maximum room temperatures (e.g., 70°F and 74°F). In such cases, the system should be able to heat the cooler room up to the target temperature or the warmer room to absolute maximum of 75°F, whichever comes first. However, if the temperature spread is too wide, the user will need to manually adjust the doors and windows to help equalize the room temperatures. For cooling, the opposite applies.
   - If the heating target and cooling target windows do not overlap, the system doesn’t need to switch between heat and cool modes like standard thermostats do. It can operate as a “set and forget” device, maintaining appropriate home temperatures throughout the year.

Pros:

- This proposed solution aligns perfectly with all the product goals, ranging from low cost to a fully customizable solution that includes data gathering and user interface (UI) capabilities with a very low development effort.
- It has the potential to be expanded to accommodate future product objectives outlined in [Future Goals][futureGoals].

Cons:

- In its current state, this prototype solution lacks consumer-friendliness and requires a tech-savvy individual with expertise in hardware and software to build and maintain it. While the average consumer can use it once it’s set up and running, they may still need assistance occasionally.
- There’s a possibility of experiencing uneven thermal distribution (edge case when there's a large min-max temperature difference) due to the inability of the HVAC system to direct air solely to specific rooms instead of an entire zone.

### Other Solutions

Other solutions that were considered include:

1. Integrating a complete Smart Thermostat and Zone Controller into the ESP32 Relay Board, thereby eliminating the need for a Raspberry Pi and Home Assistant.
   - Xaomi Temp Sensors can be directly connected to the ESP32 Board via Bluetooth. However, there’s a limit to the number of BT sensors that can be connected without data loss (use an ESP32 Bluetooth proxy to overcome this limitation).
   - Advantages:
     - Significantly cheaper (~$50) due to the absence of a Raspberry Pi hardware.
     - Simpler to productize for the average consumer.
   - Disadvantages:
     - Requires a substantial amount of development time to create a comprehensive solution, including UI, data storage, and Smart Home integration.

2. Using ESP32 with room sensors only to replace thermostats and interface with commercial Zone controllers in a similar manner to current thermostats.
   - This can be implemented with or without Home Assistant and Raspberry Pi trading off development time and cost.
   - This approach necessitates three relays for each thermostat/zone, which is not cost-effective beyond two zones.
   - Although keeping the standard Zone controller was considered for safety reasons, it appears redundant as the safeguards can easily be integrated into the ESP32 Relay Board.

3. Exploring other microcontrollers or custom hardware/solutions.
   - While this option is certainly viable, it was eliminated due to the increased engineering effort required to develop both hardware and software.
   - It’s worth considering once we have successfully developed prototypes and aim to create a real GTM (go-to-market) consumer product.

## Testability

The initial prototype will be designed to work with a single-speed HVAC heater with A/C and two zones, similar to the setup in the example house in [Context][contextSection]. Manual testing will be conducted to simulate various heating and cooling scenarios and error conditions, such as power or WiFi outages. Additionally, the system’s ability to cycle and switch zones correctly will be checked to ensure it meets the failsafe and timing requirements specified in [Product/Technical Requirements][prodTechRequirements].

## Observability & Alerting

Home Assistant has the capability to monitor the up & down times of WiFi and ESP32. It also offers the option to set up alerts using add-on services which is for a future enhancement.

## Success Metrics / Impact

The primary success metrics are:

- How effectively the system maintains a comfortable temperature throughout the home compared to previous thermostats and zone controllers.
- User satisfaction with the solution to setup and use.
- Whether the system consumes similar or lower energy to maintain a comfortable temperature.
  
## Risks

The key risk factors include:

- Availability and cost of the hardware components.
- The reliability, range, and scalability of the wireless (BT or WiFi) connections.
- Considering whether the lack of granular controls in HVAC systems may hinder the effectiveness of any algorithm designed to address the issues.
- Recognizing that the prototype solution requires technical expertise from a knowledgeable individual.
- ESPHome and Home Assistant platforms stop being maintained.

## Conclusion

The proposed solution meets all the product requirements:

- It’s a scalable room-temperature monitoring solution with an integrated zone controller.
- It utilizes a Smart Home platform that offers a user-friendly interface for configuring and viewing room temperatures.
- The platform also logs data to monitor heating and cooling patterns, enabling system optimization.
- It allows easy modification of the thermostat algorithm.
- It facilitates seamless OTA updates to the ESP controller.
- It facilitates more advanced home automations with other smart home devices present in a home such as motion or presence sensors, door/window sensors, and so on.
- It trades off a little cost ($207 vs $200 target) and consumer-friendliness for a fast and engineering efficient prototype solution.
- It is extensible for future enhancements outlined in the roadmap and more.

## References

Various references that may be useful to development.

#### Home Assistant

- https://www.home-assistant.io

#### ESPHome

- https://esphome.io/guides/getting_started_hassio
- Flash initial base ESPHome firmware using esptool or similar, then can develop code using OTA through Home Assistant

#### ESP32

- Relay Board Guide: https://werner.rothschopf.net/microcontroller/202208_esp32_relay_x8_en.htm
- Test web server: https://werner.rothschopf.net/microcontroller/202108_esp_generic_webserver_en.htm
- Use VSCode with PlatformIO to build, upload and test the generic webserver sketch if desired.
- Flashing using USB-TTL programmer:
  - Pins used: 5V, TX, RX, GND (don't need RTS/CTS).
  - Connect cable TX to RX header pin and cable RX to TX header pin i.e. connecting transmitter (TX) on one side to receiver (RX) on the other side.

#### Xiaomi Temperature Sensor

- Flash temperature sensor with custom firmware
- Guides:
  - https://community.home-assistant.io/t/xiaomi-temperature-humidity-sensor-home-assistant-integration-pvvx-custom-firmware-may-2023/572569 
  - https://hackaday.com/2025/01/16/fighting-to-keep-bluetooth-thermometers-hackable/
- Firmware: https://github.com/pvvx/ATC_MiThermometer
- Flasher: https://pvvx.github.io/ATC_MiThermometer/TelinkMiFlasher.html
- Too many or poor BT connections can result in dropped packets for ESP32. Using a BT proxy can help:
  - https://esphome.io/components/bluetooth_proxy.html
  - https://community.home-assistant.io/t/increase-polling-with-xiaomi-temperature-sensors/357522/8

#### HVAC

- Ecobee Settings: https://support.ecobee.com/s/articles/Threshold-settings-for-ecobee-thermostats
- DAT sensor for HVAC furnace:
  - https://documents.alpinehomeair.com/product/C7835A%201009%20installation%20instructions.pdf
  - https://esphome.io/components/sensor/resistance.html
- HVAC Gas Furnace Control Board Diagram with explanation:
  - https://www.youtube.com/watch?v=2bbZbrHIX2w
  - https://www.youtube.com/watch?v=lYxMIzV1v1w

<!-- Document/Section Links -->
[contextSection]: #context
[futureGoals]: #future-goals
[license]: <LICENSE>
[prdFile]: <smart_thermostat_controller_prd.md>
[prodTechRequirements]: #producttechnical-requirements

<!-- ESPHome Links -->
[espHomePlatform]: https://esphome.io

<!-- ESP Hardware Links -->
[espPSU]: https://www.amazon.com/ALITOVE-100-240V-Converter-Security-Surveillance/dp/B07VQHYGRD/ref=sr_1_9?th=1
[espRelayBoard]: https://www.amazon.com/Development-Programmable-Wireless-Channel-ESP32-WROOM-32E/dp/B0DK6QKNBM/ref=pd_sbs_d_sccl_2_4/135-5556824-9943814?th=1
[espRelayBoardx4]: https://www.amazon.com/Development-Programmable-Wireless-Channel-ESP32-WROOM-32E/dp/B0DCZ549VQ/ref=pd_sbs_d_sccl_2_4/135-5556824-9943814?th=1
[hvac24VacTransformer]: https://www.amazon.com/Transformer-Multi-Tap-Isolation-Secondary-Replacement/dp/B0FQSFM934/ref=sr_1_4

<!-- Home Assistant Links -->
[homeAssistantKiosk]: https://www.youtube.com/watch?v=FbCn5zaOh6E
[homeAssistantPlatform]: https://www.home-assistant.io

<!-- HVAC System Links -->
[hvacCurrentSystemDesign]: <images/HVAC-Architecture-Current.png>
[hvacHouseHeatFlow]: <images/HVAC-Home-Heat-Flow.png>
[hvacNewDesignSolution]: <images/HVAC-Architecture-Proposed.png>
[hvacWiring]: https://cielowigle.com/blog/thermostat-wiring/

<!-- RPi Hardware Links -->
[rpiCase]: https://www.amazon.com/Geekworm-P579-V2-Raspberry-Support-Active/dp/B0CRD7XQCK
[rpiCompute]: https://www.amazon.com/Raspberry-Pi-8GB-SC1112-Quad-core/dp/B0CK2FCG1K/ref=sr_1_1
[rpiHeatsink]: https://www.amazon.com/Geekworm-H505-Raspberry-Aluminum-Heatsink/dp/B0D41NH1S8
[rpiNvmeHat]: https://www.amazon.com/Geekworm-X1001-Key-M-Peripheral-Raspberry/dp/B0CPPGGDQT
[rpiPSU]: https://www.amazon.com/Geekworm-USB-C-Power-Supply-Raspberry/dp/B0FGN6Q8BC
[rpiSSD]: https://www.amazon.com/Silicon-Power-128GB-P34A60-SP128GBP34A60M28/dp/B09HMWH1DG/ref=pd_ci_mcx_di_int_sccai_cn_d_sccl_1_10/135-5556824-9943814?th=1
[xiaomiCustomFirmware]: https://github.com/pvvx/ATC_MiThermometer
[xiaomiTempSensors]: https://www.aliexpress.com/w/wholesale-lywsd03mmc.html
