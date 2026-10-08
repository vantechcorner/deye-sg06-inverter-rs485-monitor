# Deye SG06 inverter Modbus slave emulator

> Vietnamese: [README-vn.md](README-vn.md)

PC-side **Modbus RTU slave** that pretends to be a Deye SG05/SG06 on USB–RS485. Use it to bring up IRIV IOC MQTT Gateway or ESPHome **without** a live inverter.

| Item | Default |
|------|---------|
| Role | Modbus **slave** |
| Baud | **9600** 8N1 |
| Slave ID | **1** |
| Entry point | [`deye-sg06-ivt-emu.py`](deye-sg06-ivt-emu.py) |
| Profile | [`rs485_emu/profiles/deye_sg_inverter.py`](rs485_emu/profiles/deye_sg_inverter.py) |

Register map follows Deye PDF V118 and the field scales used by [`../iriv-ioc-mqtt-gateway/`](../iriv-ioc-mqtt-gateway/) / [`../esphome/`](../esphome/).

**One master on the bus** — point a single USB–RS485 adapter (or a shared A/B pair) at this emulator; do not run IRIV and ESPHome against it at the same time.

## Requirements

From the repo root:

```bash
pip install -r requirements.txt
```

Needs `pymodbus` and `pyserial` (see root `requirements.txt`).

## Run

```bash
# From repo root
python emulator/deye-sg06-ivt-emu.py --port COM35 --debug --scenario day

# Or from this folder
cd emulator
python deye-sg06-ivt-emu.py --port COM35 --trace --scenario fault
```

| Flag | Meaning |
|------|---------|
| `--port` | USB–RS485 COM port (default `COM35`) |
| `--baudrate` | Default `9600` |
| `--slave-id` | Default `1` |
| `--scenario` | `day` · `night` · `cloud` · `fault` |
| `--tick` | Physics tick seconds (default `1.0`) |
| `--debug` | Log register reads with engineering values |
| `--trace` | Dump raw RTU hex RX/TX |
| `--seed` | Optional RNG seed for reproducible noise |

Smoke check from a Modbus master: holding reg **59** ≈ `2` (normal) or `4` (fault scenario).

## Layout

```text
emulator/
  deye-sg06-ivt-emu.py   # CLI entry
  rs485_emu/
    core/                # serial server, registers, tracer, CLI helpers
    profiles/            # deye_sg_inverter.py
```

Pylon-style BMS bench slave lives in the sister repo `jk-pb-rs485-monitor`, not here.
