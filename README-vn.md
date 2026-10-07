# Deye SG06 RS485 Monitor

Công cụ **monitor** biến tần Deye SG05/SG06 qua **Modbus RTU (RS485)** trên cổng datalogger: cấu hình IRIV IOC MQTT Gateway, ESPHome / ESP32 + RS485, emulator, Mosquitto Docker, Home Assistant và dashboard web MQTT.

> English: [README.md](README.md)

**Dự án chị em:** [jk-pb-rs485-monitor](../jk-pb-rs485-monitor) — BMS JK-PB* (bus/baud riêng).

## Tính năng

- Giao tiếp Modbus với biến tần (master → slave ID **1**, **9600** 8N1).
- Thanh ghi theo tài liệu hãng (PDF V118) kèm hiệu chỉnh thực tế trên SG06.
- Hai đường phần cứng: **IRIV IOC MQTT Gateway** (RS485 → MQTT) hoặc **ESP32/ESPHome + mạch RS485**.
- **Một master / một bus RS485** — không chạy IRIV và ESPHome cùng lúc trên cùng A/B.

## Cấu trúc

| Thư mục | Nội dung |
|---------|----------|
| `iriv-ioc-mqtt-gateway/` | JSON IRIV, Docker Mosquitto, YAML Home Assistant |
| `esphome/` | YAML ESPHome + hướng dẫn nối ESP32 ↔ RS485 và flash |
| `emulator/` | Emulator slave Modbus Deye |
| `web/` | Dashboard MQTT WebSocket (`iriv/ivt/#`) |
| `docs/` | HANDOFF, PDF protocol, ảnh |

## Nhanh

```bash
pip install -r requirements.txt

# Mosquitto + IRIV
cd iriv-ioc-mqtt-gateway && docker compose up -d
# Lần đầu: USB-C tới gateway → http://10.0.0.1 → import iriv-ioc-config.json (≥ V1.2.6)

# ESPHome
cd esphome && esphome run deye-sg06-nodemcu.yaml

# Emulator bench
python emulator/deye-sg06-ivt-emu.py --port COM35 --debug --scenario day
```

Chi tiết IRIV / Mosquitto / HA: [`iriv-ioc-mqtt-gateway/README.md`](iriv-ioc-mqtt-gateway/README.md).  
Chi tiết ESP32 + RS485: [`esphome/README.md`](esphome/README.md).  
Dashboard web chỉ dùng khi đã có publisher MQTT `iriv/ivt/#` (IRIV hoặc ESP32 publish MQTT): [`web/README.md`](web/README.md).

## Đã thử nghiệm

**Deye 6 kW SG06** + pin **16S 51.2 V 100 Ah** (JK qua **CAN**). Modbus inverter **9600**, slave **1**.
