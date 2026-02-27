# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Home Assistant smart home automation project with ESPHome-based IoT devices. All configuration is YAML-based — there is no application code, build system, or test suite. Comments in Bulgarian are common throughout.

## Repository Structure

- `homeassistant/` — Home Assistant configuration
  - `configuration.yaml` — minimal main config (uses `!include` for automations, scripts, scenes, templates)
  - `automations.yaml` — all HA automations (climate scheduling, lockouts, device coordination, night mode)
  - `esphome/` — ESPHome device configs (15 devices)
  - `scripts.yaml`, `templates.yaml`, `scenes.yaml` — currently empty placeholders
- `doc/` — SVG wiring and logic diagrams for physical hardware
- `prodinos-yml/` — PRODINo board YAML starter templates (outdated vs production configs — missing `min_auth_mode`, `hidden: true`, newer OTA format)

## Architecture

### ESPHome Devices — Two Hardware Generations

**ESP8266 (`esp_wroom_02` / `esp01_1m`):** Light controllers, window controllers, corridor-common
- SPI: GPIO14 (CLK), GPIO13 (MOSI), GPIO12 (MISO), CS on GPIO15
- MCP23S08 expander: inputs on pins 0-3 (inverted), relay outputs on pins 4-7

**ESP32 (`esp32dev`):** Convector thermostats (bedroom-1/2, living-room)
- SPI: GPIO18 (CLK), GPIO23 (MOSI), GPIO19 (MISO), CS on GPIO32
- Same MCP23S08 pin mapping as ESP8266 devices
- Living-room uses `esp-idf` framework; bedroom convectors use `arduino`

### Device Categories

**Light controllers** (corridor-lights, wet-room, bathroom, toilet-device):
- PWM light on GPIO5 (`monochromatic`, `restore_mode: ALWAYS_OFF`)
- DHT22 on GPIO4 (bathroom, wet-room) or PIR on MCP inputs
- `Night switch` template switch backed by `is_night_time` global — ON sets 50% brightness, OFF sets 100%
- `light_timer` script with `mode: restart` — turns on, waits, then fades off
- `small_lamp_relay` on MCP pin 4 (corridor-lights only): ON when night mode active

**Convector thermostats** (bedroom-1/2, corridor, sitting-room, living-room, north-terrace):
- Three fan-speed relays with interlock: Low (MCP 7), Medium (MCP 6), High (MCP 5)
- Bypass valve via direct GPIO pulse (GPIO13=on, GPIO14=off, 10s auto-off) — not MCP
- `FunSpeed` select entity ("0"/"1"/"2"/"3") drives relay selection
- `Thermostat` climate entity: HEAT_COOL mode, 15-30°C, 0.5°C step
- Core logic runs in `interval: 30s` lambda implementing proportional fan speed based on two deltas: dTt (room temp vs setpoint) and dTf (coil temp vs room temp)
- Anti-freeze protection: forces bypass on if temp < 4°C
- `climate_lockout_active` template switch — toggled by HA automations when windows open
- DHT22 on GPIO22 (room temp with `calibrate_linear` + moving average), DS18B20 on GPIO21 (coil temp)

**Window controllers** (bedroom-1/2, sitting-room, living-room):
- `time_based` cover: open relay (MCP 4), close relay (MCP 5) with interlock
- Open: 23s, Close: 25s, plus 5s end-stop confirmation pulse

**Corridor common:**
- `door_relay` (MCP 4) with 15s pulse via template button
- `ac_hot_relay` (MCP 5) / `ac_cold_relay` (MCP 6) with mutual interlock — controlled by HA automation
- `External Power` binary sensor (MCP 1) — drives light brightness sync across all light devices

### HA Automations

**Automation IDs** are HA-generated Unix timestamps (e.g., `id: '1737553922481'`). Device references use opaque `device_id` hashes, not friendly entity names.

**Climate schedule (daily cycle):**
- 06:00 — Night mode OFF (all lights to 100%)
- 07:30 weekdays — Bedroom heating starts
- 09:00 weekends + holidays — All rooms heat
- 17:00 — All rooms to evening temps
- 21:00 — Bedrooms to sleep temps, living/sitting rooms off
- 22:00 — Night mode ON (all lights to 50%)
- 23:00 weekdays — Living room + sitting room switch off

**Holiday mechanism (`praznik`):** The automation `praznik_9_chasa_sutrin` acts as a flag. Weekday automations check `state: ['off']` on this entity — when it's enabled (ON), weekday-only automations are skipped, making weekdays follow the weekend schedule.

**Climate lockout (per-room):** Triggers on window sensor, thermostat state, and cover position. Lockout activates when: heating and window position > 0%, or cooling and window position > 25%, or any window binary sensor ON. Sets `switch.<room>_convector_climate_lockout_active`.

**AC relay coordination:** Single automation watches all 5 convector `hvac_action` attributes and lockout switches. Computes heating/cooling booleans, controls hot/cold relays with mutual interlock and conflict logging.

**External power sync:** Propagates `binary_sensor.corridor_common_external_power` state to all light device switches.

## Conventions

- WiFi credentials in `secrets.yaml` (gitignored); all production devices use `min_auth_mode: WPA2` and `hidden: true`
- ESPHome entity naming: `<device_name>_<entity_name>` (e.g., `switch.corridor_lights_night_switch`)
- All automations use `mode: single`
- Automation aliases and descriptions are in Bulgarian
- Device configs include board-level comments referencing PRODINo hardware
