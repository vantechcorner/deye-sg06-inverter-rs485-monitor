# IRIV IOC MQTT Gateway — Deye SG06

Cytron **IRIV IOC MQTT Gateway** polls the Deye inverter over **Modbus RTU (RS485)** and publishes scaled values to MQTT under `iriv/ivt/...`.

**One Modbus master per RS485 bus.** Do not run this gateway on the same A/B wires as ESPHome / ESP32.

| Item | Value |
|------|--------|
| Baud | **9600** 8N1 |
| Slave ID | **1** |
| MQTT base | `iriv/ivt` |
| Payload | JSON `{"value": <number>}` |
| Firmware | Prefer **≥ V1.2.6** for the 27-job config |

## Files

| File | Role |
|------|------|
| [`iriv-ioc-config.json`](iriv-ioc-config.json) | Import on firmware **≥ V1.2.6** (27 jobs, full PV2) |
| [`iriv-ioc-config-26.json`](iriv-ioc-config-26.json) | Older firmware only (drops PV2 Current) |
| [`_gen_iriv_jobs.py`](_gen_iriv_jobs.py) | Regenerates both JSON files |
| [`iriv_deye_mqtt.yaml`](iriv_deye_mqtt.yaml) | Home Assistant MQTT sensor package |
| [`docker-compose.yml`](docker-compose.yml) + [`mosquitto.conf`](mosquitto.conf) | Local Mosquitto (1883 + WebSocket 9001) |

Device web login in the JSON templates: user **`admin`**, password **`12345678`** (stored as `adminPassHash` / `adminPassSalt`). Change this after first setup if the device is reachable beyond your LAN.

Regenerate jobs (do not hand-edit dozens of poll slots if avoidable):

```bash
python iriv-ioc-mqtt-gateway/_gen_iriv_jobs.py
```

---

## 1. Run Mosquitto with Docker

From this folder:

```bash
cd iriv-ioc-mqtt-gateway
docker compose up -d
```

| Port | Use |
|------|-----|
| **1883** | MQTT TCP — IRIV gateway, HA MQTT integration |
| **9001** | MQTT over WebSockets — [`web/`](../web/) dashboard |

Check the broker:

```bash
docker compose logs -f mosquitto
```

Point the IRIV MQTT host at the machine running Docker (hostname or LAN IP). The templates default to `iriv-pi-control`; edit on the device or re-export after changing host in the JSON if needed.

Stop:

```bash
docker compose down
```

Do **not** expose 1883/9001 to the public internet without TLS and authentication.

---

## 2. First-time restore on the IRIV IOC MQTT Gateway

On a new or factory-reset gateway, configuration is done over the USB gadget Ethernet interface:

1. Connect **USB-C** from the IRIV IOC MQTT Gateway to your PC.
2. Wait until the host gets a USB Ethernet / RNDIS link (Windows may install a driver).
3. Open a browser to **`http://10.0.0.1`**.
4. Log in with the credentials in the JSON template (**`admin` / `12345678`**) or the factory defaults if you have not imported yet (Cytron docs often ship `admin` / `admin` until you change the password).
5. Restore / import the poll config:
   - Firmware **≥ V1.2.6** → [`iriv-ioc-config.json`](iriv-ioc-config.json)
   - Older firmware → [`iriv-ioc-config-26.json`](iriv-ioc-config-26.json) (a 27th enabled job wiped the list on pre–V1.2.6 builds)
6. Confirm MQTT: host = your Mosquitto IP/hostname, port **1883**, base topic **`iriv/ivt`**, auth off unless your broker requires it.
7. Wire **RS485 A/B** from the gateway to the Deye **datalogger / meter RS485** port (not the BMS CAN RJ45). Baud **9600**, slave **1**.
8. After Ethernet LAN is enabled, use the device’s LAN IP for later config; USB `10.0.0.1` is mainly for first bring-up.

Verify traffic (example with mosquitto clients):

```bash
mosquitto_sub -h <broker-host> -t "iriv/ivt/#" -v
```

You should see topics such as `iriv/ivt/battery/soc`, `iriv/ivt/pv1/power`, `iriv/ivt/load/current`.

---

## 3. Add sensors in Home Assistant

1. Ensure HA can reach the same Mosquitto broker (Settings → Devices & services → MQTT).
2. Copy [`iriv_deye_mqtt.yaml`](iriv_deye_mqtt.yaml) into your HA config, e.g. `config/packages/iriv_deye_mqtt.yaml`, with:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

3. Restart HA or reload MQTT entities.
4. Entities appear under device **Deye SG06 (IRIV)**; each sensor reads `value_json.value` from `iriv/ivt/...`.

Alternatively merge the `mqtt:` block into `configuration.yaml`.

---

## Poll periods (template)

| Group | Period |
|-------|--------|
| Voltage / Current / Power | **1 s** |
| Temp / SOC / frequency / status | **10 s** |
| `*_today` energy | **30 s** |

Field-verified scales (SG06): charge/discharge today **70 / 71**; PV current **0.1** A; Grid current **160** ×**0.01** A; Load current **179** ×0.01 A; Inverter frequency **193** ×0.01 Hz. Use `dataType` **s16** (`3`) for 16-bit holdings.
