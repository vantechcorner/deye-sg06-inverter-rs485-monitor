# ESPHome / ESP32 + RS485 — Deye SG06 Modbus master

Run a Modbus RTU **master** on ESP8266/ESP32 with a UART→RS485 transceiver, poll the same Deye holding registers as the IRIV job list, and expose entities to Home Assistant (ESPHome API). Optionally publish MQTT if you configure it — the static [`web/`](../web/) dashboard only works when topics under `iriv/ivt/#` exist on a broker.

**One Modbus master per RS485 bus.** Do not share A/B with the IRIV IOC MQTT Gateway.

| Item | Value |
|------|--------|
| Baud | **9600** 8N1 |
| Slave ID | **1** |
| Port | Deye **datalogger / meter RS485** (not BMS CAN) |
| Example YAML | [`deye-sg06-nodemcu.yaml`](deye-sg06-nodemcu.yaml) (NodeMCU v2) |

Register map: [Deye PDF V118](../docs/protocol/Deye%20SG05%20Modbus%20Protocol.V118.pdf) + field notes in the [root README](../README.md). Job list reference: [`../iriv-ioc-mqtt-gateway/`](../iriv-ioc-mqtt-gateway/).

---

## Wiring ESP32 (or NodeMCU) to an RS485 module

Use a transceiver such as **MAX3485 / MAX485** or a Waveshare USB/TTL–RS485 board in TTL mode.

```text
  ESP32 / NodeMCU          RS485 module              Deye inverter
  ----------------         -------------             -------------
  UART TX  --------------> DI (driver in)
  UART RX  <-------------- RO (receiver out)
  3V3 / 5V ---------------> VCC (match module)
  GND  ------------------+-> GND
                         |
                         +-- A  ------------------> RS485 A
                         +-- B  ------------------> RS485 B
```

- If the module has **DE/RE**, tie them for automatic direction or drive DE high when transmitting (many cheap boards use a jumper or single pin). For half-duplex Modbus, ensure the bus is released after each poll.
- Swap **A/B** if the link is silent.
- Prefer **hardware UART** on ESP32 for denser polling; NodeMCU YAML uses software serial on GPIO14/12.

### Pin map

| Role | NodeMCU (YAML as shipped) | Suggested ESP32 HW UART |
|------|---------------------------|-------------------------|
| RX ← RO | **D5** = GPIO14 | GPIO16 (or any free UART RX) |
| TX → DI | **D6** = GPIO12 | GPIO17 (or any free UART TX) |
| GND | GND | GND |

To use ESP32: change `esp8266:` → `esp32:` (board variant), set `uart:` `rx_pin` / `tx_pin` to your pins, and keep baud **9600**. Leave `logger: baud_rate: 0` if the logger would steal the UART.

---

## Flash the firmware

1. Install [ESPHome](https://esphome.io/) (CLI or HA add-on).
2. Create `secrets.yaml` next to the device YAML (or use the HA secrets store):

```yaml
wifi_ssid: "your-ssid"
wifi_password: "your-wifi-password"
```

3. Edit [`deye-sg06-nodemcu.yaml`](deye-sg06-nodemcu.yaml): set `esphome.name`, Wi‑Fi, API encryption key / OTA password as you prefer. Hostname in the template is `sg06-nodemcu`.
4. Connect USB and flash:

```bash
cd esphome
esphome run deye-sg06-nodemcu.yaml
```

5. In Home Assistant, adopt the node via the ESPHome / HA API integration (encryption key must match).

### Poll timing

The Deye map is sparse → many FC03 ranges. The template uses ~**15 s** `update_interval` and `command_throttle: 100ms`. If logs show `Frame already active` or `Poll refused by hub`, increase the interval or throttle further.

### Bench without the inverter

```bash
python emulator/deye-sg06-ivt-emu.py --port COMxx --debug --scenario day
```

Point the ESP UART–RS485 adapter at the same USB–RS485 COM port bus (or a second adapter on a shared A/B pair with GND).

---

## ESP32 native poller (no ESPHome)

A non-ESPHome ESP-IDF / Arduino master that publishes MQTT `iriv/ivt/...` (for the web dashboard and HA MQTT package) is still a planned option. Same registers, baud, slave ID, and **one-master** rule as above. Until that lands, use ESPHome or the IRIV gateway.
