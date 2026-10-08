# Emulator slave Modbus biến tần Deye SG06

> English: [README.md](README.md)

Emulator **Modbus RTU slave** trên PC (USB–RS485), giả lập Deye SG05/SG06. Dùng để mang lên IRIV IOC MQTT Gateway hoặc ESPHome **không cần** biến tần thật.

| Mục | Mặc định |
|-----|----------|
| Vai trò | Modbus **slave** |
| Baud | **9600** 8N1 |
| Slave ID | **1** |
| Entry | [`deye-sg06-ivt-emu.py`](deye-sg06-ivt-emu.py) |
| Profile | [`rs485_emu/profiles/deye_sg_inverter.py`](rs485_emu/profiles/deye_sg_inverter.py) |

Bản đồ thanh ghi theo PDF V118 của Deye và scale thực tế dùng trong [`../iriv-ioc-mqtt-gateway/`](../iriv-ioc-mqtt-gateway/) / [`../esphome/`](../esphome/).

**Một master trên bus** — chỉ một adapter USB–RS485 (hoặc một cặp A/B chung) nối tới emulator; không chạy IRIV và ESPHome cùng lúc.

## Cài phụ thuộc

Từ root repo:

```bash
pip install -r requirements.txt
```

Cần `pymodbus` và `pyserial` (xem `requirements.txt` ở root).

## Chạy

```bash
# Từ root repo
python emulator/deye-sg06-ivt-emu.py --port COM35 --debug --scenario day

# Hoặc từ thư mục này
cd emulator
python deye-sg06-ivt-emu.py --port COM35 --trace --scenario fault
```

| Cờ | Ý nghĩa |
|----|---------|
| `--port` | Cổng COM USB–RS485 (mặc định `COM35`) |
| `--baudrate` | Mặc định `9600` |
| `--slave-id` | Mặc định `1` |
| `--scenario` | `day` · `night` · `cloud` · `fault` |
| `--tick` | Chu kỳ mô phỏng (giây, mặc định `1.0`) |
| `--debug` | Log mỗi lần đọc register kèm giá trị kỹ thuật |
| `--trace` | Dump khung RTU hex RX/TX |
| `--seed` | Seed RNG tùy chọn để nhiễu lặp lại được |

Smoke test từ master Modbus: holding reg **59** ≈ `2` (normal) hoặc `4` (scenario fault).

## Cấu trúc

```text
emulator/
  deye-sg06-ivt-emu.py   # CLI
  rs485_emu/
    core/                # serial server, registers, tracer, CLI
    profiles/            # deye_sg_inverter.py
```

Emulator BMS kiểu Pylon nằm ở repo chị em `jk-pb-rs485-monitor`, không nằm ở đây.
