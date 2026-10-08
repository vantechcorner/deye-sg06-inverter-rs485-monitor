# Agent notes — Deye SG06 Inverter RS485 Monitor

Read **[docs/HANDOFF.md](docs/HANDOFF.md)** before changing Modbus maps, IRIV JSON, or ESP32 / ESPHome pollers.

## Project role

Monitor a **Deye SG05/SG06** hybrid inverter via **Modbus RTU slave** on the datalogger RS485 port. This repo does **not** talk to the JK BMS over RS485 (that is the sister repo `jk-pb-rs485-monitor`).

## Layout

- `iriv-ioc-mqtt-gateway/` — IRIV job generator, import JSON, Mosquitto Docker, HA MQTT YAML  
- `esphome/` — NodeMCU/ESP32 master YAML + wiring/flash README  
- `emulator/` — Deye Modbus slave + `rs485_emu/`  
- `web/` — MQTT WebSocket dashboard (`iriv/ivt/#`)  
- `docs/` — HANDOFF, protocol PDF, images  

## Hard constraints

- **One Modbus master per RS485 bus** (IRIV **or** ESPHome / ESP32 — never two).
- Baud **9600** 8N1, slave **1**.
- Cytron IRIV IOC firmware **before V1.2.6**: a 27th enabled poll job caused **reboot + wipe**. **V1.2.6** fixes that. Import `iriv-ioc-mqtt-gateway/iriv-ioc-config.json` (27 jobs) on V1.2.6+; use `iriv-ioc-config-26.json` only on older firmware.
- Prefer battery SOC/V/I from **inverter** registers when the pack is on **CAN** to Deye.
- Regenerate IRIV jobs only via `python iriv-ioc-mqtt-gateway/_gen_iriv_jobs.py`.

## Sister repo

`../jk-pb-rs485-monitor` — JK-PB UART1 Modbus @ 115200, slave 15. Pylon-style BMS emulator: `emulator/bms-pylon-emu.py`.
