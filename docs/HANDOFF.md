# HANDOFF — Deye SG06 RS485 Monitor

**Audience:** future Cursor chats / humans opening this repo cold.  
**Status:** Field-validated lab toolkit (Monitor), split from umbrella `Inverter-BMS-RS485-Emulator`.

---

## 1. What this project is

Tools to **read and publish** Deye SG05/SG06 inverter telemetry over **Modbus RTU**:

1. Parameters & PDF register map (`docs/protocol/`)
2. Cytron **IRIV IOC MQTT Gateway** import JSON + Mosquitto Docker + HA package (`iriv-ioc-mqtt-gateway/`)
3. Python **slave emulator** for offline logger bring-up (`emulator/`)
4. ESPHome / ESP32 master example (`esphome/`)
5. Static **MQTT web dashboard** (`web/`) — browser uses MQTT over WebSockets; needs a publisher on `iriv/ivt/#`

**Not in scope here:** JK BMS Modbus on UART1 (see sister repo `jk-pb-rs485-monitor`). LCD firmware lives in `deye-mqtt-dashboard-lcd-35` (separate repo).

---

## 2. Lab topology (setup A — ESS)

```text
JK-PB1A16S10P ──CAN──► Deye SG06 ──RS485@9600──► IRIV (master) ──MQTT──► broker / HA / web
                         slave 1
```

- Pack: **16S 51.2 V 100 Ah**, BMS **JK-PB1A16S10P**, CAN protocol e.g. app `001` Deye LV hybrid.
- Prefer SOC/V/I from **inverter** Modbus (regs 183/184/190/191/…).
- Alternate master: ESPHome / ESP32 on RS485 (not at the same time as IRIV).

---

## 3. Repo layout

| Path | Role |
|------|------|
| `iriv-ioc-mqtt-gateway/` | `_gen_iriv_jobs.py`, import JSON, Docker Mosquitto, `iriv_deye_mqtt.yaml` |
| `emulator/` | `deye-sg06-ivt-emu.py`, `rs485_emu/` (Deye profile only) |
| `esphome/` | NodeMCU YAML + ESP32/RS485 wiring README |
| `web/` | Browser dashboard (MQTT WS) |
| `docs/protocol/` | Deye Modbus PDF V118 |

---

## 4. IRIV IOC — lessons learned

| Finding | Detail |
|---------|--------|
| Job cap | Firmware **before V1.2.6**: job **#27** → reboot + **empty** job list. **V1.2.6** fixes that. Import `iriv-ioc-config.json` (27 jobs, full PV2) on V1.2.6+. Use `iriv-ioc-config-26.json` only on older firmware (drops PV2 Current, keeps Inverter Frequency). |
| dataType | Use **s16** (`dataType: 3`) for 16-bit Deye holdings. `dataType: 1` → values ≈ scale only |
| Topics | One scale per job → hierarchy `iriv/ivt/battery/soc`, `pv1/power`, `load/current`, … |
| Host | MQTT host often `iriv-pi-control` |
| Rate | Raise `globalRateMax` / `globalBurst` when many 1 s jobs |
| Regenerate | `python iriv-ioc-mqtt-gateway/_gen_iriv_jobs.py` — do not hand-edit dozens of jobs if avoidable |
| First restore | USB-C → `http://10.0.0.1` → import JSON (see `iriv-ioc-mqtt-gateway/README.md`) |

Current job set on **V1.2.6+** (**27** enabled, `iriv-ioc-config.json`): PV1 + **PV2** V/I/P, **Load Current (179)**, and **Inverter Frequency (193)**. Older firmware: import the 26-job file (no PV2 Current).

Periods: **V/I/P = 1 s**; status/temp/SOC/grid+inv Hz = **10 s**; energy today = **30 s**.

---

## 5. Register cheat sheet (SG06 field)

| Reg | Meaning | Scale |
|-----|---------|-------|
| 59 | Operating status | 1 |
| 70 / 71 | Charge / Discharge today | 0.1 kWh (**70=charge**) |
| 84 | Load energy today | 0.1 kWh |
| 91 | Inv temp | 0.1, offset -100 |
| 108 | PV energy today | 0.1 kWh |
| 109 / 110 / 186 | PV1 V / I / P | 0.1 / **0.1** / 1 |
| 111 / 112 / 187 | PV2 V / I / P | 0.1 / **0.1** / 1 |
| 150 / 160 / 172 | Grid V / I / P_CT | 0.1 / **0.01** / 1 |
| 175 / 193 | Inv power / frequency | 1 / 0.01 |
| 178 / **179** | Load power / **current** | 1 / **0.01** |
| 182–184, 190–191 | Batt T/V/SOC/P/I | temp offset -100; V 0.01; I 0.01 |

PDF: `docs/protocol/Deye SG05 Modbus Protocol.V118.pdf` (trust field table above when PDF disagrees).

---

## 6. Emulator & ESPHome

```bash
pip install -r requirements.txt
python emulator/deye-sg06-ivt-emu.py --port COMxx --debug --scenario day
```

- Emulator is a **slave** (answers FC03). Smoke: reg 59 = 2 (normal) / 4 (fault).
- ESPHome: `esphome/deye-sg06-nodemcu.yaml` + `esphome/README.md`. Sparse map → many FC03 ranges; keep ~15 s + `command_throttle` or you get `Frame already active` / `Poll refused`.

---

## 7. Web dashboard

`web/` subscribes to `iriv/ivt/#` over WebSockets. Compatible publishers: **IRIV IOC MQTT Gateway** or an **ESP32 that publishes the same MQTT topics**. Not for ESPHome-only HA API without MQTT.

---

## 8. Sister repo

| Item | JK-PB monitor |
|------|----------------|
| Path | `../jk-pb-rs485-monitor` |
| Baud | 115200 |
| Slave | 15 |
| Port | JK I/O leftmost **RS485** (UART1), protocol app `001` |
| MQTT | `iriv/jkbms/...` |
| Bench BMS slave | `emulator/bms-pylon-emu.py` (Pylon-style — **not** JK) |
