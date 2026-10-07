# Deye SG06 RS485 Monitor

Monitor a **Deye SG05/SG06** hybrid inverter over **Modbus RTU (RS485)** on the datalogger port. This repo provides register maps, a Cytron IRIV IOC MQTT Gateway import, an ESPHome / ESP32 RS485 master example, a PC slave emulator, Mosquitto Docker for MQTT, a Home Assistant package, and a browser MQTT dashboard.

> Vietnamese: [README-vn.md](README-vn.md)  
> Agent notes: [AGENTS.md](AGENTS.md) · [docs/HANDOFF.md](docs/HANDOFF.md)

**Sister project:** [jk-pb-rs485-monitor](../jk-pb-rs485-monitor) — JK-PB* BMS Modbus (separate bus / baud).

---

## What this project does

1. Talks to the inverter as a Modbus **slave** (you are the **master**).
2. Uses holding-register addresses from Deye’s protocol PDF, with **field-verified** scales for SG06.
3. Publishes or exposes live metrics: battery, PV1/PV2, grid, load, inverter power / frequency / temperature, and daily energy counters.

Protocol PDF: [`docs/protocol/Deye SG05 Modbus Protocol.V118.pdf`](docs/protocol/Deye%20SG05%20Modbus%20Protocol.V118.pdf)

### Bus parameters

| Parameter | Value |
|-----------|--------|
| Baud | **9600** 8N1 |
| Slave ID | **1** |
| Function codes | 03 (read), 10 (write) |
| Port | Datalogger / meter **RS485** — not the BMS **CAN** RJ45 |

**One Modbus master per RS485 bus.** Run either IRIV **or** ESPHome/ESP32 on that A/B pair — never both.

### Field-verified register corrections (SG06)

| Topic | Reg | Scale / note |
|-------|-----|----------------|
| Charge / Discharge today | **70** / **71** | 0.1 kWh (**70 = charge**) |
| PV current | 110, 112 | **0.1** A |
| Grid current | 160 | **0.01** A |
| Load current | **179** | 0.01 A |
| Load power | 178 | 1 W |
| Inverter frequency | 193 | 0.01 Hz |
| Batt / Inv temp | 182, 91 | 0.1, offset **-100** |

IRIV IOC firmware **before V1.2.6**: enabling a **27th** poll job rebooted the gateway and wiped jobs. **V1.2.6** fixes that.

---

## Hardware paths

### A — Cytron IRIV IOC MQTT Gateway (RS485 → MQTT)

```text
Deye SG06 (slave 1) --RS485@9600--> IRIV IOC (master) --MQTT--> Mosquitto
                                                              |--> Home Assistant
                                                              |--> web/ dashboard (WS :9001)
```

Full guide: [`iriv-ioc-mqtt-gateway/README.md`](iriv-ioc-mqtt-gateway/README.md)  
(Mosquitto Docker, first USB restore at `http://10.0.0.1`, HA sensors.)

### B — ESP32 / ESPHome + RS485 transceiver (RS485 → ESPHome / HA)

```text
Deye SG06 (slave 1) --RS485@9600--> ESP32/NodeMCU + MAX3485 (master) --Wi‑Fi--> Home Assistant (ESPHome API)
```

Wiring and flash: [`esphome/README.md`](esphome/README.md)

### Communication methods (summary)

| Method | Master | Transport to HA / UI |
|--------|--------|----------------------|
| **RS485 → MQTT** | IRIV IOC MQTT Gateway | Mosquitto topics `iriv/ivt/#` → HA MQTT package + [`web/`](web/) |
| **RS485 → ESPHome** | ESP32 / NodeMCU | ESPHome native API (entities in HA). Web dashboard needs MQTT publishers only. |

---

## Repository layout

| Path | Contents |
|------|----------|
| [`iriv-ioc-mqtt-gateway/`](iriv-ioc-mqtt-gateway/) | IRIV JSON configs, job generator, Mosquitto Docker, HA MQTT YAML |
| [`esphome/`](esphome/) | ESPHome YAML + ESP32/RS485 wiring & flash guide |
| [`emulator/`](emulator/) | Deye Modbus **slave** emulator for bench bring-up |
| [`web/`](web/) | MQTT WebSocket dashboard (`iriv/ivt/#` only) |
| [`docs/`](docs/) | HANDOFF, protocol PDF, images |

```bash
pip install -r requirements.txt
```

---

## Quick start

### IRIV + MQTT

```bash
cd iriv-ioc-mqtt-gateway
docker compose up -d
python _gen_iriv_jobs.py   # optional: regenerate JSON
```

USB-C to the gateway → browser **`http://10.0.0.1`** → import `iriv-ioc-config.json` (firmware ≥ V1.2.6). Details: [iriv-ioc-mqtt-gateway/README.md](iriv-ioc-mqtt-gateway/README.md).

### ESPHome

See [esphome/README.md](esphome/README.md). Example:

```bash
cd esphome
esphome run deye-sg06-nodemcu.yaml
```

### Bench emulator (slave)

```bash
python emulator/deye-sg06-ivt-emu.py --port COM35 --debug --scenario day
```

### Web dashboard

Only if something publishes **`iriv/ivt/#`** (IRIV gateway or an ESP32 MQTT poller — **not** ESPHome API alone):

```bash
cd web && python -m http.server 8080
```

Open `http://127.0.0.1:8080` → broker WebSocket `ws://<host>:9001`. See [web/README.md](web/README.md).

---

## Field-tested lab

| Hardware | Verified |
|----------|----------|
| **Deye 6 kW SG06** + **16S 51.2 V 100 Ah** (JK-PB1A16S10P on **CAN** to inverter) | Inverter Modbus @ **9600**, slave **1** → IRIV MQTT / ESPHome |

Prefer battery SOC/V/I from **inverter** Modbus registers when the pack already talks CAN to Deye.

![Setup — Deye SG06 + 16S pack](docs/images/setup-deye-sg06-16s-100ah.jpg)

*Placeholder: save as `docs/images/setup-deye-sg06-16s-100ah.jpg`.*

---

## License / lab note

Lab toolkit for personal ESS monitoring. Protocol PDF is Deye’s document — redistribute per their terms.
