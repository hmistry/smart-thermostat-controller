# Smart Thermostat Controller: Final Design and Results

- Author: Hiren Mistry
- Created: 12/01/2025
- Updated: 12/26/2025

*Disclaimer: I'm no expert. The information in this repository is based on my research and what I built to solve the problem I had. It does not express or imply you should do the same nor guarantees it will solve your HVAC issues.  It is provided "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED as stated in the [License][license].*

## Overview

This document describes the final design, implementation, and performance results of the lower-cost, scalable room-sensing Smart Thermostat Controller described in the [Technical Specification][techSpec]. The implementation matches the proposed architecture.

The system has been operating in my home for 9 months. It has improved comfort significantly: room temperatures remain within a tight band, cold spots and overheated areas have nearly disappeared, and energy usage increased only a little (acceptable tradeoff). While not perfect, the system performs much better than the original thermostat and zone controller.

## Final Design

### Architecture

![HVAC Design Solution][hvacDesignSolution]

The final solution (shown in the figure above) consists of:

- A thermostat platform built on Home Assistant (HA) running on a Raspberry Pi 5 with remote sensors.
- Xiaomi Bluetooth temperature sensors placed throughout the home.
- An ESP32-based HVAC Zone Controller that drives dampers, furnace, A/C condenser, and blower fan.
- Home Assistant logic that aggregates temperatures, determines zone heating/cooling demands, controls HVAC Zone Controller, and provides dashboard interface and analytics.

### Implementation Details

In this section, we will go over setting up the Thermostat hardware and Home Assistant platform, then the HVAC Zone Controller, and finally all the Home Assistant Thermostat logic components to make it work. We will also build the dashboard components and analytics.

#### 1. Thermostat Platform: Raspberry Pi 5 + Home Assistant

![Raspberry Pi Hardware][raspberryPiHardware]

Assembled the Raspberry Pi 5 hardware. Home Assistant OS was installed following the [official guide][homeAssistantInstallation]. This part was straightforward.

Key recommendations:

- Use an SSD, not an SD card (common issue). Several SD cards repeatedly got corrupted after 2–4 weeks resulting in downtime and data loss.
- Use MariaDB instead of the default SQLite database (also common). SQLite DB frequently got corrupted when daily backups became large (100-150MB).

Raspberry Pi 5 Hardware list:

- [Raspberry Pi 5 8GB][rpiCompute]
- [Raspberry Pi Heatsink][rpiHeatsink]
- [27W USB-C PD Power Supply][rpiPSU]
- [M.2 NVMe HAT][rpiNvmeHat]
- [Raspberry Pi 5 Case][rpiCase]
- [128GB M.2 NVMe SSD][rpiSSD]

#### 2. Temperature and Humidity Sensors

![Xiaomi Sensors][xiaomiSensorsB]

The system uses the [Xiaomi LYWSD03MMC Bluetooth Temperature/Humidity sensors (LYWSD03MMC)][xiaomiTempSensors]. These sensors are such a good value; 9 sensors costed $36, cheaper than a single Ecobee/Amazon/Nest remote temperature sensor!

Steps performed:

1. Updated all sensors to [ATC_MiThermometer custom firmware][xiaomiCustomFirmware] using the [video guide][xiaomiFirmwareUpdate].
2. Adjusted the following configuration parameters in the sensors firmware (optional as unsure if reducing update interval will prolong battery life):
   1. Increased BT advertising interval to 5000 ms
   2. Changed measure interval to 12 x 5000 ms = 60 sec
   3. Changed device name
   4. See [example configuration][xiaomiConfiguration] (mostly default)
3. Calibrated the sensors by placing them together in a stable environment; all were within 0.3°F - good enough for me. You can add calibration offset to the readings if desired.
4. Installed 9 sensors across upstairs, downstairs, and one outdoor location.
5. Added all sensors to Home Assistant and created components for:
   - Temperature
   - Humidity
   - Battery
6. Composite temperatures for each zone are created later in the Home Assistant [Thermostat logic](#41-composite-temperature-sensors).

![HA Temperature Component][haXaomiTemperature]

#### 3. HVAC Zone Controller (ESP32 Relay Board)

![ESP32 x8 Relay Board][esp32Board]

The HVAC Zone controller uses an ESP32 x8 Relay Board programmed via [ESPHome][espHomePlatform]. For a single-speed HVAC with A/C Condenser, a single zone uses 3 relays, 2 zones uses 5 relays, and 5 zones will use all 8 relays. Other HVAC systems will differ.

Steps performed:

1. Flash the base ESPHome firmware to the ESP32 (see the [official guide][espHomeInstallation]) using a [USB-TTL programmer][usbProgrammer] connected to the ESP32 board (5V, RX, TX, CS header pins - see [ESP32 Relay Board guide][espBoardGuide]). Only done the first time; subsequent updates can be done via OTA through Home Assistant.
2. Install the ESPHome add-on in Home Assistant following the [ESPHome guide][espHomeInstallationGuide].
3. Load the final [HVAC Zone Controller firmware][esp32HvacControllerCode] to the ESP32 board through Home Assistant.
4. Verify all the relay timing and sequencing meets our safeguards requirements.
5. Wire the relays to:
   - Furnace heat (W) to Relay 1 NO
   - A/C condenser (Y) to Relay 2 NO
   - Blower fan (G) to Relay 3 NO
   - Furnace 24VAC power (Rh) to Relay 1 COM
   - A/C condenser 24VAC power (Rc) to Relay 2 COM and Relay 3 COM (if no Rc power, use Rh power)
   - Zone dampers (normally open) to Relay 7 NO and Relay 8 NO
   - 24VAC Transformer (+) to Relay 7 COM and Relay 8 COM
   - Zone dampers (COM) to 24VAC Transformer COM (-)

![ESP32 USB-TTL Programmer][esp32UsbTtlCable]

Hardware:

- [ESP32 x8 Relay Board][espRelayBoard]
- [ESP32 12V DC Power Supply][espPSU]
- [24VAC 40VA Transformer][hvac24VacTransformer] (to power zone dampers)
- [USB-TTL programmer][usbProgrammer] (used to upload initial ESPHome firmware to ESP32 Relay board)
  
#### 4. Thermostat + HVAC Controller Configuration in Home Assistant

After adding the Xiaomi sensors and ESP32 HVAC Zone Controller to Home Assistant, the following controller logic was implemented in Home Assistant.

##### 4.1 Composite Temperature Sensors

![HA Composite Temperature][haCompositeTemperature]

Purpose: represent the temperature of a zone using multiple room sensors.

- Implemented with Template Sensors (one per zone)
- Type: measurement
- Precision: tenths
- Unit: °F
- Computed using a window algorithm: calculate the min/max of selected sensors, if min < target, set to min temperature, else set to max temperature.
- See [Composite calculation code][haCodeCompositeTemp]
- Targets are hardcoded due to a precision bug in Generic Thermostat ([GitHub issue][githubBugIssue])

##### 4.2 Heat/Cool Demand Switches

![HA Heat/Cool Switches][haHeatCoolSwitches]

Purpose: hold thermostat demand state for heating and cooling of a zone.

- Implemented using Template Switches (two per zone)
- When the switch changes state, an automation sends commands to the ESP32 HVAC Zone Controller

##### 4.3 HVAC Zone Controller

![HA HVAC Zone Controller][haZoneController]

Purpose: view state and control ESP32 HVAC Zone Controller.

- Implemented via HVAC Zone Controller code using ESPHome builder

##### 4.4 Generic Thermostat Entities

![HA Thermostats][haThermostats]

Purpose: UI to view temps, set targets, and trigger demand switches. Two thermostats needed per zone - one for heating and other for cooling.

- Implemented using Generic Thermostat component in Home Assistant
- Settings:
  - cold/hot tolerance: 1.5°F
  - minimum cycle duration: none (this is handled in the HVAC Zone Controller)
  - minimum temperature: 60°F
  - maximum temperature: 100°F
  - temperature presets: comfort = 71.5°F, sleep = 70.5°F, others = none
  - precision: tenths
  - modes: Off, Comfort, Sleep
  - actuator switch: set to corresponding Heat/Cool Demand Switch
- Note: Display [rounding bug][githubBugIssue] exists, but internal values are correct.
- See [additional thermostat configuration code][haCodeGenericThermostat]

##### 4.5 Heat/Cool Automations

![HA Automations Heat-Cool][haAutomationsHeatCool]

Purpose: send commands to HVAC Zone Controller to heat/cool zones.

Heat/Cool Automations handles:

- Turning heater or A/C on/off based on heat/cool demand switches
- Opening/closing zone dampers
- Enforcing required delays
- See code for:
  - [Cool Off automation][haCodeAutomationCoolOff]
  - [Cool On automation][haCodeAutomationCoolOn]
  - [Heat Off automation][haCodeAutomationHeatOff]
  - [Heat On automation][haCodeAutomationHeatOn]

Optimizations added based on performance in my home:

- Ensuring no short cycling (<5 min run time)
- Only heating/cooling one zone at a time
- Limiting a single zone run to 15 minutes (added to prevent extended runs of one zone starving the other zone)

##### 4.6 Scheduler (Optional)

![HA Scheduler][haScheduler]

Purpose: set thermostat preset modes based on daily schedules. One scheduler per zone.

- Implemented using Scheduler helper component + automations
- Controls comfort/sleep preset modes of the Heat/Cool Generic Thermostats per zone
- See code for:
  - [Comfort Preset automation][haCodeAutomationComfort]
  - [Sleep Preset automation][haCodeAutomationSleep]

##### 4.7 Zone Controller Reconnect Handling

![HA Automations Zone Controller Connection][haAutomationsZoneController]

Purpose: ensure correct HVAC state after an ESP32 HVAC Zone Controller reconnects to Home Assistant (e.g. after power loss and/or Wifi connection loss).

- Automation updates furnace, A/C, and damper states on reconnect
- See [Zone Controller Reconnection automation code][haCodeAutomationHVACControllerReconnect]

##### 4.8 HVAC Dashboard & Analytics

![HA Zone Controller Status][haZoneControllerStatus]
![HA HVAC Temperature/Humidity Charts][haTempHumidityCharts]
![HA HVAC Usage Charts][haUsageCharts]

Dashboard includes:

- All thermostat controls
- Composite and room temperatures
- HVAC Zone Controller controls
- Charts for performance and historical statistics:
  - Daily heat and cool runtime for last 30 days
    - used history stats helper (set start: `{{ today_at('00:00') }}`, duration: 24hrs)
    - used [Apex charts][homeAssistantApexCharts] from [HACS][homeAssistantHacsUse] for 30 day historical chart
    - see [configuration for Apex historical charts][haCodeApexCharts]
  - HVAC Zone Controller status for last 12 hrs showing:
    - Cycle state
    - Zone state
    - Wifi connection state
- See the [full HVAC dashboard][homeAssistantHvacDashboard]

## Results

High-level summary: a major success!

1. Priority rooms maintain tight comfortable temperature ranges near the preferred targets.
2. Optimized system by adjusting air vents to improve thermal balance between rooms.
3. Temperature data and runtime charts makes it easy to diagnose issues, test changes, and tradeoff between comfort and energy consumption.

**Ratings:**

1. Comfort (9/10)
   - The system maintains a very stable temperature in priority rooms. Some edge cases of uneven temperatures still occur due to solar heat gain, open windows, or airflow imbalances but the window-based algorithm is working better to balance comfort in these cases.
2. Ease of Use (7/10)
   - Setup requires hardware, wiring, and Home Assistant experience. Once configured, the system requires very little attention. Dashboards, charts, and mobile access make monitoring easy.
3. Energy Consumption (8/10)
   - Energy use increased slightly (10–20%), but comfort improved significantly. We prioritized comfortable temperature over energy consumption which was not possible before due to thermostat coupling between zones and inaccurate sensing.

## Conclusion

The Smart Thermostat Controller met and exceeded its design goals (i.e. lower cost, scalable room-sensing, enhanced thermostat algorithm, smart home integration, and data collection) resulting in a more comfortable home. In practice, the flexibility of room-level sensing combined with a custom control strategy effectively addresses common problems found in forced-air HVAC systems with uneven heating and cooling.

To recap, the key advantages of the final design include:

1. Accurate temperature sensing
   - Temperature sensors can be placed in ideal locations within each room, avoiding sunlight, doors, windows, and vents.
2. Elimination of inter-zone thermal coupling
   - Control decisions are based on room-level measurements, not central hallway thermostats that are influenced by heat transfer between floors or zones.
3. Data-driven HVAC tuning
   - Temperature and runtime data enables systematic tuning of airflow vent settings, zone dampers, and control logic to improve thermal distribution.
4. Support for informed user intervention
   - The system makes external thermal influences visible (e.g. solar heat gain, cooling), allowing users to adjust behavior (e.g. open/close doors, windows, curtains) when necessary. In my experience, rooms do not equalize quickly enough to rely solely on HVAC control despite what HVAC technicians say.

Overall, this architecture is a good foundation for more advanced residential HVAC control. Future possibilities that can further improve comfort, efficiency, and automation:

1. Use the distributed temperature sensing and Home Assistant platform without ESP32 Zone Controller to monitor thermal distribution issues and tune/monitor existing HVAC systems (no changes required to currently installed thermostats or controllers).
2. Incorporate occupancy awareness using motion or presence sensors to prioritize room selection dynamically.
3. Add humidity sensing and control to improve comfort during humid seasons.
4. Use CO₂ sensors to trigger periodic ventilation and maintain indoor air quality.
5. Integrate weather forecasts to anticipate heating and cooling demand and adjust setpoints proactively.

<!-- Document Links -->
[homeAssistantHvacDashboard]: <reference/HA-HVAC-Dashboard.pdf>
[license]: <LICENSE>
[techSpec]: <smart_thermostat_controller_techspec.md>
[xiaomiConfiguration]: <reference/Xiaomi-Temp-Sensor-Config-Params.pdf>

<!-- HA Code Links -->
[haCodeApexCharts]: <code/apex-charts.yaml>
[haCodeAutomationComfort]: <code/automation-comfort-upstairs.yaml>
[haCodeAutomationCoolOff]: <code/automation-cool-off.yaml>
[haCodeAutomationCoolOn]: <code/automation-cool-on.yaml>
[haCodeAutomationHeatOff]: <code/automation-heat-off.yaml>
[haCodeAutomationHeatOn]: <code/automation-heat-on.yaml>
[haCodeAutomationHVACControllerReconnect]: <code/automation-hvac-controller-reconnect.yaml>
[haCodeAutomationSleep]: <code/automation-sleep-upstairs.yaml>
[haCodeCompositeTemp]: <code/composite-temperature.md>
[haCodeGenericThermostat]: <code/configuration.yaml>

<!-- Images - HVAC -->
[hvacDesignSolution]: <images/HVAC-Architecture-Proposed.png>

<!-- Images - Hardware -->
[esp32Board]: <images/ESP-HVAC-Zone-Controller.jpg>
[esp32UsbTtlCable]: <images/ESP-Relay-Board-Programmer.jpg>
[raspberryPiHardware]: <images/RPi-Hardware.jpg>
[xiaomiSensorsB]: <images/Xiaomi-Temp-Sensor.jpg>

<!-- Images - HA -->
[haAutomationsHeatCool]: <images/HA-Automations-Heat-Cool.png>
[haAutomationsZoneController]: <images/HA-Automations-Reconnect.png>
[haCompositeTemperature]: <images/HA-Composite-Temperature.png>
[haHeatCoolSwitches]: <images/HA-Heat-Cool-Switches.png>
[haScheduler]: <images/HA-Scheduler.png>
[haTempHumidityCharts]: <images/HA-Charts.png>
[haThermostats]: <images/HA-Thermostats.png>
[haUsageCharts]: <images/HA-Usage-Charts.png>
[haXaomiTemperature]: <images/HA-Temperature-Battery.png>
[haZoneController]: <images/HA-Zone-Controller.png>
[haZoneControllerStatus]: <images/HA-Zone-Controller-Status.png>

<!-- Web Links - Home Assistant -->
[githubBugIssue]: https://github.com/home-assistant/core/issues/140819#issuecomment-3622282557
[homeAssistantApexCharts]:https://github.com/RomRider/apexcharts-card
[homeAssistantHacsUse]: https://www.hacs.xyz/docs/use/
[homeAssistantInstallation]: https://www.home-assistant.io/installation/

<!-- Web Links - RPi Hardware -->
[rpiCase]: https://www.amazon.com/Geekworm-P579-V2-Raspberry-Support-Active/dp/B0CRD7XQCK
[rpiCompute]: https://www.amazon.com/Raspberry-Pi-8GB-SC1112-Quad-core/dp/B0CK2FCG1K/ref=sr_1_1
[rpiHeatsink]: https://www.amazon.com/Geekworm-H505-Raspberry-Aluminum-Heatsink/dp/B0D41NH1S8
[rpiNvmeHat]: https://www.amazon.com/Geekworm-X1001-Key-M-Peripheral-Raspberry/dp/B0CPPGGDQT
[rpiPSU]: https://www.amazon.com/Geekworm-USB-C-Power-Supply-Raspberry/dp/B0FGN6Q8BC
[rpiSSD]: https://www.amazon.com/Silicon-Power-128GB-P34A60-SP128GBP34A60M28/dp/B09HMWH1DG/ref=pd_ci_mcx_di_int_sccai_cn_d_sccl_1_10/135-5556824-9943814?th=1

<!-- Web Links - ESP Hardware -->
[espPSU]: https://www.amazon.com/ALITOVE-100-240V-Converter-Security-Surveillance/dp/B07VQHYGRD/ref=sr_1_9?th=1
[espRelayBoard]: https://www.amazon.com/Development-Programmable-Wireless-Channel-ESP32-WROOM-32E/dp/B0DK6QKNBM/ref=pd_sbs_d_sccl_2_4/135-5556824-9943814?th=1
[hvac24VacTransformer]: https://www.amazon.com/Transformer-Multi-Tap-Isolation-Secondary-Replacement/dp/B0FQSFM934/ref=sr_1_4
[usbProgrammer]: https://www.amazon.com/dp/B0CX55K4RG

<!-- Web Links - Xiaomi -->
[xiaomiCustomFirmware]: https://github.com/pvvx/ATC_MiThermometer
[xiaomiFirmwareUpdate]: https://community.home-assistant.io/t/xiaomi-temperature-humidity-sensor-home-assistant-integration-pvvx-custom-firmware-may-2023/572569
[xiaomiTempSensors]: https://www.aliexpress.com/w/wholesale-lywsd03mmc.html

<!-- Web Links - ESPHome -->
[esp32HvacControllerCode]: <zone-hvac-controller.yaml>
[espBoardGuide]: https://werner.rothschopf.net/microcontroller/202208_esp32_relay_x8_en.htm
[espHomeInstallation]: https://esphome.io/guides/physical_device_connection/
[espHomeInstallationGuide]: https://esphome.io/guides/getting_started_hassio/
[espHomePlatform]: https://esphome.io
