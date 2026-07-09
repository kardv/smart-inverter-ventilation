# Smart Inverter Ventilation Controller

This repository contains the local, data-driven firmware configuration for a custom solar inverter room ventilation system powered by **ESPHome** and integrated natively with **Home Assistant**.

<img height="500" alt="2026-07-09_22-58-10" src="https://github.com/user-attachments/assets/f95214dc-3d53-4256-a901-0c9086a3eb88" />

## Key Features
* **Silent 25kHz PWM Output:** Runs the cooling fan output at an ultrasonic frequency to eliminate coil hum.
* **500ms Inertia Kickstart:** Forces a temporary 100% duty cycle on boot to ensure low-voltage DC fans don't stall.
* **Single-Button Smart Logic:** Multi-function momentary tactile controls (Short click to cycle speeds, long press to shut down).

## Hardware Pin Mapping
Copy and paste your custom pinout table here (the table we generated in the previous step).

## Getting Started
1. Install ESPHome on your machine or inside Home Assistant.
2. Copy the `inverter-ventilation.yaml` file from this repository.
3. Update the `api.encryption.key`, `ota.password`, and your local Wi-Fi credentials.
4. Compile and flash your hardware module!


# ESP8266 Pin Mapping & Hardware Connections

<img height="500" alt="20260414_040211 1" src="https://github.com/user-attachments/assets/3bd1d9f4-a6c1-4663-88d0-0fb54cf1c8c1" />

To use this ESPHome firmware, map your microcontroller (D1 Mini / ESP8266) pins as follows:

| D1 Mini Pin | ESP8266 GPIO | Component / Function | Active State / Mode |
| :--- | :--- | :--- | :--- |
| **D5** | `GPIO14` | Room Exhaust Fan Relay | Active HIGH |
| **D7** | `GPIO13` | 12V DC Cooling Fan PWM | 25kHz PWM Output |
| **D2** | `GPIO4` | Manual Exhaust Button | INPUT_PULLUP (Active LOW) |
| **D3** | `GPIO0` | Manual Cooling Fan Button | INPUT_PULLUP (Active LOW) |
| **D6** | `GPIO12` | Cooling Fan State LED | Active HIGH |
| **D0** | `GPIO16` | Exhaust Fan State LED | Active HIGH |
| **D1** | `GPIO5` | Dallas Temperature Sensor Bus (DS18B20) | One-Wire Protocol |
| **D4** | `GPIO2` | Onboard Status LED | Active HIGH (Built-in) |

### Hardware Protection Notes
* **Relay Output:** Ensure your relay driver circuit uses a flyback diode (e.g., 1N4148 or 1N4007) to protect the GPIO from inductive kickback.
* **PWM Output:** Driven via an N-Channel MOSFET (like the IRFz44n) with a 100Ω gate resistor and a 10kΩ pull-down resistor to prevent fan floating states on boot.
* **Mains Isolation:** Always maintain a physical separation gap of at least 4mm on your custom PCB between the high-voltage AC mains section (HLK-5M05 input/Relay) and the low-voltage DC section.

## 🌀 Supported Fan Configurations & Wiring

This controller board is highly versatile and supports dual-layer cooling topologies simultaneously:

### 1. High-Volume AC Exhaust (Relay Output)
* **Usage:** Best for hot-air extraction from the top of the inverter room.
* **Support:** Directly switches one heavy-duty 220V AC room exhaust fan.

### 2. 12V DC Chassis Cooling Fans (MOSFET PWM Output)
* **Two-Wire Operation:** This build **does not require the fan's RPM or built-in PWM data wires**. It natively controls speed by high-frequency low-side switching. Simply connect the fan's Positive (+) and Negative (-) power wires.
* **Parallel Connection:** To run multiple 12V DC fans, **always connect them in parallel** (all positive wires to the 12V terminal, all negative wires to the MOSFET switch terminal). Do not wire them in series.
* **Capacity:** The onboard TO-220 MOSFET (IRFz44n) easily supports a combined load of up to **4A to 5A** without requiring a dedicated heatsink. This means you can safely wire **up to 4 standard 1A 12V server fans** in parallel.

## 🔌 Connection Layout & Terminal Map

The PCB features clearly labeled physical interfaces utilizing **angled screw terminals** to allow smooth wire routing and reduce cable strain inside tight project enclosures:

### Power & Sensor Inputs
1. **AC INPUT (2-pin):** Connects to your main 220V AC grid line to power the onboard isolated HLK-5M05 step-down module.
2. **12V DC INPUT (2-pin):** Direct DC input to drive the high-airflow cooling fan circuit (kept isolated from logic power for safety).
3. **Wired Temperature Sensor (3-pin):** Dedicated pinout for an external, waterproof **DS18B20 Dallas Temperature Sensor** probe to track room ambient or direct inverter chassis temperature.

### Controlled Outputs
4. **AC OUTPUT (2-pin):** Switched via the onboard heavy-duty physical relay to control a high-volume 220V AC room exhaust fan.
5. **12V DC OUTPUT (2-pin):** Driven by a TO-220 N-Channel MOSFET (IRFz44n) with a high-frequency PWM signal to precisely modulate 12V DC chassis cooling fan speeds.

## 📦 Need a Board? (Spare PCBs Available)
I designed this layout for my personal setup and had to order a minimum batch of 5 boards. I have **4 bare spare PCBs** sitting on my workbench. 

If you are in Pakistan and want to build this exact ventilation setup without dealing with ordering custom PCBs yourself, I’m happy to ship out these spare boards to community members at cost. 

<img height="500" alt="20260710_003951 1" src="https://github.com/user-attachments/assets/b88bc380-d8f7-4b5f-a428-b11e814bc593" />

👉 **Join our community, drop a comment, or message me directly in our Facebook group to claim one:** [Smart Solar & Automation Pakistan](https://www.facebook.com/groups/smartsolarpk)

## 🤖 Example Home Assistant Automation (Smart Dynamic Cooling)

This controller is designed to be truly smart by reacting to your **inverter's internal temperature** (pulled via Modbus/Solarman/etc.) rather than just a simple room temperature sensor. 

While the exact sensor entity IDs will vary depending on your inverter integration, you can use the blueprint below as a starting point. 

### How the Logic Works:
1. **Trigger:** When the inverter's internal heatsink temperature goes above 50°C, the DC cooling fan turns on to speed stage `2` (40% speed).
2. **Trigger:** If it continues to climb past 58°C, it ramps the cooling fan to speed stage `4` (80% speed) and kicks on the heavy AC room exhaust fan.
3. **Trigger:** Once the inverter cools down below 45°C, it safely shuts everything down to conserve power and reduce noise.

```yaml
alias: "Solar: Inverter Smart Ventilation Control"
description: "Dynamically control exhaust and chassis fans based on internal inverter temperature"
mode: restart
trigger:
  - platform: numeric_state
    entity_id: sensor.inverter_internal_temperature  # <-- REPLACE WITH YOUR INVERTER SENSOR
    above: 50
    id: warm
  - platform: numeric_state
    entity_id: sensor.inverter_internal_temperature  # <-- REPLACE WITH YOUR INVERTER SENSOR
    above: 58
    id: hot
  - platform: numeric_state
    entity_id: sensor.inverter_internal_temperature  # <-- REPLACE WITH YOUR INVERTER SENSOR
    below: 45
    id: cool
condition: []
action:
  - choose:
      # Scenario 1: Inverter gets warm -> Turn on DC fan at Low-Medium speed
      - conditions:
          - condition: trigger
            id: warm
        sequence:
          - service: fan.turn_on
            target:
              entity_id: fan.cooling_fan
            data:
              percentage: 40  # Speed stage 2 of 5

      # Scenario 2: Inverter gets hot -> Ramp up DC fan and trigger AC exhaust
      - conditions:
          - condition: trigger
            id: hot
        sequence:
          - service: fan.turn_on
            target:
              entity_id: fan.cooling_fan
            data:
              percentage: 80  # Speed stage 4 of 5
          - service: switch.turn_on
            target:
              entity_id: switch.room_exhaust

      # Scenario 3: Inverter cools down -> Turn everything off
      - conditions:
          - condition: trigger
            id: cool
        sequence:
          - service: fan.turn_off
            target:
              entity_id: fan.cooling_fan
          - service: switch.turn_off
            target:
              entity_id: switch.room_exhaust
